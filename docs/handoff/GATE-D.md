# GATE D — Deploy, Migrasi, Seed, dan Verifikasi End-to-End

Dokumen ini adalah instruksi kerja untuk agent yang mengerjakan **Gate D**. Gate D dijalankan **setelah Gate C selesai** dan URL Render sudah diketahui. Bacalah seluruh dokumen sebelum melakukan apa pun.

## 0. Konteks & Pembagian Tugas

- **Repo backend:** `https://github.com/mamaydev/0033_EduVARK-backend` (branch `main`), di-deploy ke **Render** (Docker).
- **Database:** Supabase (dibuat di Gate C), migrasi manual via direct connection.
- **Storage:** Cloudflare R2 (dibuat di Gate C).
- **Tugas Gate D:** deploy service, terapkan migrasi + seed, lalu verifikasi end-to-end semua endpoint dan alur aplikasi.
- **Bukan tugas Anda:** mengubah repo Android. Lead akan memperbarui `API_BASE_URL` APK release ke URL Render lalu meminta Anda memverifikasi APK tersebut (langkah §8).
- **Backend lama `pam.sequein.my.id` tidak boleh diubah/disentuh.** Backend baru memakai URL default Render; tidak ada custom domain.

## 1. Prasyarat

- Gate C selesai: service Render live, URL `<service>.onrender.com` diketahui, env vars terisi.
- Anda memegang (secara aman, tidak ditulis ke repo/laporan):
  - password admin hasil seed,
  - kredensial Supabase (direct URL untuk migrasi/seed),
  - kredensial R2 sudah terpasang di Render.
- Alat bantu: `curl`, `jq` (opsional), Node >= 24 bila menjalankan script lokal.

## 2. Deploy

Jika Gate C sudah membuat service Render dengan `autoDeploy`, push ke `main` memicu deploy otomatis. Jika belum:

1. Render Dashboard → **New → Web Service** → repo `0033_EduVARK-backend`.
2. Runtime **Docker**, branch `main`, region Singapore, Dockerfile di root.
3. Health check path `/health/ready`.
4. Deploy hingga status **live**.

Catat URL final: `https://<service>.onrender.com`.

## 3. Migrasi Database & Seed Admin

Jalankan **sekali** dari mesin lokal dengan **direct connection** Supabase (port 5432), bukan pooler:

```bash
cd 0033_EduVARK-backend
npm ci

DATABASE_URL="<DIRECT_URL_SUPABASE>" npm run db:migrate
DATABASE_URL="<DIRECT_URL_SUPABASE>" \
ADMIN_PASSWORD="<PASSWORD_ADMIN>" \
ADMIN_FULL_NAME="Administrator" \
npm run db:seed-admin
```

Ekspektasi:
- `schema_migrations` berisi 5 baris migrasi (`001_init` … `005_material_file_replacements`).
- Seed mencatat **login ID admin** (pola `014014XXXXX`) di log. Simpan login ID + password untuk verifikasi.
- Admin hasil seed punya `mustChangePassword = true`; wajib ganti password pada login pertama.

Jika migrasi sudah dijalankan di Gate C, jangan ulangi tanpa memastikan tabel ada (perintah idempoten, tetapi laporkan bila dijalankan ulang).

## 4. Verifikasi Dasar (Health)

```bash
curl -i https://<service>.onrender.com/health
curl -i https://<service>.onrender.com/health/ready
```

Harapan: `200`, body `{"data":{"status":"ok"|"ready",...}}`. `/health/ready` mengembalikan `503` bila database belum siap → periksa env `DATABASE_URL` (pooler) dan SSL.

**Penting:** pada `NODE_ENV=production`, middleware kompatibilitas menolak request API tanpa header versi dengan `426 CLIENT_VERSION_UNSUPPORTED`. Semua request API di bawah ini wajib menyertakan:

```text
x-eduvark-client-version: 2
```

(nilai minimal default `CLIENT_MIN_VERSION=2`). Health tidak memerlukan header.

## 5. Verifikasi Autentikasi

Ganti `BASE=https://<service>.onrender.com/api/v1` dan `CLIENT_HEADER='x-eduvark-client-version: 2'`.

1. **Login admin pertama** (login ID dari log seed):
   ```bash
   curl -s -X POST "$BASE/auth/login" \
     -H 'Content-Type: application/json' -H "$CLIENT_HEADER" \
     -d '{"loginId":"<ADMIN_LOGIN_ID>","password":"<ADMIN_PASSWORD>"}' | jq
   ```
   Harapan: `200`, `data.accessToken`, `data.refreshToken`, `data.user.role == "ADMIN"`, `data.user.mustChangePassword == true`.

2. **Ganti password admin** (wajib sebelum endpoint admin lain):
   ```bash
   TOKEN=<accessToken>
   curl -s -X POST "$BASE/auth/change-password" \
     -H 'Content-Type: application/json' -H "$CLIENT_HEADER" \
     -H "Authorization: Bearer $TOKEN" \
     -d '{"currentPassword":"<ADMIN_PASSWORD>","newPassword":"<PASSWORD_BARU>","confirmPassword":"<PASSWORD_BARU>"}' | jq
   ```
   Login ulang dengan password baru; `mustChangePassword` harus `false`.

3. **Refresh & logout:**
   ```bash
   curl -s -X POST "$BASE/auth/refresh" -H 'Content-Type: application/json' -H "$CLIENT_HEADER" \
     -d '{"refreshToken":"<refreshToken>"}' | jq
   curl -s -X POST "$BASE/auth/logout" -H "$CLIENT_HEADER" -H "Authorization: Bearer <TOKEN>" | jq
   ```
4. **`GET /auth/me`** mengembalikan profil user yang login.

## 6. Verifikasi Modul

Gunakan data yang benar-benar dibuat (bukan data asumsi). Untuk alur penuh, buat dosen dan mahasiswa melalui API admin:

1. **Admin users** (role ADMIN, sudah ganti password):
   - `POST $BASE/admin/users` → buat dosen (`role: "LECTURER"`, identitas NIDN 10–11 digit).
   - `POST $BASE/admin/users` → buat mahasiswa (`role: "STUDENT"`, identitas NIM).
   - `GET $BASE/admin/users`, `GET $BASE/admin/users/:id`, `PATCH $BASE/admin/users/:id`.
   - Login dosen/mahasiswa pertama, ganti password masing-masing (`mustChangePassword`).
   - `POST $BASE/admin/users/:id/reset-password` dan `POST $BASE/admin/users/:id/deactivate` — verifikasi akun nonaktif tidak bisa login.
   - Student harus memilih VARK pada login pertama: `POST $BASE/auth/vark` dengan `{"vark":"VISUAL"|"AURAL"|"READ_WRITE"|"KINESTHETIC"}`.

2. **Topics (dosen):** `POST/PATCH/DELETE $BASE/lecturer/topics`, `PATCH /lecturer/topics/reorder`, `POST /lecturer/topics/:id/publish`, `POST /lecturer/topics/:id/unpublish`.
3. **Materials (dosen + R2):**
   - `POST $BASE/lecturer/topics/:id/materials/link` (materi tautan).
   - `POST $BASE/lecturer/topics/:id/materials/file` (multipart field `file`) → harus sukses menulis ke R2.
   - `PATCH $BASE/lecturer/materials/:id`, `POST $BASE/lecturer/materials/:id/file` (ganti file).
   - `GET $BASE/materials/:id/content` (student/dosen) → harus mengunduh dari R2; **jangan** mengharapkan URL R2 terbuka langsung. `GET $BASE/materials/:id/link-target` untuk materi tautan.
   - `DELETE $BASE/lecturer/materials/:id` → objek R2 masuk backlog penghapusan; periksa worker (`STORAGE_DELETION_INTERVAL_MS`) dan log tidak error.
4. **Feedback (mahasiswa):** `PUT` lalu `GET $BASE/topics/:id/feedback`.
5. **Cases (dosen):** `PUT $BASE/lecturer/topics/:id/case`; mahasiswa `GET $BASE/topics/:id/case`, `POST $BASE/topics/:id/case/submission`.
6. **Assessments (dosen):** `GET $BASE/lecturer/cases/:id/submissions`, `POST $BASE/submissions/:id/assessment`; mahasiswa `GET $BASE/submissions/:id`.
7. **Analytics (dosen):** `GET $BASE/lecturer/analytics/topics/:id`.
8. **Dashboard mahasiswa:** `GET $BASE/student/dashboard` (butuh VARK terisi).
9. **Kompatibilitas klien:** request API tanpa header versi harus `426`; dengan `x-eduvark-client-version: 2` harus lolos.

Setiap pengecekan dicatat: endpoint, kode HTTP, hasil (lulus/gagal), dan pesan error bila ada. Jangan menandai lulus tanpa benar-benar memanggil endpoint.

## 7. Verifikasi Storage R2

- Upload file materi berukuran wajar (mis. < 5 MB) lewat endpoint `materials/file` → sukses.
- Unduh lewat `GET /materials/:id/content` → isi file identik (cek `sha256`).
- Pastikan bucket R2 menerima objek (cek dashboard R2 atau log backend).
- Ganti file (`/materials/:id/file`) lalu unduh ulang → isi baru; objek lama masuk backlog.
- **Jangan** mengubah konfigurasi R2 produksi tanpa laporan.

## 8. Verifikasi APK Release (setelah Lead memperbarui Android)

1. Lead akan memperbarui `API_BASE_URL` release di repo Android `0033_EduVARK` ke `https://<service>.onrender.com/api/v1/` dan membangun APK.
2. Setelah APK tersedia:
   - Install/emulator, login admin/dosen/mahasiswa.
   - Uji alur utama: login, ganti password, pilih VARK, buat topik, unggah materi, buka materi, isi feedback, kumpulkan case, nilai submission, analytics.
   - Pastikan tidak ada request ke `pam.sequein.my.id` (cek log backend/curl di device bila memungkinkan).
3. Jika ada kegagalan yang berasal dari backend, laporkan endpoint + request + respons (tanpa token/password).

## 9. Rollback

- **Render:** gunakan menu **Rollback** ke deploy terakhir yang sehat, atau revert commit bermasalah (jangan force-push). Laporkan penyebab.
- **Database:** migrasi bersifat additif/forward-only; jangan menghapus tabel/kolom produksi. Bila perlu restore, gunakan backup Supabase dan laporkan dampaknya.
- **Storage:** jangan menghapus bucket R2; gunakan backlog worker untuk pembersihan.

## 10. Kriteria Selesai (Gate D)

1. Service Render live, `/health` dan `/health/ready` `200`.
2. 5 migrasi tercatat; admin ter-seed dan password awal berhasil diganti.
3. Semua modul §6 lulus minimal satu alur sukses per endpoint penting.
4. Upload/unduh/ganti/hapus materi berfungsi dengan R2.
5. Header kompatibilitas klien ditegakkan (`426` tanpa header, lolos dengan header).
6. APK release (setelah diperbarui Lead) dapat login dan menjalankan alur utama ke URL Render.
7. `pam.sequein.my.id` dan repo Android tidak diubah oleh Gate D.

## 11. Format Laporan Gate D

Kirim laporan berisi:

1. URL service dan status deploy terakhir (commit hash + waktu).
2. Hasil health (`/health`, `/health/ready`) dengan kode HTTP.
3. Bukti migrasi (id migrasi tanpa URL/password) dan status seed.
4. Tabel verifikasi per modul: endpoint, kode HTTP, lulus/gagal, catatan.
5. Hasil verifikasi R2 (upload/unduh hash, ganti file, backlog).
6. Hasil verifikasi APK release (bila sudah dilakukan).
7. Kendala, error, atau risiko yang tersisa; langkah lanjutan bila ada.
8. Konfirmasi bahwa `pam.sequein.my.id` dan repo Android tidak tersentuh.
