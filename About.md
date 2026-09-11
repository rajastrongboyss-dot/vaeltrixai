# VaeltrixAI — Full-Stack (V1.8.0 → FastAPI + Supabase)

Frontend existing (vanilla HTML/CSS/JS, PWA) + backend FastAPI + Supabase, satu project, satu origin.

## Arsitektur

```
Browser
  │
  ├─ GET /                  → index.html
  ├─ GET /css|js|assets|fonts/* → static files
  ├─ GET /manifest.json, /sw.js
  └─ /api/v1/*              → FastAPI
                                 │
                                 ├─ Supabase Auth (REST, via httpx)
                                 ├─ PostgREST (RLS, token milik user)
                                 ├─ Gemini / Groq (chat)
                                 └─ Stripe (billing)
```

Frontend tidak butuh konfigurasi base URL — `VAELTRIX_BACKEND_BASE` di `js/01-config.js` sudah default relatif ke origin sendiri.

Auth (`/login`, `/register`, `/forgot-password`, `/reset-password`, `/auth/callback`) adalah
route client-side di app statis yang sama (`index.html` di-serve buat semua path itu lewat
SPA fallback di `main.py`, lalu `js/17-account.js` yang nentuin tampilan mana yang muncul) —
bukan file HTML terpisah, jadi gak butuh route baru di backend.

## Setup

1. **Buat project Supabase** di [supabase.com](https://supabase.com/dashboard).
2. **Jalankan migration** — buka SQL Editor di dashboard, jalankan berurutan isi tiap file di `supabase/migrations/` (0001 → 0008).
3. **Isi `.env`** (copy dari `.env.example`) — minimal `SUPABASE_URL`, `SUPABASE_PUBLISHABLE_KEY`, `SUPABASE_SECRET_KEY` dari Project Settings > API. Provider AI (`GEMINI_API_KEYS`/`GROQ_API_KEYS`), `PREMIUM_REDEEM_CODES`, dan Stripe bersifat opsional — endpoint terkait balikin error yang jelas kalau kosong, bukan crash.

## Menjalankan lokal

```bash
cd backend
pip install -r requirements.txt
uvicorn app.main:app --reload
```

Buka `http://localhost:8000`. Set `COOKIE_SECURE=False` di `.env` untuk dev via `http://` (browser tidak mengirim cookie `secure` di koneksi non-HTTPS).

## Docker

```bash
docker build -t vaeltrixai .
docker run -p 8000:8000 --env-file .env vaeltrixai
```

## Testing

```bash
cd backend
pip install -r requirements-dev.txt
python -m pytest tests/ --ignore=tests/integration
```

Test yang ada (`backend/tests/`, TIDAK termasuk `tests/integration/`) murni logic (entitlement, rate limit atomik, webhook idempotency, body-size middleware, model registry, format SSE, validasi schema) — **tidak butuh Supabase/provider asli**, dipakai mock/fake object buat batas luar (`RestClient.rpc`, `PrivilegedRestClient.insert_one`, `stripe.Webhook.construct_event`).

`tests/integration/test_rls_cross_user.py` **beda karakter** — butuh project Supabase TEST asli (bukan production, karena bikin user baru tiap jalan) lewat 2 env var (`VAELTRIX_TEST_SUPABASE_URL`, `VAELTRIX_TEST_SUPABASE_PUBLISHABLE_KEY`), dan **skip otomatis** kalau keduanya kosong. RLS ditegakkan Postgres sendiri, jadi gak ada cara jujur nge-"unit test"-nya pakai mock — jalankan `pytest tests/integration/ -v` terpisah setelah env var-nya diisi.

## Keamanan — hal yang perlu kamu tahu

- **`DEFAULT_KEY`/`DEFAULT_KEY2`/`GROQ_KEY`/`TAVILY_KEY` di `js/01-config.js` ditemukan
  terisi kredensial ASLI VaeltrixLabs**, browser-visible, tanpa rate limit (panggilan
  langsung ke provider, gak lewat backend). Sudah dikosongin — **tapi key lamanya sendiri
  masih valid sampai kamu rotasi manual** di dashboard Google AI Studio/Groq/Tavily/
  Supabase (`.env` yang ikut ke-bundle di release zip juga ada kredensial yang sama,
  cek riwayat/backup rilis lama kalau ada). ElevenLabs & Pollinations diperiksa juga —
  keduanya TIDAK PERNAH punya kredensial platform (murni BYOK/keyless), gak ada yang
  perlu dirotasi di situ.
- **Kode premium lama (`PREMIUM_CODES` di `js/01-config.js`) harus dianggap bocor.** Sebelumnya divalidasi 100% di browser (`localStorage.vaeltrix_premium`), gampang dipalsukan dari devtools. Sekarang user yang login divalidasi lewat `/api/v1/account/redeem-code` + `PREMIUM_REDEEM_CODES` (server-side, **isi kode baru**, jangan pakai yang lama). Guest tetap pakai jalur lokal lama (gak ada sesi server, gak nyentuh kuota berbayar platform).
- **`profiles.tier` tidak bisa di-PATCH langsung oleh user** — kolomnya di-`REVOKE` dari role `authenticated` (migration `0003`). Hanya berubah lewat redeem-code atau webhook Stripe, keduanya pakai `SUPABASE_SECRET_KEY` (privileged), selalu di-scope manual ke user yang sudah terverifikasi token-nya — tidak pernah percaya `user_id`/`tier` dari body request.
- **RLS aktif di semua tabel** (`profiles`, `projects`, `conversations`, `messages`, plus `rate_limit_hits`/`processed_stripe_events`/`auth_attempt_log` yang baru — tanpa policy `authenticated` sama sekali, cuma lewat function `security definer`/service role) + pengecekan eksplisit di service layer untuk kasus yang RLS saja tidak cukup (mis. `projectId` yang direferensikan saat import conversation).
- **Rate limit atomik di level database** (migration `0007`): `check_and_reserve_rate_limit`
  gabungin count+reservasi jadi satu transaksi (advisory lock per-user via `auth.uid()`) —
  sebelumnya count lalu compare terpisah di Python, race di request konkuren.
  Limit tetap sama (20/jam free, 60/jam premium+pro).
- **Login/register/forgot-password ada throttle per-IP dan per-email** (migration `0008`,
  `check_and_log_auth_attempt`, dipanggil lewat service role) — belum progressive delay
  (threshold tetap, bukan naik bertahap), tapi cukup buat brute-force/spam kasar.
- **Webhook Stripe idempoten** (migration `0007`, tabel `processed_stripe_events`,
  primary key = event ID) — event yang sama gak diproses dua kali kalau Stripe retry.
  Signature verification tetap wajib seperti sebelumnya.
- **Batas panjang input** (section 19) — `message` (8000), `systemPrompt` (4000), judul
  percakapan (200), instruksi project (8000), isi pesan yang di-import (20000), jumlah
  pesan per import (2000). Sebelumnya cuma ketahan `MaxBodySizeMiddleware` generik (2MB
  buat semua endpoint) — sekarang tiap field ada batasnya sendiri.
- **CATATAN JUJUR, belum dikerjakan**: `payload.systemPrompt` (isinya persona+FORMAT_GUIDE+memory+instruksi user, semua digabung CLIENT-SIDE lewat `buildSystemPrompt()`) dikirim APA ADANYA ke provider AI, backend gak nyuntikkan versi non-bisa-ditimpa dari kontennya sendiri. Detail lengkap + kenapa ini bukan "aku yang mesti nulis kebijakan baru" ada di `SECURITY.md`.
- **XSS & CSRF (section 56-57) sudah ditelusuri, aman** — markdown AI di-escape penuh sebelum dirender + whitelist protokol URL, dan semua endpoint kecuali refresh/logout pakai Bearer token (bukan cookie), jadi gak rawan CSRF klasik. Detail di `SECURITY.md`.
- **Context AI dibatasi** (section 20) — history yang dikirim ke model AI sekarang dibatasi 40 pesan terakhir (`CHAT_HISTORY_LIMIT`, `chat_service.py`), gak lagi ambil SELURUH percakapan tanpa batas.
- **`conversations.updated_at` sekarang beneran ke-touch** (section 22) — trigger `touch_updated_at` sudah ada dari awal tapi cuma nyala kalau baris conversations di-UPDATE langsung; kirim pesan baru sekarang eksplisit nyentuh itu, jadi urutan sidebar (`updated_at.desc`) beneran refleksiin percakapan yang baru aktif.
- **Security headers ditambah** (section 58): `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`, `Permissions-Policy` selalu; `Strict-Transport-Security` cuma kalau `APP_ENV=production`. **Content-Security-Policy SENGAJA belum ditambah** — frontend ini pakai `onclick="..."` inline di hampir semua tombol, CSP yang benar (`script-src` tanpa `unsafe-inline`) bakal mematikan hampir semua interaksi tanpa revisi arsitektur event dulu. Detail di `SECURITY.md`.
- **Cloud sync (section 43), Docker (section 53), validasi env saat startup (section 27) — ditelusuri, sudah benar dari awal, tidak diubah**: whitelist cloud sync eksplisit menolak semua BYOK key, Dockerfile non-root+healthcheck+gak nge-bake secret, `/api/v1/health` beneran ngecek Supabase (dan sekarang juga ngecek key AI provider) alih-alih selalu 200.
- **Pagination percakapan sekarang cursor-based** (section 21): `GET /conversations` terima `?cursor=...`, balikin `nextCursor`. `vaeltrixSyncConversationList()` (`18-cloud-sync.js`) looping otomatis sampai habis — sebelumnya cuma fetch 50 percakapan terbaru sekali, sisanya ilang gitu aja buat user yang percakapannya lebih dari itu pas login di device baru.
- **Artifact preview** (iframe `sandbox="allow-scripts allow-modals"`, tanpa `allow-same-origin`) sudah benar dari awal — dicek, tidak diubah.

## Yang sengaja tidak dikerjakan (audit tidak menemukan gap nyata)

Backend Storage untuk attachments, proxy search/voice, account export/delete — semua ini saat ini murni client-side atau tidak punya UI sama sekali di frontend. Menambahnya sekarang berarti menciptakan fitur baru, bukan menutup lubang — di luar scope "transformasi backend", jadi tidak ditambahkan tanpa diminta eksplisit.

## Batasan jujur

Seluruh kode di atas ditulis dan di-cross-check teliti terhadap kontrak yang sudah ada di frontend (endpoint, bentuk request/response, event SSE) — **tapi belum pernah benar-benar dijalankan**. Sandbox tempat ini ditulis tidak punya akses internet maupun Docker/pytest ter-install, jadi tidak bisa connect ke Supabase/Gemini/Groq/Stripe asli atau build image sungguhan. Jalankan `python -m pytest`, lalu tes manual end-to-end (register → login → chat → reload → cek conversation tersimpan) di environment kamu sendiri sebelum dianggap production-ready.

## Referensi endpoint

| Endpoint | Method | Auth | Keterangan |
|---|---|---|---|
| `/api/v1/health` | GET | — | 200 = sehat, 503 = degraded |
| `/api/v1/auth/register`, `/login`, `/refresh`, `/logout` | POST | cookie refresh token | |
| `/api/v1/auth/resend-verification` | POST | — | Kirim ulang email konfirmasi |
| `/api/v1/auth/forgot-password` | POST | — | Selalu balas sukses generik (anti-enumeration), throttled |
| `/api/v1/auth/reset-password` | POST | — | Butuh token recovery dari link email |
| `/api/v1/auth/oauth/{provider}` | GET | — | Redirect ke Supabase (`google`/`github`) |
| `/api/v1/auth/oauth-callback` | POST | — | Tukar token hasil redirect OAuth jadi sesi kita |
| `/api/v1/chat/stream` | POST | Bearer | SSE: `start`/`chunk`/`error`/`done` |
| `/api/v1/conversations`, `/projects` | GET/POST/PATCH/DELETE | Bearer | RLS-scoped |
| `/api/v1/account/redeem-code` | POST | Bearer | Upgrade tier via kode |
| `/api/v1/account/change-password` | POST | Bearer | Verifikasi password lama dulu |
| `/api/v1/account/profile` | PATCH | Bearer | Update nama (`profiles.name`) |
| `/api/v1/account` | DELETE | Bearer | Hapus akun + semua data (cascade) |
| `/api/v1/settings` | GET/PATCH | Bearer | Sync preferensi lintas device |
| `/api/v1/usage` | GET | Bearer | Total request & token terpakai |
| `/api/v1/billing/subscription`, `/checkout`, `/portal` | GET/POST | Bearer | Stripe |
| `/api/v1/billing/webhook` | POST | Stripe signature | Dipanggil Stripe, bukan frontend |
