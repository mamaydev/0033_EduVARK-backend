# GATE C — Konfigurasi Backend untuk Render + Supabase

Dokumen ini adalah instruksi kerja untuk agent yang mengerjakan **Gate C** di workspace baru. Bacalah seluruh dokumen sebelum melakukan apa pun.

## 0. Konteks & Pembagian Tugas

- **Repo backend:** `https://github.com/mamaydev/0033_EduVARK-backend` (branch `main`) — berisi kode Express + TypeScript EduVARK. Repo ini **terpisah** dari repo Android `0033_EduVARK`.
- **Tugas Gate C:** menyiapkan project Supabase, menyesuaikan koneksi database, menyiapkan Cloudflare R2, membuat `render.yaml`, dan menyiapkan deploy Render. **Belum wajib melakukan verifikasi end-to-end aplikasi** — itu Gate D.
- **Bukan tugas Anda:** mengubah repo Android, mengubah `pam.sequein.my.id`, atau memigrasikan kode lagi.
- **Dua backend berbeda:** `pam.sequein.my.id` adalah backend lama milik pemilik proyek dan **tidak boleh diubah/disentuh**. Backend baru ini memakai URL default Render (`https://<service>.onrender.com`). Tidak ada custom domain untuk backend baru.

## 1. Kondisi Awal

- Node.js **>= 24** (`node -v`), `npm >= 11`.
- Docker tersedia (dipakai Render untuk build, tidak wajib lokal).
- Kode sudah lengkap di repo: `src/`, `tests/`, `Dockerfile`, `.env.example`, `.env.production.example`.
- Health check: `GET /health` (liveness) dan `GET /health/ready` (database ready).
- Tabel migrasi: `schema_migrations` (dikelola `src/db/migrate.ts`).

## 2. Setup Workspace

```bash
git clone https://github.com/mamaydev/0033_EduVARK-backend.git
cd 0033_EduVARK-backend
npm ci
```

Jangan pernah menyalin `.env` dari mesin lain; repo hanya berisi `.env.example` dan `.env.production.example`.

## 3. Langkah 1 — Buat Project Supabase

1. Masuk ke https://supabase.com/dashboard → **New project**.
2. Isi:
   - **Name:** `eduvark` (atau nama lain yang jelas).
   - **Database password:** buat password kuat, **simpan di password manager**. Password ini tidak boleh masuk repo.
   - **Region:** pilih yang terdekat dengan Render (disarankan **Singapore**). Region DB tidak bisa diubah setelah dibuat.
3. Tunggu project selesai provisioning (± 2 menit).
4. Ambil connection string dari **Project Settings → Database → Connection string**:
   - **Direct connection** (untuk migrasi, port `5432`):
     ```text
     postgresql://postgres:<PASSWORD>@db.<PROJECT_REF>.supabase.co:5432/postgres?sslmode=require
     ```
   - **Pooler / Transaction mode** (untuk runtime aplikasi, port `6543`):
     ```text
     postgresql://postgres.<PROJECT_REF>:<PASSWORD>@aws-0-<REGION>.pooler.supabase.com:6543/postgres?pgbouncer=true&sslmode=require
     ```
   Catatan: nilai persis bisa berbeda; **salin apa adanya dari dashboard**, jangan mengarang host/region.

Aturan pemakaian:

| Keperluan | Connection string |
|---|---|
| `npm run db:migrate` (manual, sekali) | **Direct** port 5432 |
| `DATABASE_URL` di Render (runtime) | **Pooler** port 6543 |

Jangan menjalankan migrasi lewat pooler. Jangan menaruh direct URL di Render.

## 4. Langkah 2 — Sesuaikan Koneksi Database (Wajib)

Supabase mewajibkan SSL. Ubah `src/db/client.ts` agar mendukung SSL tanpa merusak mode lokal:

```ts
export function getPool(): Pool {
  if (pool === null) {
    const env = getEnv();
    const connectionString = env.DATABASE_URL;
    const needsSsl = /supabase\.(co|com)|sslmode=require/i.test(connectionString);
    pool = new Pool({
      connectionString,
      ssl: needsSsl ? { rejectUnauthorized: false } : undefined,
      max: 10,
      idleTimeoutMillis: 30_000,
      connectionTimeoutMillis: 5_000,
    });
  }
  return pool;
}
```

Catatan:
- `rejectUnauthorized: false` dipakai karena Supabase pooler memakai sertifikat yang tidak selalu bisa divalidasi otomatis di lingkungan build. Jika nanti ingin lebih ketat, pasang CA certificate Supabase.
- Jangan mengubah `src/config/env.ts` untuk hal lain; `DATABASE_URL` sudah wajib di schema.
- Uji lokal bila memungkinkan:
  ```bash
  DATABASE_URL="<DIRECT_URL>" npm run db:migrate
  ```

## 5. Langkah 3 — Terapkan Migrasi ke Supabase

Dengan environment direct URL (jangan di-commit):

```bash
DATABASE_URL="<DIRECT_URL>" npm run db:migrate
```

Ekspektasi:
- Tabel `schema_migrations` berisi **5** baris: `001_init`, `002_admin_login_ids`, `003_admin_session_revoke_and_range`, `004_material_object_deletions`, `005_material_file_replacements`.
- Perintah idempoten: menjalankan ulang tidak menerapkan migrasi yang sudah ada.
- Jika gagal karena checksum, **jangan** mengedit file migrasi; laporkan.

Seed admin (password minimal 6 karakter; login ID admin dibuat sistem dan dicatat di log):

```bash
DATABASE_URL="<DIRECT_URL>" \
ADMIN_PASSWORD="<PASSWORD_ADMIN_KUAT>" \
ADMIN_FULL_NAME="Administrator" \
npm run db:seed-admin
```

Simpan password admin yang dipakai; Gate D membutuhkannya untuk login. Jangan commit nilainya.

## 6. Langkah 4 — Siapkan Cloudflare R2 (Penyimpanan Materi)

Render memakai disk ephemeral, jadi **wajib** object storage. Backend sudah mendukung R2 (S3 API).

1. Cloudflare Dashboard → R2 → **Create bucket** (mis. `eduvark-materials`).
2. R2 → **Manage R2 API Tokens** → Create API token dengan permission **Object Read & Write** untuk bucket tersebut.
3. Catat:
   - `R2_ENDPOINT`: `https://<ACCOUNT_ID>.r2.cloudflarestorage.com`
   - `R2_BUCKET`: nama bucket
   - `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY` dari token
4. Tidak perlu mengubah `src/lib/storage.ts`; cukup isi env di Render.
5. Pastikan `STORAGE_DELETION_INTERVAL_MS`, `STORAGE_DELETION_MAX_ATTEMPTS`, `FILE_REPLACEMENT_LEASE_MS` dibiarkan default kecuali ada alasan khusus.

## 7. Langkah 5 — Buat `render.yaml` dan Deploy Service

Tambahkan `render.yaml` di root repo:

```yaml
services:
  - type: web
    name: eduvark-backend
    runtime: docker
    plan: starter
    region: singapore
    branch: main
    dockerfilePath: ./Dockerfile
    healthCheckPath: /health/ready
    autoDeploy: true
    envVars:
      - key: NODE_ENV
        value: production
      - key: DATABASE_URL
        sync: false
      - key: JWT_ACCESS_SECRET
        sync: false
      - key: JWT_REFRESH_SECRET
        sync: false
      - key: CORS_ORIGINS
        sync: false
      - key: PUBLIC_API_BASE_URL
        value: https://<service>.onrender.com
      - key: R2_ENDPOINT
        sync: false
      - key: R2_BUCKET
        sync: false
      - key: R2_ACCESS_KEY_ID
        sync: false
      - key: R2_SECRET_ACCESS_KEY
        sync: false
      - key: R2_SIGNED_URL_TTL_SECONDS
        value: "300"
      - key: MAX_UPLOAD_BYTES
        value: "104857600"
      - key: LOG_LEVEL
        value: info
```

Catatan:
- `<service>` adalah nama service Render Anda; URL final baru diketahui setelah deploy. Anda boleh commit `render.yaml` dengan placeholder, lalu perbarui `PUBLIC_API_BASE_URL` di dashboard setelah URL ada, atau adjust `render.yaml` dan push ulang.
- `sync: false` = nilai diisi manual di dashboard Render (secret), tidak disimpan di repo.
- **Jangan** menulis nilai secret apa pun ke `render.yaml` atau repo.
- `PORT` diinject Render otomatis; `src/config/env.ts` membaca `PORT`. Tidak perlu diset manual.
- Plan boleh `free` untuk uji (service tidur saat idle), tetapi untuk pemakaian nyata gunakan plan berbayar agar tidak cold start.

Langkah deploy:
1. Render Dashboard → **New → Web Service** → hubungkan repo `0033_EduVARK-backend`.
2. Runtime **Docker** (Dockerfile di root), branch `main`, region **Singapore**.
3. Health check path `/health/ready`.
4. Isi env vars pada tabel §8.
5. Deploy. Tunggu status **live**.
6. Catat URL akhir: `https://<service>.onrender.com`. Laporkan ke Lead/user.

## 8. Env Vars di Render

| Key | Wajib | Nilai / Sumber |
|---|---|---|
| `NODE_ENV` | ya | `production` |
| `DATABASE_URL` | ya | **Pooler** Supabase (port 6543) |
| `JWT_ACCESS_SECRET` | ya (production) | `openssl rand -hex 32` |
| `JWT_REFRESH_SECRET` | ya (production) | `openssl rand -hex 32` |
| `ACCESS_TOKEN_TTL` | tidak | default `15m` |
| `REFRESH_TOKEN_TTL` | tidak | default `12h` |
| `SESSION_IDLE_TTL` | tidak | default `8h` |
| `SESSION_ABSOLUTE_TTL` | tidak | default `12h` |
| `CORS_ORIGINS` | tidak | kosongkan (mobile tanpa Origin) atau daftar origin browser yang diizinkan |
| `PUBLIC_API_BASE_URL` | tidak | `https://<service>.onrender.com` |
| `R2_ENDPOINT` | ya untuk materi | `https://<ACCOUNT_ID>.r2.cloudflarestorage.com` |
| `R2_BUCKET` | ya untuk materi | nama bucket |
| `R2_ACCESS_KEY_ID` | ya untuk materi | token R2 |
| `R2_SECRET_ACCESS_KEY` | ya untuk materi | token R2 |
| `R2_SIGNED_URL_TTL_SECONDS` | tidak | `300` |
| `MAX_UPLOAD_BYTES` | tidak | `104857600` |
| `LOG_LEVEL` | tidak | `info` |
| `STORAGE_DELETION_*`, `FILE_REPLACEMENT_LEASE_MS` | tidak | default |
| `CLIENT_COMPATIBILITY_ENFORCE`, `CLIENT_MIN_VERSION` | tidak | biarkan default |

Secret hanya boleh ada di dashboard Render. Jangan pernah menaruhnya di repo, log, atau laporan.

## 9. Langkah 6 — Perbarui `README.md` (Backend)

Tambahkan bagian singkat: arsitektur, cara lokal, daftar env, cara migrasi, catatan deploy Render + Supabase (tanpa nilai secret).

## 10. Aturan Commit

- Commit **per fitur/perubahan**; test tetap di commit terpisah bila menambah test.
- Jangan `git add .`; stage file eksplisit.
- Jangan force-push, amend, rebase, atau squash.
- Jangan commit `.env`, node_modules, dist, atau nilai secret.
- Verifikasi **tidak** dilakukan per commit; lakukan sekaligus sebelum push:
  ```bash
  npm run build && npm run typecheck && npm run lint && npm test
  ```
- Push ke `main` hanya setelah verifikasi lulus, dan laporkan hasilnya.

## 11. Kriteria Selesai (Gate C)

1. Project Supabase ada; 5 migrasi tercatat di `schema_migrations`; admin ter-seed (login ID dicatat di log).
2. `src/db/client.ts` mendukung SSL Supabase.
3. `render.yaml` ada, valid, tanpa secret.
4. Service Render live; `/health` dan `/health/ready` mengembalikan 200.
5. `npm run build`, `typecheck`, `lint`, `test` lulus (jika test butuh DB, laporkan hasil apa adanya; jangan membaca/mengubah konfigurasi DB produksi`.
6. URL `<service>.onrender.com` dilaporkan ke Lead/user.
7. `pam.sequein.my.id` dan repo Android tidak tersentuh.

## 12. Larangan

- Jangan mengubah domain/backend lama `pam.sequein.my.id`.
- Jangan mengubah repo Android `0033_EduVARK`.
- Jangan memakai `LOCAL_STORAGE_DIR` di production.
- Jangan menaruh `DATABASE_URL` direct di Render (runtime harus pooler).
- Jangan menuliskan secret di commit, laporan, atau screenshot.

## 13. Format Laporan Gate C

Kirim laporan ringkas berisi:

1. URL service Render final.
2. Daftar commit yang dibuat (`git log --oneline <baseline>..HEAD`).
3. Hasil `npm run build`, `typecheck`, `lint`, `test` (lulus/gagal + ringkasan).
4. Bukti migrasi: 5 baris `schema_migrations` (tanpa menampilkan password/URL).
5. Hasil `curl -i https://<service>.onrender.com/health` dan `/health/ready`.
6. Env vars yang sudah diisi (nama saja) dan yang masih kosong.
7. Kendala/risiko yang ditemui, jika ada.
