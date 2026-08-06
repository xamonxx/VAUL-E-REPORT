# Change Log

Catat setiap perubahan kode, konfigurasi, schema, deployment, atau dokumentasi penting di sini.

## Template

```md
### YYYY-MM-DD - Judul singkat

- Area: backend / frontend / database / ops / docs
- Ringkasan:
- File berubah:
- Risiko / follow-up:
- Agent:
```

## Entries

### 2026-08-04 - Video tutorial admin Bagian 1 (Login & Absen)

- Area: docs / media
- Ringkasan: Buat video panduan admin E-Report Bagian 1 (login → diarahkan absen) pakai HyperFrames, 9:16 vertikal, ~47 detik. Semua layar = **capture asli app** (login, dashboard + banner "LAPOR ABSEN", halaman `/report-attendances` 3 kategori) diambil HD 1080-wide via Playwright (login dijalankan user; aku tak ketik password). Motion: kamera zoom/pan, kursor + ripple klik, caption Bahasa Indonesia, SFX klik offline. Render MP4 sukses (19.8 MB). Voiceover ID belum (menunggu `npx hyperframes auth login` HeyGen). Data app tak diubah (baris absen user 98 tak dihapus — memang belum absen).
- File berubah:
  - `videos/bagian1-login-absen/` (project HyperFrames baru: `index.html`, `assets/img/*`, `assets/audio/click.wav`, `renders/*.mp4`)
- Risiko / follow-up: Bagian 2 = tatacara isi data konsumen. Tambah voiceover Indonesia setelah sign-in HeyGen lalu render ulang. Aset capture + `auth_state.json` ada di scratchpad sesi.
- Agent: REMOX

### 2026-08-04 - Review aaPanel Security Scan + ACT-013 (serverone)

- Area: ops / security
- Ringkasan: Review 9 risiko dari aaPanel Security Scan (skor 85) pada `serverone`. Hasil verifikasi read-only: praktis tak ada yang nyata — 2 false-positive (SSH idle timeout sudah `ClientAliveInterval 300` efektif via drop-in; min-password-interval sudah `PASS_MIN_DAYS=1`), FTP N/A (pure-ftpd mati), 3 upsell (panel alarm/whitelist/all-SSL → tekan "Mengabaikan"), 2 area-Nginx-dilindungi (website SSL & anti-cross-site → jangan diserahkan ke aaPanel; SSL sudah di Cloudflare tunnel, app set CSP sendiri), 1 SUID/SGID (582 file normal, jangan one-click). **ACT-013** set `PASS_MIN_DAYS=1` (login.defs + `mki`) — ternyata sudah patuh (`mki` & `root` min=1), jadi no-op idempoten, hanya backup dibuat. Peringatan dicatat: JANGAN pakai "Perbaikan satu klik" aaPanel (bundel sshd+nginx+suid, risiko putus akses & sentuh area dilindungi).
- File berubah:
  - `E-REPORT-APP/08-Ops/Server-Audit-2026-08-04.md` (tambah ACT-013 + bagian G2 review scan)
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
  - Server `serverone` (luar repo): `/etc/login.defs` (no-op, backup `.bak-2026-08-04`)
- Risiko / follow-up: Nol perubahan efektif (sistem sudah patuh). Rollback `login.defs.bak-2026-08-04` bila perlu. Rescan aaPanel mestinya bersihkan item false-positive.
- Agent: REMOX

### 2026-08-04 - Audit & hardening server home lab (serverone)

- Area: ops / security
- Ringkasan: Audit menyeluruh mini PC `serverone` (Ubuntu 24.04.4, i3-7100U, 15GiB) via SSH Tailscale, mode read-only dulu lalu tindakan per-persetujuan (`ACT-xxx`). Nginx & Cloudflare/cloudflared diperlakukan read-only (tak disentuh). Dieksekusi: **ACT-001** stop container `hermes` (AI-agent eksperimen expose 0.0.0.0:8642/9119 + probe host tailscale:8443); **BATCH-A** vacuum journald 2.2G→481M + 3 security update (tzdata/tzdata-legacy/distro-info-data) + apt clean; **ACT-006** hapus permanen hermes (container+image 4.59GB+network+def CasaOS+data `/DATA/AppData/hermes`); **ACT-007** salin `/www/backup` ke disk sda `/data` (redundansi); **ACT-007b** cron auto-sync harian 02:00 (snapshot bertanggal + retensi 30h, skrip `/usr/local/bin/backup-sync-sda.sh`); **ACT-008** hapus sisa host `/home/mki/.hermes` (2.2G); **ACT-005** upgrade Docker 29.6.2→29.7.1 + Node 22.23.1→22.23.2 (maintenance window, 3 container auto-restart bersih); **ACT-009** fix fail2ban jail sshd `port = 2201` (sebelumnya ban salah port 22, terbukti efektif via uji ban-dummy). Ruang dibebaskan ~7,7GB; update 8→0. Temuan tersisa: NVMe 342 unsafe-shutdown (akar = daya, saran UPS + smartd), Media/Data Integrity Errors 3.428 kemungkinan artefak firmware SX8200 (Spare 100%, Error-Log 0, Critical 0x00) — drive sehat, ~6-7th sisa umur.
- File berubah:
  - `E-REPORT-APP/08-Ops/Server-Audit-2026-08-04.md` (baru)
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
  - Server `serverone` (di luar repo): container/image hermes dihapus, journald vacuum, paket upgrade, cron+skrip backup baru, `jail.local` fail2ban (backup `.bak-2026-08-04`)
- Risiko / follow-up: ACT-006/008 irreversible (hermes hapus permanen). Rollback tersedia: ACT-007b (hapus cron+skrip), ACT-009 (`jail.local.bak-2026-08-04`). Terbuka: UPS+NUT (akar unsafe-shutdown), smartd monitoring (ACT-010), rclone off-site (ACT-012, butuh kredensial), hardening SSH pubkey-only (verifikasi key `mki` dulu). Catatan: edit jail SSH via UI aaPanel Fail2ban Manager bisa menimpa `port = 2201`.
- Agent: REMOX

### 2026-08-03 - Audit keamanan menyeluruh dan hardening PWA

- Area: backend auth/role/notification, Web Push, frontend PWA, deployment, audit
- Ringkasan: re-audit working tree dengan fixture sintetis; redaksi kredensial produksi pada `DEPLOYMENT.md`; penambahan `dynamicStartUrl: false`; regression check service worker; browser smoke desktop/mobile untuk protected-route guard dengan assertion tab authenticated; harness performance desktop/mobile; validasi dependency, build, lint, migration, route, dan feature test.
- File utama: `E-REPORT-BACKEND_API/DEPLOYMENT.md`, `E-REPORT-FRONTEND/.gitignore`, `E-REPORT-FRONTEND/next.config.js`, `E-REPORT-FRONTEND/package.json`, `E-REPORT-FRONTEND/scripts/verify-pwa-security.mjs`, `E-REPORT-FRONTEND/scripts/browser_smoke_audit.py`, `E-REPORT-FRONTEND/scripts/performance_audit.py`, `E-REPORT-APP/12-Security/Security-Bug-Audit-2026-08-03.md`
- Risiko/follow-up: credential database masih ada di histori Git dan harus dirotasi/purge; HTTPS/HSTS produksi belum terbukti; authenticated Android PWA/Web Push perlu uji dengan akun sintetis valid.
- Agent: Codex

### 2026-08-03 - Audit performa aplikasi

- Area: frontend performance, backend query, testing, docs
- Ringkasan: build production dan benchmark Playwright desktop/Android tiga run per route; ukur TTFB, DOM ready, LCP, CLS, long task, transfer size, API lokal, dashboard query, analytics query, dan bundle chunk. Tambah harness `test:performance` serta laporan performa.
- File utama: `E-REPORT-FRONTEND/scripts/performance_audit.py`, `E-REPORT-FRONTEND/package.json`, `E-REPORT-APP/13-Testing/Performance-Audit-2026-08-03.md`
- Risiko/follow-up: authenticated benchmark memerlukan credential fixture valid; chunk MapLibre sekitar 970 KB uncompressed dan initial transfer sekitar 433-519 KB perlu backlog optimasi.
- Agent: Codex

### 2026-08-03 - Quick wins performa frontend

- Area: frontend performance
- Ringkasan: hentikan preload font mono yang tidak dipakai pada first paint; naikkan cache master-data menjadi 5 menit dengan invalidasi mutation; hentikan refetch notification summary pada setiap window focus karena query sudah aktif saat panel dibuka dan memiliki polling fallback.
- File utama: `E-REPORT-FRONTEND/src/app/layout.tsx`, `E-REPORT-FRONTEND/src/lib/hooks/useMasterData.ts`, `E-REPORT-FRONTEND/src/lib/hooks/useNotifications.ts`
- Risiko/follow-up: authenticated benchmark dan optimasi MapLibre/initial private shell tetap pending.
- Agent: Codex

### 2026-08-03 - Fix absensi 500 jika notifikasi gagal / migration belum jalan

- Area: backend / database / docs
- Ringkasan: Toast `Server sedang bermasalah` di halaman Absensi terjadi karena row `report_attendances` berhasil masuk, lalu proses notifikasi superadmin crash saat tabel `attendance_notifications` belum dimigrate (`SQLSTATE[42S02]`). Fix: proses notifikasi absensi dibungkus `try/catch` dan di-log, sehingga kegagalan notifikasi/Web Push tidak membatalkan submit absensi. Migration lokal `attendance_notifications` dijalankan. Backfill notifikasi untuk absensi hari ini yang sempat tersimpan sebelum crash, lalu cache aplikasi dibersihkan.
- File berubah:
  - `E-REPORT-BACKEND_API/app/Http/Controllers/Api/ReportAttendanceController.php`
  - Database lokal: migration `2026_08_03_000001_create_attendance_notifications_table`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Production tetap wajib menjalankan `php artisan migrate`. Jika notifikasi gagal karena masalah infra, absensi tetap sukses dan error masuk log `Attendance notification failed`.
- Agent: Codex

### 2026-08-03 - Redesign panel notifikasi + hilangkan flash saat pindah tab

- Area: frontend / docs
- Ringkasan: Panel notifikasi didesain ulang agar lebih lega dan tab tidak lagi terpotong (`Survey`, `Absen`, `Catat`, `Ingat`). Bug flash/kedip saat pindah tab diperbaiki dengan mengganti banyak `TabsContent` menjadi satu area konten stabil (`ScrollArea` tunggal + `renderNotificationPanel()`), sehingga body popover tidak unmount/remount saat tab berubah. Ukuran popover dinaikkan sedikit, tab dipadatkan, dan empty/loading state tetap memakai tinggi tetap agar modal tidak loncat.
- File berubah:
  - `E-REPORT-FRONTEND/src/components/layout/notification-center.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Build pertama sempat gagal transient di page collection Next (`/geo-analytics`, `/login`) walau file route ada; rerun build sukses. Verifikasi final: `npm run lint` bersih, `npm run build` sukses.
- Agent: Codex

### 2026-08-03 - Notifikasi absensi admin ke superadmin + Web Push PWA

- Area: backend / frontend / database / docs
- Ringkasan: Saat admin sukses mengirim absensi harian, backend membuat notifikasi absensi hanya untuk user role `super_admin`, mengirim Web Push PWA ke subscription superadmin, dan menghapus cache notifikasi superadmin. Ditambah tabel `attendance_notifications` dengan relasi ke `report_attendances`, endpoint read/delete khusus `/notifications/attendances/{notification}`, count `unread_attendances`, serta validasi server-side agar role non-superadmin tetap melihat 0/kosong meski ada data salah. Panel notifikasi frontend kini punya tab `Absensi` khusus superadmin dengan kartu informatif berisi admin, akun, tanggal, dan kategori absensi; tab dibuat responsif agar label tidak menabrak di Android/PWA.
- File berubah:
  - `E-REPORT-BACKEND_API/database/migrations/2026_08_03_000001_create_attendance_notifications_table.php`
  - `E-REPORT-BACKEND_API/app/Models/AttendanceNotification.php`
  - `E-REPORT-BACKEND_API/app/Http/Controllers/Api/ReportAttendanceController.php`
  - `E-REPORT-BACKEND_API/app/Http/Controllers/Api/NotificationController.php`
  - `E-REPORT-BACKEND_API/app/Services/NotificationSummaryService.php`
  - `E-REPORT-BACKEND_API/routes/api.php`
  - `E-REPORT-FRONTEND/src/components/layout/notification-center.tsx`
  - `E-REPORT-FRONTEND/src/lib/hooks/useNotifications.ts`
  - `E-REPORT-FRONTEND/src/types/index.ts`
  - `E-REPORT-APP/05-Database/Data-Model.md`
  - `E-REPORT-APP/06-API/API-Catalog.md`
  - `E-REPORT-APP/11-Modules/Notifications.md`
  - `E-REPORT-APP/09-Decisions/Decision-Log.md`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Deploy wajib `php artisan migrate`; produksi PWA tetap perlu VAPID key valid, HTTPS, dan user superadmin menekan `Aktifkan HP` di perangkat Android. Belum ada test suite backend di repo, jadi verifikasi dilakukan lewat lint/syntax/build.
- Agent: Codex

### 2026-08-03 - Build bersih tanpa warning (lint cleanup)

- Area: frontend / ops
- Ringkasan: `npm run build` sebelumnya sukses tapi 10 warning ESLint. Dibersihkan: (1) matikan rule `@next/next/no-img-element` di `eslint.config.mjs` — logo/avatar disajikan dari host API yang di-resolve runtime (localhost/IP LAN/produksi), `next/image` remotePatterns tak bisa match host dinamis jadi `<img>` disengaja (7 warning: accounts, settings, header); (2) hapus import & var mati di `master-data-view.tsx` (`CardDescription/CardHeader/CardTitle`, `DialogTrigger`, `setStatusesPage`→`const [statusesPage]`, `updateCat`) (6 warning); (3) hapus 2 komentar `eslint-disable-next-line no-img-element` yang jadi redundant setelah rule off (bug-reports-view, bug-report-widget). Build final: `✓ Compiled successfully`, nol warning/error, exit 0.
- File berubah:
  - `E-REPORT-FRONTEND/eslint.config.mjs`
  - `E-REPORT-FRONTEND/src/app/master-data/master-data-view.tsx`
  - `E-REPORT-FRONTEND/src/app/bug-reports/bug-reports-view.tsx`
  - `E-REPORT-FRONTEND/src/components/bug-report/bug-report-widget.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Nol. Bila kelak host gambar dibuat statis/tunggal, bisa migrasi ke `next/image` + remotePatterns lalu nyalakan lagi rule-nya.
- Agent: REMOX

### 2026-08-03 - Bersihkan data dummy (pertahankan akun, semua user, master data)

- Area: database / ops
- Ringkasan: Atas permintaan pemilik, hapus semua data dummy/transaksi TAPI pertahankan Akun Interior, SEMUA user (semua role), dan Master Data. Backup DB dibuat dulu (`storage/backup_e_report_20260803_114021.sql`, mysqldump, terverifikasi). Truncate (FK dimatikan sementara): consultations (360), consultation_needs_category (1008), consultation_notes (19), consultation_status_histories (1), surveys (261), survey_activity_logs (616), survey_status_histories (615), survey_notifications (212), survey_reschedules (1), audit_logs (24), login_attempts (81), report_attendances (3), consultation_sequences (22, reset counter), + reminders/bug_reports/push_subscriptions/consultation_imports (sudah 0). DIPERTAHANKAN: accounts (22), users (44), account_user, needs_categories (39), status_categories (10), survey_statuses (5). Catatan: `ResetForLaunch`/`ResetForProduction` TIDAK dipakai karena keduanya menghapus user (non-admin/admin), melanggar syarat "ALL USER jangan dihapus".
- File berubah: (data DB saja, tak ada perubahan kode)
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Irreversible. Restore via backup SQL bila perlu: `mysql -u root e_report < storage/backup_e_report_20260803_114021.sql`. consultation_sequences kosong → nomor lead baru mulai dari 1 per akun (dibuat otomatis firstOrCreate). Verifikasi: sisa akun/user/master sesuai target.
- Agent: REMOX

### 2026-08-03 - Validasi realtime input (cegah karakter/panjang melanggar rule)

- Area: frontend
- Ringkasan: Terapkan pencegahan realtime pada input sesuai rule backend. **Form Akun** (`accounts-view`): nama `maxLength=100` + strip `<>` saat ketik + validasi realtime "minimal 3 karakter" (merah) + counter `x/100`; Target Bulanan ganti `type=number` → `type=text inputMode=numeric`, strip non-digit + clamp ≤ 1.000.000 realtime (cegah negatif/desimal/kelebihan). **Form Profil** (`settings-view`): nama strip `<>` realtime (samakan backend `regex:^[^<>]+$`), `maxLength=255` sudah ada. Backend tetap validasi final (defense-in-depth).
- File berubah:
  - `E-REPORT-FRONTEND/src/app/accounts/accounts-view.tsx`
  - `E-REPORT-FRONTEND/src/app/settings/settings-view.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Nol. Typecheck lolos. Rule dicek dari `AccountRequest` (name min:3/max:100, target_leads int 1..1000000) & `SettingsController` (name max:255 + no `<>`).
- Agent: REMOX

### 2026-08-03 - Fix: nama panjang overflow header + maxLength input profil

- Area: frontend
- Ringkasan: Nama super panjang (tanpa batas di input) bikin overflow di header (trigger profil `shrink-0` melebar, tak terpotong). Fix: (1) header trigger nama+role dibungkus `min-w-0 max-w-[160px]` + `truncate`; (2) input Nama & Email di settings ditambah `maxLength={255}` (samakan backend `max:255`). Backend sudah validasi `name max:255` — ini guard frontend + perbaikan tampilan.
- File berubah:
  - `E-REPORT-FRONTEND/src/components/layout/header.tsx`
  - `E-REPORT-FRONTEND/src/app/settings/settings-view.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Nama super panjang yang SUDAH tersimpan kini tampil terpotong (bagus), user bisa edit pendekkan. Input akun (`accounts-view`) belum diberi maxLength — pertimbangkan bila perlu. Typecheck lolos.
- Agent: REMOX

### 2026-08-03 - Fitur upload foto profil (crop + preview) + hardening upload

- Area: frontend / backend / database / security
- Ringkasan: Tambah fitur unggah foto profil dengan crop lingkaran (pan/zoom) + preview, tersimpan instan setelah crop. **Backend:** migration kolom `users.avatar_path` (nullable); `User::$fillable` +`avatar_path`; endpoint baru `POST /settings/avatar` (`SettingsController::updateAvatar`) terima file `avatar` atau flag `remove_avatar=1`, pola contek `AccountController` (store `avatars`/public, hapus file lama, nama acak `hashName`); `throttle:20,1`; `AuthController::formatUser` +`'avatar' => $user->avatar_path` (persisten via `/auth/me`). **Frontend:** dep `react-easy-crop@6.2.3`; `AuthUser.avatar`; hook `useUpdateAvatar` (postForm, merge store `setUser({...user, avatar})` seperti `useUpdateTheme`); komponen `components/settings/avatar-cropper.tsx` (crop → canvas 512x512 PNG → Blob, re-encode strip EXIF); seksi Foto Profil di settings (Unggah/Ganti/Hapus + loading); header ganti 2 fallback inisial jadi `<img>` bila `user.avatar`. **Validasi & keamanan:** backend `image|mimes:jpeg,png,jpg,gif,webp|max:2048|dimensions:max 2000x2000` (SVG ditolak—anti stored-XSS) + pesan Indonesia jelas; frontend pra-cek size/type sebelum crop (toast "Ukuran foto maksimal 2 MB."); error server lewat `getErrorMessage`.
- File berubah:
  - Backend: `database/migrations/2026_08_03_000001_add_avatar_path_to_users_table.php` (baru), `app/Models/User.php`, `app/Http/Controllers/Api/SettingsController.php`, `app/Http/Controllers/Api/AuthController.php`, `routes/api.php`
  - Frontend: `package.json` (react-easy-crop), `src/types/index.ts`, `src/lib/hooks/useSettings.ts`, `src/components/settings/avatar-cropper.tsx` (baru), `src/app/settings/settings-view.tsx`, `src/components/layout/header.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Dev server frontend WAJIB restart (dependency baru). Deploy WAJIB `php artisan migrate`. Verifikasi (via token sementara, sudah dicabut): tanpa-file→422 "Pilih foto terlebih dahulu.", non-gambar→422 "Berkas harus berupa gambar.", upload PNG valid→tersimpan+URL 200, hapus→null+file hilang. Typecheck `tsc --noEmit` bersih. Avatar reuse disk public (symlink `storage:link` sudah ada, lihat [[deploy-storage-link]]).
- Agent: REMOX

### 2026-08-03 - Badge akun di header pakai logo asli (kosong bila tak ada)

- Area: frontend
- Ringkasan: Badge konteks akun di header menampilkan huruf pertama nama (mis. "H") di kotak warna, bukan logo. Diganti: kalau `user.account.logo` ada, tampilkan `<img>` logo (`${api.baseUrl}/storage/${logo}`, 5x5 rounded object-cover); kalau tidak ada, dikosongkan (tak render badge huruf). Format `account.logo` = path mentah `logo_path` (backend `AuthController` line 169), URL dibangun sama seperti `AccountLogo`. Import `api` ditambah ke header. Else-branch tanpa-akun (Shield icon) tak diubah.
- File berubah:
  - `E-REPORT-FRONTEND/src/components/layout/header.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Nol. Typecheck lolos. `primaryColor` masih dipakai di elemen lain, tak jadi unused.
- Agent: REMOX

### 2026-08-03 - Fix: preview logo nyangkut antar-akun (logoPreview tak di-reset)

- Area: frontend
- Ringkasan: Buka form Edit akun A, pilih logo baru → preview muncul. Buka Edit akun B, preview masih tampilkan logo pilihan akun A (stale). Sebab: `handleOpenCreate`/`handleOpenEdit` reset `logoFile` & `removeLogo` tapi lupa `logoPreview` (state data URL dari FileReader yang ditambah sebelumnya). Fix: tambah `setLogoPreview(null)` di kedua handler saat dialog dibuka.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/accounts/accounts-view.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Nol. Typecheck lolos. HMR langsung apply (kode komponen, bukan config).
- Agent: REMOX

### 2026-08-03 - Fix: CSP blokir logo di dev (img-src tanpa http:)

- Area: frontend / security
- Ringkasan: Setelah symlink storage dibuat, logo masih broken di dev. Sebab asli: header CSP `img-src 'self' data: blob: https:` — izinkan `https:` tapi bukan `http:` dari origin lain. Frontend dev di `http://localhost:3000`, logo di `http://localhost:8000` (http, beda origin) → console: `violates ... img-src ... The action has been blocked`. Di produksi API-nya `https://api-ereport.interiorcustom.id` jadi lolos (`https:` di-allow), makanya bug ini cuma muncul di dev/LAN. Fix: tambah `http:` ke `img-src` hanya saat non-produksi — `` `img-src 'self' data: blob: https:${isProduction ? '' : ' http:'}` `` — mengikuti pola `connect-src` yang sudah ada. Produksi tetap ketat (https saja).
- File berubah:
  - `E-REPORT-FRONTEND/next.config.js`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Perubahan `next.config.js` (header) **tidak hot-reload** — dev server WAJIB restart. Produksi tak terpengaruh (http tak ditambah saat production). Alternatif lebih rapi (belum dikerjakan): proxy `/storage` lewat rewrites Next biar logo jadi same-origin, hilangkan kebutuhan cross-origin sama sekali.
- Agent: REMOX

### 2026-08-03 - Fix: logo akun tidak tampil (symlink storage hilang)

- Area: ops / backend
- Ringkasan: Logo akun broken di list & form (URL `/storage/accounts/*.png` → 404). Akar masalah: symlink `public/storage` di backend hilang, jadi Laravel tak bisa serve aset publik. Fix: `php artisan storage:link` — bikin ulang `public/storage` → `storage/app/public`. Verifikasi: logo URL balik `200 OK`. Bukan bug kode frontend; image preview yang ditambah sebelumnya tetap valid.
- File berubah:
  - `E-REPORT-BACKEND_API/public/storage` (symlink dibuat ulang)
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Symlink bisa hilang lagi setelah clone/deploy fresh — pastikan `php artisan storage:link` masuk langkah deploy. Cek juga di produksi apakah symlink ada.
- Agent: REMOX

### 2026-08-03 - Tambah image preview di form upload logo (Edit Detail Akun)

- Area: frontend
- Ringkasan: Form "Edit Detail Akun" — logo hanya tampil sebagai text/icon, tidak ada image preview. Sekarang: (1) logo lama ditampilkan sebagai `<img>` thumbnail (10x10, rounded, object-cover) di atas dropzone, bukan cuma icon + "Logo terunggah aktif"; (2) saat user pilih file baru, `FileReader` buat data URL, dropzone tampilkan `<img>` preview (h-20, object-contain) alih-alih text filename; (3) kalau cancel file, preview hilang balik ke placeholder "Pilih file logo..." + icon upload. Preview real-time, tidak perlu submit.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/accounts/accounts-view.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Nol. File preview local (data URL di memory), tidak ada POST sampai submit. Typecheck lolos.
- Agent: REMOX

### 2026-08-03 - Fix: filename terpotong di dropzone upload logo (Edit Detail Akun)

- Area: frontend
- Ringkasan: Form "Edit Detail Akun" — dropzone upload logo menampilkan filename panjang (UUID) tanpa truncate. Sebab: flex container tidak punya `min-w-0`, jadi Tailwind `truncate` tidak bekerja di flex child. Tambah `min-w-0` ke div dropzone + `shrink-0` ke icon Upload — filename sekarang terpotong elipsis bila melebihi space.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/accounts/accounts-view.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Nol. Perubahan pure presentasi, tak sentuh logika upload/form.
- Agent: REMOX

### 2026-08-03 - Pesan error API/network jadi mudah dipahami (normalisasi terpusat)

- Area: frontend
- Ringkasan: Sebelumnya pesan mentah bocor ke pengguna: network putus tampil `"Failed to fetch"`, respons 500/502 non-JSON bikin `response.json()` lempar `"Unexpected token '<'"`, dan pesan teknis Laravel (`"Server Error"`, `SQLSTATE`) muncul langsung di toast. Dibuat helper terpusat `src/lib/api/errors.ts` (`friendlyApiMessage(status, raw, errors)` + `getErrorMessage(err, fallback)`): peta status HTTP ke kalimat Indonesia, deteksi pesan teknis (`looksTechnical`) lalu diganti default, 422 pakai error field pertama, 4xx pertahankan pesan backend bila manusiawi, 0/5xx selalu pakai default. `client.ts` di-harden: `fetch` dibungkus try/catch (network → `status:0`), `response.json()` dibungkus try/catch (non-JSON → null), dan `downloadFile` ikut. Call-site yang cek `instanceof Error` (ApiError itu objek biasa, cek itu selalu gagal → pesan ramah hilang) dialihkan ke `getErrorMessage`: bug-reports (2), push-toggle, survey-assignment-form, audit-logs (bonus: audit-logs sempat pakai bentuk axios `err.response.data.message` yang selalu undefined). Boundary `error.tsx`/`global-error.tsx` sudah jelas, tak diubah.
- File berubah:
  - `E-REPORT-FRONTEND/src/lib/api/errors.ts` (baru)
  - `E-REPORT-FRONTEND/src/lib/api/client.ts`
  - `E-REPORT-FRONTEND/src/app/bug-reports/bug-reports-view.tsx`
  - `E-REPORT-FRONTEND/src/components/layout/push-toggle.tsx`
  - `E-REPORT-FRONTEND/src/components/surveys/survey-assignment-form.tsx`
  - `E-REPORT-FRONTEND/src/app/audit-logs/audit-logs-view.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: `tsc --noEmit` lolos. Call-site lain (~40) masih pola `err.message || 'Gagal ...'` — sekarang aman karena `client.ts` menjamin `message` sudah ramah; bisa dimigrasi ke `getErrorMessage` bertahap kalau mau seragam. Verifikasi runtime penuh butuh memancing error (matikan backend / kirim data invalid) saat login.
- Agent: REMOX

### 2026-08-03 - Redesain kartu pengingat absensi (motif punch-clock/tiket)

- Area: frontend
- Ringkasan: Kartu pengingat absensi admin di dashboard didesain ulang biar tidak generic. Dari pola "kotak-ikon + badge + judul + tombol solid" jadi motif kartu absen/tiket: rail aksen kiri, jam berjalan WIB (mono, `HH:MM:SS`, colon berkedip, update tiap detik lewat `setInterval`) sebagai hook, garis perforasi vertikal dashed, dan stub tiket kanan dengan notch bulat + CTA "Cap Absen". Jam init `null` untuk hindari hydration mismatch, format via `Intl.DateTimeFormat` zona `Asia/Jakarta`.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/dashboard/dashboard-view.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Kartu hanya tampil untuk role non-super-admin yang belum lapor (`needsAttendance`), jadi verifikasi visual penuh butuh login admin + state absensi kosong. Hanya perubahan presentasi, tak sentuh logika `needsAttendance`/data.
- Agent: REMOX

### 2026-07-31 - Audit keamanan + pentest pra-produksi (nihil kerentanan)

- Area: security / docs
- Ringkasan: Audit keamanan menyeluruh pra-produksi. (1) Audit kode backend Laravel: SQLi (aman, Eloquent binding + intval), mass-assignment (aman, role tak fillable/F-014), auth login (timing-safe + throttle berlapis), IDOR (policy account-scoped), enumeration (colorLookup ditutup), debug endpoint (gated env+role), upload bug-report (image+dimensions+honeypot), CORS (non-wildcard/F-010), secret (bersih, .env gitignore). (2) Verifikasi produksi live `api-ereport.interiorcustom.id` non-intrusif: APP_DEBUG off (error JSON bersih tanpa stack trace), cookie secure+httponly+samesite, auth 401 JSON, security headers ada, path sensitif (.env/telescope/horizon/_ignition) semua 404, di belakang Cloudflare. (3) Pentest aktif LOKAL saja (DB e_report lokal, token via tinker lalu dicabut): 8 vektor — IDOR lintas akun (403), SQLi boolean+time-based (tak tereksekusi), mass-assignment escalate role/account (diabaikan), rate-limit login (429 di attempt ke-6), authorization surveyor/admin ke endpoint super-admin (403). Semua tertahan, nihil kerentanan. Produksi live TIDAK diuji aktif (hindari lockout admin asli/korupsi data/DoS).
- File berubah:
  - `E-REPORT-APP/12-Security/Pentest-Audit-2026-07-31.md` (baru)
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Satu item nyata tersisa di produksi = aktifkan HSTS di Cloudflare (P1). Kosmetik: header keamanan dobel (P2), X-XSS-Protection deprecated (P3). Tak ada perubahan kode dari audit ini (read-only). Realtime/WebSocket tak disentuh.
- Agent: REMOX

### 2026-07-31 - Redesain kartu absensi admin + optimasi bundle HP kelas bawah

- Area: frontend / ops / docs
- Ringkasan: (1) Redesain panel "Kehadiran Laporan Hari Ini" pada Absensi Report Harian menjadi kartu kategori berchip ikon, deskripsi per tombol, spinner per tombol saat submit, plus panel "Panduan Pengisian" di atas tombol dengan kontras surface berlapis (kartu induk `--card`, panduan `--background`, tombol `--muted`). (2) Optimasi performa untuk admin & surveyor HP RAM 2-4GB: `optimizePackageImports` (lucide-react, date-fns, recharts, framer-motion) di next.config; framer-motion dibuang total dari halaman login (turun 259KB → 215KB First Load JS), chrome (`bottom-nav`, `sidebar`, `pwa-install-button`), serta `bug-report-widget`, `surveys-view`, dan `app/loading.tsx` — diganti CSS + `tw-animate-css`. Dua animasi blur infinite pada tab aktif bottom-nav dihapus (hemat CPU/baterai). Surveyor kini 100% bebas framer-motion (sisa hanya `master-data-view` = super admin). (3) Perbaikan pasca code-review: swipe-turun untuk menutup More sheet dipulihkan via handler touch ringan, transisi box-shadow FAB dikembalikan, `overflow-hidden` mati pada alert login dihapus.
- File berubah:
  - `E-REPORT-FRONTEND/next.config.js`
  - `E-REPORT-FRONTEND/src/app/report-attendances/report-attendances-view.tsx`
  - `E-REPORT-FRONTEND/src/app/login/page.tsx`
  - `E-REPORT-FRONTEND/src/app/surveys/surveys-view.tsx`
  - `E-REPORT-FRONTEND/src/app/loading.tsx`
  - `E-REPORT-FRONTEND/src/components/layout/bottom-nav.tsx`
  - `E-REPORT-FRONTEND/src/components/layout/sidebar.tsx`
  - `E-REPORT-FRONTEND/src/components/layout/pwa-install-button.tsx`
  - `E-REPORT-FRONTEND/src/components/bug-report/bug-report-widget.tsx`
  - `E-REPORT-FRONTEND/.claude/launch.json` (baru, untuk preview dev)
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Animasi keluar (exit) modal/sheet kini instan tanpa framer (tradeoff diterima). `master-data-view.tsx` masih pakai framer-motion (super admin only). Realtime & WebSocket sengaja TIDAK disentuh. Wajib build produksi (`next build`) dengan `npm run dev` dimatikan agar `.next` tidak bentrok.
- Agent: REMOX

### 2026-07-31 - Modernisasi banner pengingat absensi dashboard

- Area: frontend / docs
- Ringkasan: Menghapus label `Live` dari header Dashboard dan merapikan banner pengingat absensi admin menjadi lebih informatif: copy dibuat lebih jelas, status `Menunggu laporan` ditampilkan sebagai chip kecil, CTA diganti menjadi `Isi Absen`, serta visual dibuat lebih modern/adaptif dengan aksen amber dan layout mobile yang tetap rapi. Garis aksen samping dan efek hover gerak pada banner dihapus agar tampil lebih tenang.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/dashboard/dashboard-view.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Perubahan hanya visual/copy. Perlu cek langsung pada light dan dark mode untuk memastikan kontras amber sesuai selera tema akun.
- Agent: Codex

### 2026-07-31 - Batasi Data Konsumen Survey untuk admin pengaju

- Area: backend / frontend / security / docs
- Ringkasan: Membuka menu dan route `/survey-consumers` untuk role admin biasa, lalu membatasi daftar/detail survey operasional di backend ke `requested_by` user login selain tetap berada pada `account_id` admin tersebut. Filter lintas akun/surveyor disembunyikan dari admin biasa agar halaman tidak memberi kesan bisa melihat jadwal admin lain.
- File berubah:
  - `E-REPORT-BACKEND_API/app/Http/Controllers/Api/SurveyController.php`
  - `E-REPORT-BACKEND_API/app/Policies/SurveyPolicy.php`
  - `E-REPORT-FRONTEND/src/components/layout/sidebar.tsx`
  - `E-REPORT-FRONTEND/src/components/layout/bottom-nav.tsx`
  - `E-REPORT-FRONTEND/src/lib/auth/roles.ts`
  - `E-REPORT-FRONTEND/src/app/survey-consumers/survey-consumers-view.tsx`
  - `E-REPORT-APP/11-Modules/Survey.md`
  - `E-REPORT-APP/12-Security/Security-and-Access.md`
  - `E-REPORT-APP/12-Security/Role-Matrix.md`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Admin biasa tidak lagi dapat membuka detail/mengubah maps/reschedule/membatalkan survey yang diajukan admin lain walaupun masih satu akun. Manager Surveyor tetap melihat semua.
- Agent: Codex

### 2026-07-31 - Redesain kartu Pipeline Konversi analytics

- Area: frontend / docs
- Ringkasan: Merapikan kartu `Pipeline Konversi` di halaman Analytics dari bar panjang bertumpuk menjadi stage cards ringkas untuk Leads, Survey, dan Deal. Menambahkan chip rasio transisi antar tahap dan panel rasio keseluruhan agar angka utama lebih mudah dipindai tanpa ruang kosong visual yang berlebihan.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/analytics/analytics-view.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Perubahan hanya visual dan memakai data funnel yang sama. Perlu cek langsung pada desktop dan mobile untuk memastikan lebar stage cards nyaman.
- Agent: Codex

### 2026-07-31 - Ringkas panel percakapan dan pengingat konsultasi

- Area: frontend / docs
- Ringkasan: Merapikan layout detail konsultasi menjadi dua kolom independen: kolom kiri berisi `Informasi Klien`, `Detail & Kebutuhan Produk`, lalu `Percakapan & Catatan`; kolom kanan berisi `Status Survey` lalu `Pengingat`. Struktur ini menghilangkan ruang kosong besar di atas chat pada desktop. Panel chat juga tidak memakan ruang kosong terlalu tinggi saat belum ada pesan, tinggi riwayat chat dibatasi responsif, input chat diperkecil, dan reminder dibuat sebagai modul ringkas dengan badge jumlah aktif, form cepat, serta daftar yang mudah dipindai.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/consultations/[id]/page.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Perubahan hanya visual dan tidak mengubah API. Perlu cek langsung pada viewport mobile dan desktop untuk memastikan tinggi sidebar terasa pas dengan data real.
- Agent: Codex

### 2026-07-30 - Tambahkan pengelolaan pesan percakapan konsultasi

- Area: backend / frontend / security / docs
- Ringkasan: Menambahkan menu titik tiga pada Percakapan & Catatan untuk memilih pesan, mengedit satu pesan milik sendiri, menghapus beberapa pesan, dan membersihkan percakapan. Admin hanya dapat mengelola pesan miliknya dalam konsultasi akun sendiri; super admin dapat menghapus pesan siapa pun dan membersihkan seluruh chat, tetapi tidak dapat mengubah isi pesan pengguna lain. Edit dan penghapusan disinkronkan realtime ke peserta percakapan. Akses relasi konsultasi pada observer, policy, dan aksi tandai-dibaca dibuat eksplisit agar kompatibel dengan pencegahan lazy loading Laravel.
- File berubah:
  - `E-REPORT-BACKEND_API/app/Events/ConsultationNoteCreated.php`
  - `E-REPORT-BACKEND_API/app/Events/ConsultationNotesChanged.php`
  - `E-REPORT-BACKEND_API/app/Http/Controllers/Api/ConsultationNoteController.php`
  - `E-REPORT-BACKEND_API/app/Http/Controllers/Api/NotificationController.php`
  - `E-REPORT-BACKEND_API/app/Policies/ConsultationNotePolicy.php`
  - `E-REPORT-BACKEND_API/app/Providers/AppServiceProvider.php`
  - `E-REPORT-BACKEND_API/routes/api.php`
  - `E-REPORT-FRONTEND/src/app/consultations/[id]/page.tsx`
  - `E-REPORT-FRONTEND/src/components/survey-realtime-listener.tsx`
  - `E-REPORT-FRONTEND/src/lib/api/client.ts`
  - `E-REPORT-FRONTEND/src/lib/hooks/useConsultations.ts`
  - `E-REPORT-APP/11-Modules/Consultations.md`
  - `E-REPORT-APP/11-Modules/Notifications.md`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Penghapusan bersifat permanen. UI meminta konfirmasi sebelum hapus pilihan atau bersihkan percakapan.
- Agent: Codex

### 2026-07-30 - Realtime dan push catatan konsultasi

- Area: backend / frontend / security / docs
- Ringkasan: Menambahkan event realtime catatan pada private channel per pengguna, toast catatan masuk, invalidasi badge/detail konsultasi, dan Web Push untuk PWA tertutup. Penerima realtime dan push disatukan: super admin serta admin akun konsultasi yang sama selain pengirim. Query notifikasi catatan dikunci agar surveyor, manager surveyor, dan admin akun lain tidak mendapat jumlah maupun isi catatan. Langganan push dicabut saat logout dan URL klik push dibatasi ke origin aplikasi.
- File berubah:
  - `E-REPORT-BACKEND_API/app/Events/ConsultationNoteCreated.php`
  - `E-REPORT-BACKEND_API/app/Http/Controllers/Api/ConsultationNoteController.php`
  - `E-REPORT-BACKEND_API/app/Http/Controllers/Api/NotificationController.php`
  - `E-REPORT-BACKEND_API/app/Services/NotificationSummaryService.php`
  - `E-REPORT-BACKEND_API/routes/channels.php`
  - `E-REPORT-FRONTEND/src/components/survey-realtime-listener.tsx`
  - `E-REPORT-FRONTEND/src/lib/hooks/useAuth.ts`
  - `E-REPORT-FRONTEND/worker/index.js`
  - `E-REPORT-APP/11-Modules/Notifications.md`
  - `E-REPORT-APP/11-Modules/Consultations.md`
  - `E-REPORT-APP/12-Security/Role-Matrix.md`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Web Push memerlukan HTTPS/installed PWA, izin browser, dan subscription aktif dari tombol `Aktifkan HP`. Reverb tetap perlu proses server aktif; bila mati, catatan tetap tersimpan dan Web Push tetap dicoba.
- Agent: Codex

### 2026-07-30 - Pulihkan nama surveyor saja pada export rekap

- Area: backend / docs
- Ringkasan: Mengembalikan isi sel jadwal pada export Excel Rekap Jadwal Surveyor menjadi nama surveyor saja seperti format awal. Detail konsumen tetap tersedia pada halaman rekap, tetapi tidak dimasukkan ke grid Excel. Lebar kolom hari juga dikembalikan ke ukuran awal.
- File berubah:
  - `E-REPORT-BACKEND_API/app/Services/Reports/SurveyorScheduleRecapExcelExporter.php`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Export memakai `scheduleItems.surveyorName` dan tetap mendukung `surveyorNames` sebagai fallback untuk kontrak lama.
- Agent: Codex

### 2026-07-30 - Tambahkan ID konsumen dan ringkas UI rekap surveyor

- Area: backend / frontend / docs
- Ringkasan: Menambahkan `ID Konsumen` pada setiap item Rekap Jadwal Surveyor dan label export Excel. Tampilan desktop diringkas menjadi daftar terstruktur tanpa badge metadata berlebihan; setiap kolom hari diberi jarak, garis aksen atas, dan selang warna baris agar lebih mudah dipindai tanpa ring. Mobile memakai pemilih hari Senin-Minggu yang selalu muat satu baris tanpa scroll horizontal, sehingga hanya jadwal hari terpilih yang ditampilkan.
- File berubah:
  - `E-REPORT-BACKEND_API/app/Services/Reports/SurveyorScheduleRecapService.php`
  - `E-REPORT-FRONTEND/src/types/index.ts`
  - `E-REPORT-FRONTEND/src/app/rekap-jadwal-surveyor/rekap-jadwal-surveyor-view.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Kontrak lama `surveyorNames` tetap dipertahankan untuk kompatibilitas; konsumen tanpa ID ditampilkan sebagai `-`.
- Agent: Codex

### 2026-07-30 - Lengkapi label rekap jadwal surveyor

- Area: backend / frontend / docs
- Ringkasan: Mengubah item harian pada Rekap Jadwal Surveyor menjadi format `NAMA SURVEYOR ((GRUP)-NAMA KONSUMEN-KOTA/KAB)` agar jadwal mingguan menampilkan konteks akun, konsumen, dan wilayah. Tampilan tabel dibuat wrap-friendly dan kolom export Excel rekap dilebarkan.
- File berubah:
  - `E-REPORT-BACKEND_API/app/Services/Reports/SurveyorScheduleRecapService.php`
  - `E-REPORT-BACKEND_API/app/Services/Reports/SurveyorScheduleRecapExcelExporter.php`
  - `E-REPORT-FRONTEND/src/app/rekap-jadwal-surveyor/rekap-jadwal-surveyor-view.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Ringkasan jumlah per surveyor tetap memakai nama surveyor murni; hanya daftar jadwal per hari yang memakai label lengkap.
- Agent: Codex

### 2026-07-30 - Buat label item rekap lebih mudah dibaca

- Area: backend / frontend / docs
- Ringkasan: Mengubah item harian rekap jadwal dari teks panjang berkoma/kurung menjadi struktur berlabel `Surveyor`, `Grup`, `Konsumen`, `Kota/Kab`, dan `Jam`. Tampilan desktop diubah dari tabel besar menjadi papan 7 kolom per hari dengan card jadwal ringkas: jam, surveyor, konsumen, grup, dan kota. Export tetap menerima `displayLabel` berisi label lengkap.
- File berubah:
  - `E-REPORT-BACKEND_API/app/Services/Reports/SurveyorScheduleRecapService.php`
  - `E-REPORT-FRONTEND/src/types/index.ts`
  - `E-REPORT-FRONTEND/src/app/rekap-jadwal-surveyor/rekap-jadwal-surveyor-view.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Kontrak lama `surveyorNames` tetap dipertahankan sebagai fallback dan untuk export.
- Agent: Codex

### 2026-07-30 - Pulihkan format export leads Excel

- Area: backend / docs
- Ringkasan: Mengembalikan sheet `Data Leads` pada export leads Excel agar sama dengan template import leads 15 kolom. Format tracking survey 22 kolom dari perubahan sebelumnya dihapus dari jalur export leads.
- File berubah:
  - `E-REPORT-BACKEND_API/app/Services/Reports/LeadsExcelExporter.php`
  - `E-REPORT-BACKEND_API/app/Services/Reports/LeadsReportService.php`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Export kembali fokus pada data import/export lead dasar; informasi tracking survey tidak lagi muncul di sheet `Data Leads`.
- Agent: Codex
    
### 2026-07-29 - Restrukturisasi Data Analytics Ekspor (DATABASE v2.7) dan Ranking Surveyor Rill

- Area: database / backend / docs
- Ringkasan: Menganalisis skema data leads (export Analytics) yang sebelumnya tidak memuat detail pelacakan respon, detail surveyor, dan konversi. Ditemukan kesalahan logika di mana "Klasemen Surveyor" digaris bawahi berdasarkan CS/Admin yang menginput data, bukan surveyor lapangan. Menambahkan 2 tabel baru (`data_survey` & opsional `lead_status_history` jika terpisah/extend) atau memodifikasi query Excel dari tabel yang sudah ada yaitu `surveys` dan `consultation_status_histories`. Membuat file rancangan MASTER PLAN untuk 21 kolom eksport baru.
- File berubah:
  - `E-REPORT-BACKEND_API/app/Services/Reports/LeadsReportService.php` (menambahkan logika 21 kolom eksport dengan meload relasi history dan survey)
  - `E-REPORT-BACKEND_API/app/Services/Reports/LeadsExcelExporter.php` (mendeklarasikan Header, Lebar Kolom 21 unit, Validasi export, Cell mapping)
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
  - Plus pembuatan: `MASTER_PLAN_EREPORT_V2.7_TRACKING.md`, script SQL Migration `migration_add_surveys_table.sql`, dll.
- Risiko / follow-up: Data Surveyor sekarang MURNI diambil berdasarkan penugasan survey (`surveys.surveyor_id`) dan performa Deal Rate dikalkulasi sesuai ID Status Deal.
- Agent: REMOX

### 2026-07-29 - Samakan hitungan badge dan panel notifikasi

- Area: backend / frontend
- Ringkasan: Memperbaiki bug angka notifikasi survey yang turun saat dropdown dibuka. Endpoint summary sebelumnya menghitung unread survey dari 30 item terbaru yang ditampilkan, sehingga unread lama tidak ikut terhitung. Summary kini menghitung seluruh survey notification unread, dan frontend memakai field `unread_surveys` dari response agar badge header dan panel tetap konsisten.
- File berubah:
  - `E-REPORT-BACKEND_API/app/Http/Controllers/Api/NotificationController.php`
  - `E-REPORT-FRONTEND/src/components/layout/notification-center.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: List panel tetap dibatasi 30 item terbaru untuk performa; angka unread menampilkan total seluruh item belum dibaca.
- Agent: Codex

### 2026-07-29 - Perbaiki realtime notifikasi survey lintas role

- Area: backend / frontend / docs
- Ringkasan: Menambahkan channel privat `survey.account.{account_id}` agar admin menerima pembaruan realtime untuk survey milik akunnya. Event survey kini broadcast juga ke channel akun, listener frontend admin ikut subscribe, dan event realtime memaksa refetch query notifikasi aktif. Fallback polling notifikasi dibuat lebih responsif: badge 10 detik saat aktif dan panel detail 5 detik saat dropdown terbuka.
- File berubah:
  - `E-REPORT-BACKEND_API/routes/channels.php`
  - `E-REPORT-BACKEND_API/app/Events/SurveyRealtimeUpdated.php`
  - `E-REPORT-FRONTEND/src/components/survey-realtime-listener.tsx`
  - `E-REPORT-FRONTEND/src/lib/hooks/useNotifications.ts`
  - `E-REPORT-APP/11-Modules/Survey.md`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Realtime penuh tetap membutuhkan `php artisan reverb:start` berjalan dan env `NEXT_PUBLIC_REVERB_*` sesuai. Jika WebSocket mati, fallback polling tetap memperbarui notifikasi dengan jeda singkat.
- Agent: Codex

### 2026-07-29 - Selaraskan modal survey dengan theme

- Area: frontend
- Ringkasan: Menyesuaikan modal reschedule admin dan modal pembatalan survey agar memakai token theme yang sama dengan sistem. Surface, border, icon badge, input tanggal, textarea, dan tombol footer kini adaptif di light/dark mode serta tidak lagi hardcoded terlalu gelap atau terlalu putih.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/surveys/surveys-view.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Perubahan hanya visual; flow reschedule dan cancel tidak berubah.
- Agent: Codex

### 2026-07-29 - Perluas pembatalan survey lintas role

- Area: backend / frontend / docs
- Ringkasan: Mengubah aturan pembatalan survey agar admin akun terkait, manager surveyor, dan surveyor yang ditugaskan dapat membatalkan survey selama belum selesai. Notifikasi cancel kini juga dikirim ke Manager Surveyor/Super Admin selain pengaju dan surveyor. Halaman `/surveys` menampilkan tab `Dibatalkan`, tombol `Batal`, dan dialog alasan pembatalan agar histori tetap jelas.
- File berubah:
  - `E-REPORT-BACKEND_API/app/Policies/SurveyPolicy.php`
  - `E-REPORT-BACKEND_API/app/Events/SurveyRealtimeUpdated.php`
  - `E-REPORT-FRONTEND/src/app/surveys/surveys-view.tsx`
  - `E-REPORT-FRONTEND/src/lib/hooks/useSurveys.ts`
  - `E-REPORT-APP/11-Modules/Survey.md`
  - `E-REPORT-APP/12-Security/Role-Matrix.md`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Survey yang sudah `completed` tetap tidak bisa dibatalkan; pembatalan berulang tetap ditolak backend.
- Agent: Codex

### 2026-07-29 - Bedakan warna jenis notifikasi survey

- Area: frontend
- Ringkasan: Menyesuaikan palet notifikasi survey agar tiap event lebih mudah dibedakan sekilas. `Permintaan baru`, `Terjadwal`, `Jadwal berubah` dari admin/manager, `Berlangsung`, `Selesai`, `Dibatalkan`, `Maps berubah`, dan `Surveyor diganti` kini memakai tone warna berbeda yang tetap adaptif pada light/dark mode.
- File berubah:
  - `E-REPORT-FRONTEND/src/components/layout/notification-center.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Perubahan hanya visual pada daftar notifikasi.
- Agent: Codex

### 2026-07-29 - Tambahkan edit jadwal awal admin di status survey

- Area: frontend / docs
- Ringkasan: Memisahkan tampilan `Jadwal Awal Admin` dari `Jadwal Ditetapkan` pada kartu Status Survey detail konsultasi. Admin kini bisa mengedit jadwal awal langsung dari kartu; perubahan memakai endpoint reschedule yang sudah ada sehingga riwayat tercatat dan Manager Surveyor menerima notifikasi reschedule. Tombol Maps juga dirapikan menjadi action button dengan ikon di samping label.
- File berubah:
  - `E-REPORT-FRONTEND/src/components/consultations/survey-request-card.tsx`
  - `E-REPORT-FRONTEND/src/lib/hooks/useSurveys.ts`
  - `E-REPORT-APP/11-Modules/Survey.md`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Edit jadwal awal tetap mengikuti guard backend; hanya berlaku untuk survey `requested`/`scheduled` dan tanggal tidak boleh lampau.
- Agent: Codex

### 2026-07-29 - Redesain panel status survey kosong

- Area: frontend
- Ringkasan: Merapikan panel `Status Survey` saat lead sudah berada di tahap survey tetapi belum diajukan ke Manager Surveyor. Panel kini memakai highlight status kecil, copy yang lebih jelas, tombol utama lebih tegas, responsive, dan tetap adaptif pada light/dark mode. Footer tombol modal pengajuan survey juga distandarkan agar ukuran, radius, spacing, dan warna primary/secondary lebih modern.
- File berubah:
  - `E-REPORT-FRONTEND/src/components/consultations/survey-request-card.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Perubahan hanya visual; tidak mengubah payload atau alur pengajuan survey.
- Agent: Codex

### 2026-07-29 - Adaptifkan modal pengajuan survey di light mode

- Area: frontend
- Ringkasan: Mengubah modal `Ajukan Survey ke Manager Surveyor` dan editor Link Maps di kartu Status Survey agar memakai token warna adaptif. Panel, highlight konsumen, tanggal, calendar popover, input Maps, textarea, tombol batal, dan empty state kini terbaca di light mode maupun dark mode.
- File berubah:
  - `E-REPORT-FRONTEND/src/components/consultations/survey-request-card.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Tidak ada perubahan flow atau payload API; hanya penyesuaian visual dan keterbacaan.
- Agent: Codex

### 2026-07-29 - Samakan bubble tab survey dengan bottom nav

- Area: frontend
- Ringkasan: Menyamakan active bubble pada tab status survey dengan bahasa visual bottom navigation. Tab aktif kini memakai campuran `--card`/`--background`, border adaptif, glow lebih tipis, dan teks tanpa shadow berlebih agar light mode tidak terasa gelap atau berbeda material.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/surveys/surveys-view.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Perubahan hanya visual pada tab status survey.
- Agent: Codex

### 2026-07-29 - Terangkan bubble aktif bottom nav di light mode

- Area: frontend
- Ringkasan: Menyesuaikan active bubble pada bottom navigation agar di light mode tidak terlihat terlalu gelap. Surface aktif kini memakai campuran `--card` dan `--background`, border lebih lembut, glow ikon dikurangi, sementara dark mode tetap mempertahankan nuansa liquid glass. Icon di sheet `Menu Lainnya` juga dibuat adaptif agar tidak hilang pada light mode.
- File berubah:
  - `E-REPORT-FRONTEND/src/components/layout/bottom-nav.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Perubahan hanya visual pada indikator tab aktif mobile.
- Agent: Codex

### 2026-07-29 - Tambahkan data dummy operasional lokal

- Area: backend / database / ops
- Ringkasan: Menambahkan `DummyOperationalDataSeeder` untuk mengisi ulang data dummy konsultasi dan jadwal survey lokal. Seeder membuat 360 konsultasi, 260 survey, 1008 pivot kategori kebutuhan, 600 histori survey, 600 aktivitas survey, dan 260 notifikasi dengan variasi state `requested`, `scheduled`, `in_progress`, dan `completed`.
- File berubah:
  - `E-REPORT-BACKEND_API/database/seeders/DummyOperationalDataSeeder.php`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Seeder ini mengosongkan data operasional lokal sebelum mengisi dummy baru, tetapi tidak menghapus akun, user, account/cabang, kategori kebutuhan, status pipeline, atau master status survey.
- Agent: Codex

### 2026-07-29 - Bersihkan data operasional lokal

- Area: database / ops
- Ringkasan: Mengosongkan data transaksi/operasional lokal untuk konsultasi, survey, histori survey, notifikasi survey, reminder, import, sequence ID konsultasi, report attendance, audit log, bug report, dan login attempts. Data akun, user, account/cabang, kategori kebutuhan, status pipeline, dan master status survey tetap dipertahankan.
- File berubah:
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Data operasional lokal sudah dihapus permanen. Browser mungkin perlu refresh halaman agar cache client ikut kosong.
- Agent: Codex

### 2026-07-29 - Adaptifkan modal jadwal survey untuk light dan dark mode

- Area: frontend
- Ringkasan: Mengubah modal `Jadwalkan Survey`/`Reschedule Survey`, dropdown jam, dan dropdown select agar memakai token warna adaptif. Panel, field, placeholder, popup kalender/jam, tombol batal, tombol simpan, dan tombol close kini tetap terbaca di light mode maupun dark mode.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/surveys/surveys-view.tsx`
  - `E-REPORT-FRONTEND/src/components/ui/time-search-select.tsx`
  - `E-REPORT-FRONTEND/src/components/ui/select.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Perubahan border select bersifat reusable dan dapat membuat dropdown di halaman lain terlihat sedikit lebih ringan pada light mode.
- Agent: Codex

### 2026-07-28 - Seragamkan kredensial akun surveyor

- Area: backend / database / docs
- Ringkasan: Mengubah email seluruh akun surveyor operasional ke pola `{nama}-srvy@npp.com`, dengan spasi nama dinormalisasi menjadi titik. Seeder kini turut mengelola akun BAGJA, memigrasikan email lama tanpa membuat duplikat, dan mereset password seluruh akun target ke nilai `SEED_SURVEYOR_PASSWORD` saat dijalankan.
- File berubah:
  - `E-REPORT-BACKEND_API/database/seeders/SurveyorUserSeeder.php`
  - `E-REPORT-APP/11-Modules/Auth-and-Users.md`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Password default bersama perlu diganti bila kredensial sudah dibagikan ke pihak yang tidak berwenang.
- Agent: Codex

### 2026-07-28 - Padatkan kartu survey menjadi empat kolom desktop

- Area: frontend / docs
- Ringkasan: Mengaktifkan empat kolom kartu survey mulai breakpoint desktop `xl` dan menambahkan ukuran compact khusus layar lebar. Padding, tinggi blok jadwal, tombol, serta jarak antarbagian dipadatkan tanpa mengurangi ukuran kontrol pada mobile dan tablet.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/surveys/surveys-view.tsx`
  - `E-REPORT-APP/11-Modules/Survey.md`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Teks panjang tetap dibatasi dan seluruh informasi tersedia melalui `Lihat Detail`.
- Agent: Codex

### 2026-07-28 - Tambahkan tombol Google Maps pada kartu survey

- Area: frontend / docs
- Ringkasan: Menambahkan tombol `Buka Google Maps` tepat di bawah tombol WhatsApp pada kartu survey. Tombol membuka `google_maps_url` di tab baru; bila link belum tersedia, kartu menampilkan state nonaktif yang jelas agar susunan kartu tetap konsisten.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/surveys/surveys-view.tsx`
  - `E-REPORT-APP/11-Modules/Survey.md`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Tidak ada perubahan data atau API.
- Agent: Codex

### 2026-07-28 - Redesain kartu jadwal survey summary-first

- Area: frontend / backend / docs
- Ringkasan: Meredesain daftar survey agar jadwal, nama konsumen, WhatsApp, dan aksi utama menjadi fokus. Alamat serta kebutuhan diringkas, detail lengkap dibuka melalui dialog responsif, grid desktop dibatasi tiga kolom, dan pencarian nama/WA/ID ditambahkan. Urutan endpoint kini mengikuti waktu operasional setiap state dengan opsi urutan update terbaru.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/surveys/surveys-view.tsx`
  - `E-REPORT-FRONTEND/src/types/index.ts`
  - `E-REPORT-BACKEND_API/app/Http/Controllers/Api/SurveyController.php`
  - `E-REPORT-APP/06-API/API-Catalog.md`
  - `E-REPORT-APP/11-Modules/Survey.md`
  - `E-REPORT-APP/09-Decisions/Decision-Log.md`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Perubahan urutan dapat membuat posisi kartu berbeda dari urutan update lama; pengguna tetap dapat memilih `Terbaru diperbarui`.
- Agent: Codex

### 2026-07-28 - Perjelas tombol isi Link Maps

- Area: frontend
- Ringkasan: Merapikan area Link Google Maps di kartu Status Survey agar lebih user friendly. Tombol kecil `Isi` diganti menjadi tombol utama `Isi Link Maps`/`Ubah Link`, empty state diberi pesan yang jelas, dan input/simpan dibuat lebih mudah terlihat.
- File berubah:
  - `E-REPORT-FRONTEND/src/components/consultations/survey-request-card.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Tidak ada perubahan data; hanya peningkatan keterbacaan dan affordance UI.
- Agent: Codex

### 2026-07-28 - Pindahkan Status Survey ke kolom aktivitas

- Area: frontend
- Ringkasan: Memindahkan kartu Status Survey pada detail konsultasi dari kolom informasi utama ke kolom kanan, tepat di atas Catatan Aktivitas, agar input Google Maps dan status survey berada dekat area aktivitas.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/consultations/[id]/page.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Pada layar kecil urutan tetap mengikuti aliran halaman, dengan Status Survey muncul setelah informasi dan detail kebutuhan.
- Agent: Codex

### 2026-07-28 - Update link Maps pada survey aktif

- Area: backend / frontend / docs
- Ringkasan: Menambahkan endpoint `PATCH /surveys/{survey}/maps` agar admin akun terkait dapat melengkapi atau memperbaiki link Google Maps pada request survey yang sudah telanjur diajukan. Kartu Status Survey di detail konsultasi kini punya input `Link Google Maps` untuk survey aktif; setelah tersimpan query survey, konsultasi, dashboard, dan notifikasi di-refresh.
- File berubah:
  - `E-REPORT-BACKEND_API/routes/api.php`
  - `E-REPORT-BACKEND_API/app/Http/Controllers/Api/SurveyController.php`
  - `E-REPORT-BACKEND_API/app/Policies/SurveyPolicy.php`
  - `E-REPORT-BACKEND_API/app/Events/SurveyRealtimeUpdated.php`
  - `E-REPORT-FRONTEND/src/lib/hooks/useSurveys.ts`
  - `E-REPORT-FRONTEND/src/components/consultations/survey-request-card.tsx`
  - `E-REPORT-APP/06-API/API-Catalog.md`
  - `E-REPORT-APP/11-Modules/Survey.md`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Realtime lintas user tetap bergantung service Reverb/Echo aktif; tanpa itu data tetap tersimpan dan terlihat setelah refetch/refresh.
- Agent: Codex

### 2026-07-28 - Tampilkan strip admin kosong di kartu survey

- Area: frontend
- Ringkasan: Menyesuaikan format nama akun pada kartu survey agar admin kosong tetap tampil sebagai strip, contoh `AKBAR INTERIOR (-)`.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/surveys/surveys-view.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Tidak ada perubahan data, hanya format tampilan.
- Agent: Codex

### 2026-07-28 - Ringkas data kosong kartu survey

- Area: frontend
- Ringkasan: Mengubah fallback data kosong pada kartu survey menjadi strip (`-`) untuk akun/admin, ID konsumen, nama konsumen, nomor WA, alamat, kebutuhan, dan jadwal awal survey.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/surveys/surveys-view.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Tampilan kartu menjadi lebih ringkas; detail kosong tidak lagi memakai kalimat panjang.
- Agent: Codex

### 2026-07-28 - Susun ulang info kartu survey

- Area: frontend
- Ringkasan: Mengubah urutan informasi pada kartu survey menjadi nama akun beserta admin, ID konsumen, nama konsumen, nomor WA, alamat, kebutuhan, jadwal awal survey, dan link Maps bila tersedia. Baris status dikeluarkan dari blok informasi utama agar detail konsumen lebih fokus.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/surveys/surveys-view.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Link Maps hanya tampil ketika data `google_maps_url` tersedia.
- Agent: Codex

### 2026-07-28 - Perbaiki koordinat visual marker peta

- Area: frontend
- Ringkasan: Menghapus override `position: relative` pada elemen root marker yang menimpa positioning absolut milik MapLibre dan membuat pin berderet ke laut/Australia. Ukuran marker disatukan ke helper compact dan koordinat marker divalidasi terhadap batas Indonesia sebelum dirender.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/geo-analytics/_components/geo-map.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Marker wilayah yang pusat polygonnya valid tetap tampil tepat di wilayahnya; data geometri atau ID wilayah yang tidak cocok tidak akan menghasilkan pin liar.
- Agent: Codex

### 2026-07-28 - Pulihkan posisi pin marker peta

- Area: frontend
- Ringkasan: Menghapus offset anti-overlap marker Analisis Wilayah yang membuat pin terlihat bergeser ke laut, lalu mengecilkan ukuran marker agar tetap compact tanpa mengorbankan posisi wilayah.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/geo-analytics/_components/geo-map.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Wilayah yang sangat berdekatan masih bisa sedikit tumpang tindih; solusi final yang lebih akurat adalah clustering marker, bukan menggeser pin dari titik wilayah.
- Agent: Codex

### 2026-07-28 - Perbaiki pin marker Analisis Wilayah

- Area: frontend
- Ringkasan: Menambahkan anti-overlap pada pin marker peta Analisis Wilayah agar marker yang berdekatan tidak saling menimpa, menstabilkan anchor/tooltip pin, dan mereset warna polygon saat filter berubah supaya peta tidak menyisakan data lama.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/geo-analytics/_components/geo-map.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Marker yang sangat berdekatan akan sedikit bergeser secara visual dari pusat wilayah agar angkanya tetap terbaca.
- Agent: Codex

### 2026-07-28 - Paksa filename export rekap dari frontend

- Area: backend / frontend
- Ringkasan: Menambahkan nama file eksplisit saat tombol Excel rekap jadwal mengunduh file agar browser tidak fallback ke `export_timestamp.xlsx`, serta menyamakan kode minggu memakai pekan kalender Senin-Minggu.
- File berubah:
  - `E-REPORT-BACKEND_API/app/Http/Controllers/ExportController.php`
  - `E-REPORT-FRONTEND/src/lib/hooks/useFileDownload.ts`
  - `E-REPORT-FRONTEND/src/app/rekap-jadwal-surveyor/rekap-jadwal-surveyor-view.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Filename client dan server sengaja memakai pola sama agar tetap benar walau header Content-Disposition tidak dipakai browser.
- Agent: Codex

### 2026-07-28 - Ringkas nama file export rekap jadwal

- Area: backend
- Ringkasan: Mengubah nama file Excel rekap jadwal surveyor menjadi format singkat informatif, contoh `REKAP_JADWAL_27JUL-02AGU2026_M5.xlsx`.
- File berubah:
  - `E-REPORT-BACKEND_API/app/Http/Controllers/ExportController.php`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: `M1` sampai `M5` dihitung dari tanggal awal periode minggu.
- Agent: Codex

### 2026-07-28 - Jadikan nomor darurat collapsible

- Area: frontend
- Ringkasan: Menyembunyikan input nomor telepon darurat secara default, menambahkan tombol `+` di samping input WhatsApp untuk membukanya, dan tombol tutup pada input darurat untuk menyembunyikan sekaligus mengosongkan nilainya.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/consultations/create/page.tsx`
  - `E-REPORT-FRONTEND/src/app/consultations/[id]/edit/page.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Pada form edit, nomor darurat yang sudah tersimpan tetap otomatis tampil agar tidak terhapus tanpa terlihat.
- Agent: Codex

### 2026-07-28 - Tambahkan nomor telepon darurat lead

- Area: backend / frontend / database / docs
- Ringkasan: Menambahkan field `emergency_phone` opsional pada data konsultasi, validasi nomor darurat bila diisi, input pada form tambah/edit lead, tampilan pada detail konsumen dan detail Data Konsumen Survey, serta pencarian backend berdasarkan nomor darurat.
- File berubah:
  - `E-REPORT-BACKEND_API/database/migrations/2026_07_28_000001_add_emergency_phone_to_consultations_table.php`
  - `E-REPORT-BACKEND_API/app/Models/Consultation.php`
  - `E-REPORT-BACKEND_API/app/Http/Requests/ConsultationRequest.php`
  - `E-REPORT-BACKEND_API/app/Http/Controllers/Api/ConsultationController.php`
  - `E-REPORT-BACKEND_API/app/Http/Controllers/Api/SurveyController.php`
  - `E-REPORT-FRONTEND/src/app/consultations/create/page.tsx`
  - `E-REPORT-FRONTEND/src/app/consultations/[id]/edit/page.tsx`
  - `E-REPORT-FRONTEND/src/app/consultations/[id]/page.tsx`
  - `E-REPORT-FRONTEND/src/app/survey-consumers/survey-consumers-view.tsx`
  - `E-REPORT-FRONTEND/src/types/index.ts`
  - `E-REPORT-APP/05-Database/Data-Model.md`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Migration lokal sudah dijalankan; server/deployment tetap perlu menjalankan migration agar kolom baru tersedia di database production.
- Agent: Codex

### 2026-07-28 - Pakai kategori kebutuhan pada data konsumen survey

- Area: frontend
- Ringkasan: Mengganti sumber kolom Item/Kebutuhan di Data Konsumen Survey dari detail produk bebas menjadi kategori kebutuhan yang dipilih, sehingga teks seperti `dan minibar` tidak muncul sebagai kategori.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/survey-consumers/survey-consumers-view.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Bila lead belum punya kategori kebutuhan, kolom menampilkan `-`.
- Agent: Codex

### 2026-07-28 - Solidkan kartu mobile data konsumen survey

- Area: frontend
- Ringkasan: Mengganti background kartu mobile Data Konsumen Survey dari transparan menjadi warna solid agar grid halaman tidak menembus kartu dan informasi lebih mudah dibaca.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/survey-consumers/survey-consumers-view.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Warna dark mode memakai surface solid khusus agar tetap selaras dengan tema halaman.
- Agent: Codex

### 2026-07-28 - Buka data konsumen survey untuk surveyor

- Area: backend / frontend
- Ringkasan: Menambahkan akses halaman Data Konsumen Survey untuk role surveyor, mengganti bottom nav surveyor menjadi Survey, Data, dan Pengaturan tanpa tombol Lainnya, serta membatasi data surveyor ke survey miliknya pada akun yang sama.
- File berubah:
  - `E-REPORT-BACKEND_API/app/Http/Controllers/Api/SurveyController.php`
  - `E-REPORT-FRONTEND/src/app/survey-consumers/survey-consumers-view.tsx`
  - `E-REPORT-FRONTEND/src/components/layout/bottom-nav.tsx`
  - `E-REPORT-FRONTEND/src/components/layout/sidebar.tsx`
  - `E-REPORT-FRONTEND/src/lib/auth/roles.ts`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Surveyor tanpa `account_id` tetap dibatasi ke `surveyor_id`; surveyor berakun hanya melihat data pada akun tersebut.
- Agent: Codex

### 2026-07-28 - Perbaiki arah layout tooltip status

- Area: frontend
- Ringkasan: Memaksa tooltip status konsultasi desktop memakai layout vertikal agar label status dan detail jadwal tidak saling menekan, serta mencegah tanggal terpecah menjadi potongan angka.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/consultations/consultations-view.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Tooltip dibuat sedikit lebih lebar agar detail waktu tetap terbaca utuh.
- Agent: Codex

### 2026-07-28 - Lengkapi progres survey di tooltip dan detail konsumen

- Area: frontend
- Ringkasan: Memperbaiki tooltip status konsultasi agar tanggal tidak terpotong dan menambahkan informasi progres survey penting pada detail konsumen, termasuk status lengkap, surveyor, jadwal, waktu mulai, waktu selesai, hasil survey, catatan, dan tautan Maps bila tersedia.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/consultations/consultations-view.tsx`
  - `E-REPORT-FRONTEND/src/components/consultations/survey-request-card.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Field yang belum diisi tetap tampil sebagai `-` agar jelas bahwa datanya kosong.
- Agent: Codex

### 2026-07-28 - Ringkas tooltip progres status konsultasi

- Area: frontend
- Ringkasan: Merapikan tooltip progres status tabel konsultasi desktop menjadi panel compact satu kolom dengan header status dan baris waktu ringkas agar tidak menutupi tabel dan lebih mudah dibaca.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/consultations/consultations-view.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Tooltip tetap bergantung pada data active_survey; baris waktu kosong tidak ditampilkan.
- Agent: Codex

### 2026-07-28 - Tooltip progres status konsultasi

- Area: frontend
- Ringkasan: Menambahkan indikator info pada badge status tabel konsultasi desktop. Hover/focus badge kini menampilkan tooltip progres berisi status lengkap, surveyor, hasil survey, jadwal, waktu mulai, dan waktu selesai bila tersedia.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/consultations/consultations-view.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Tooltip memakai data active_survey yang tersedia dari API; field waktu yang kosong tidak ditampilkan.
- Agent: Codex

### 2026-07-28 - Tambahkan hasil survey pada badge status selesai

- Area: backend / frontend
- Ringkasan: Memuat relasi hasil survey pada activeSurvey di daftar konsultasi dan menampilkan badge status selesai dengan format “Selesai Survey - (Surveyor) - Hasil Survey”.
- File berubah:
  - `E-REPORT-BACKEND_API/app/Http/Controllers/Api/ConsultationController.php`
  - `E-REPORT-FRONTEND/src/app/consultations/consultations-view.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Jika hasil survey belum tersedia, badge hanya menampilkan status dan nama surveyor.
- Agent: Codex

### 2026-07-28 - Refresh konsultasi saat status survey berubah

- Area: frontend
- Ringkasan: Menambahkan invalidasi query Konsultasi pada listener realtime survey dan aksi start/submit/cancel survey agar status pipeline seperti “Selesai Survey” langsung muncul tanpa refresh manual.
- File berubah:
  - `E-REPORT-FRONTEND/src/components/survey-realtime-listener.tsx`
  - `E-REPORT-FRONTEND/src/lib/hooks/useSurveys.ts`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Event survey akan memicu refetch daftar konsultasi tambahan, diperlukan karena backend menyinkronkan status pipeline lead saat state survey berubah.
- Agent: Codex

### 2026-07-28 - Tampilkan surveyor pada status survey konsultasi

- Area: backend / frontend
- Ringkasan: Memuat relasi surveyor pada activeSurvey di daftar konsultasi dan menampilkan nama surveyor di badge status pipeline survey, misalnya “Sedang Survey (Bagja)”.
- File berubah:
  - `E-REPORT-BACKEND_API/app/Http/Controllers/Api/ConsultationController.php`
  - `E-REPORT-FRONTEND/src/app/consultations/consultations-view.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Jika survey belum punya surveyor, label tetap memakai nama status tanpa tambahan.
- Agent: Codex

### 2026-07-28 - Hilangkan badge status survey dobel

- Area: frontend
- Ringkasan: Menyembunyikan badge active survey tambahan pada kartu konsultasi mobile ketika status pipeline sudah menampilkan status survey, sehingga tidak muncul dobel seperti “Sedang Survey” dan “Survey: in_progress”.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/consultations/consultations-view.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Badge “Survey aktif” tetap muncul bila pipeline belum berada di status survey tetapi lead sudah memiliki active_survey.
- Agent: Codex

### 2026-07-28 - Izinkan CORS broadcast auth

- Area: backend
- Ringkasan: Menambahkan `broadcasting/auth` ke path CORS agar frontend bisa mengotorisasi private channel Reverb/Echo dari origin Next.js. Tanpa ini listener realtime survey tidak bisa subscribe ke channel surveyor/manager.
- File berubah:
  - `E-REPORT-BACKEND_API/config/cors.php`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Tetap memakai daftar origin yang sudah dibatasi oleh FRONTEND_URLS dan pola IP jaringan lokal.
- Agent: Codex

### 2026-07-28 - Listener realtime survey frontend

- Area: frontend
- Ringkasan: Menambahkan listener Laravel Echo/Reverb di frontend untuk channel survey.managers dan survey.surveyor.{id}. Saat event survey.updated diterima, query Survey, notifikasi, dan dashboard langsung di-refresh agar penugasan manager muncul realtime di akun surveyor tanpa reload manual.
- File berubah:
  - `E-REPORT-FRONTEND/package.json`
  - `E-REPORT-FRONTEND/package-lock.json`
  - `E-REPORT-FRONTEND/src/components/providers.tsx`
  - `E-REPORT-FRONTEND/src/components/survey-realtime-listener.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Realtime membutuhkan service Reverb aktif; jika NEXT_PUBLIC_REVERB_HOST bernilai localhost saat akses via IP jaringan, frontend otomatis memakai hostname halaman saat ini.
- Agent: Codex

### 2026-07-28 - Refresh survey setelah kebutuhan konsultasi berubah

- Area: frontend
- Ringkasan: Menambahkan invalidasi query Survey saat data konsultasi dibuat, diubah, statusnya diganti, dihapus, atau diimpor agar kartu Survey Lokasi langsung mengambil kategori kebutuhan terbaru.
- File berubah:
  - `E-REPORT-FRONTEND/src/lib/hooks/useConsultations.ts`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Perubahan konsultasi akan memicu refetch Survey tambahan, tetapi diperlukan agar data turunan tidak stale.
- Agent: Codex

### 2026-07-28 - Gagal-aman broadcast realtime survey

- Area: backend
- Ringkasan: Membuat broadcast realtime survey gagal-aman saat Reverb/Pusher tidak bisa dihubungi, sehingga aksi survey tetap berhasil dan error koneksi websocket hanya dicatat sebagai warning.
- File berubah:
  - `E-REPORT-BACKEND_API/app/Http/Controllers/Api/SurveyController.php`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Jika Reverb mati, pembaruan realtime tidak terkirim sampai service dinyalakan, tetapi data dan notifikasi tetap diproses.
- Agent: Codex

### 2026-07-28 - Ambil kebutuhan survey dari kategori

- Area: backend / frontend
- Ringkasan: Menambahkan relasi kategori kebutuhan pada response Survey dan mengubah kartu Survey Lokasi agar field Kebutuhan hanya mengambil data dari kategori kebutuhan, bukan product_details atau requested_item.
- File berubah:
  - `E-REPORT-BACKEND_API/app/Http/Controllers/Api/SurveyController.php`
  - `E-REPORT-FRONTEND/src/app/surveys/surveys-view.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Data lama tanpa kategori akan tampil sebagai “Kebutuhan belum diisi”.
- Agent: Codex

### 2026-07-28 - Fallback jadwal assignment survey

- Area: frontend
- Ringkasan: Membuat dialog Jadwalkan Survey memakai Jadwal Awal dari requested_date/requested_time ketika tanggal atau jam survey dikosongkan oleh manager. Pesan error kini hanya muncul bila tanggal dialog dan Jadwal Awal sama-sama tidak tersedia.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/surveys/surveys-view.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Jika Jadwal Awal tidak punya jam, sistem memakai default 09:00 sebagai jam penugasan.
- Agent: Codex

### 2026-07-28 - Perjelas fallback kosong kartu Survey

- Area: frontend
- Ringkasan: Memperbaiki parsing Jadwal Awal agar tidak menampilkan Invalid Date dan mengganti nilai kosong pada kartu Survey Lokasi dengan informasi yang lebih sesuai, seperti lokasi belum dikonfirmasi, jadwal awal belum diajukan, nomor telepon belum tersedia, dan admin akun belum ditentukan.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/surveys/surveys-view.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Label fallback mengikuti istilah operasional saat ini; bisa disesuaikan lagi bila ada standar copy final.
- Agent: Codex

### 2026-07-28 - Perbaiki sumber Admin pada kartu Survey

- Area: backend / frontend
- Ringkasan: Mengubah sumber field Admin di kartu Survey Lokasi agar mengambil admin dari akun terkait melalui relasi account.admins, bukan user requester yang bisa berupa Super Admin.
- File berubah:
  - `E-REPORT-BACKEND_API/app/Http/Controllers/Api/SurveyController.php`
  - `E-REPORT-FRONTEND/src/types/index.ts`
  - `E-REPORT-FRONTEND/src/app/surveys/surveys-view.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Jika satu akun memiliki lebih dari satu admin, semua nama admin akan ditampilkan dipisahkan koma.
- Agent: Codex

### 2026-07-28 - Lengkapi informasi kartu Survey Lokasi

- Area: frontend
- Ringkasan: Menghapus duplikasi nama pada kartu Survey Lokasi, mengambil Jadwal Awal dari requested_date/requested_time yang diajukan admin, serta menambahkan informasi Akun dan Admin.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/surveys/surveys-view.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Perlu validasi data API pada survey lama yang belum memiliki requested_date agar fallback “Belum diajukan” sesuai ekspektasi.
- Agent: Codex

### 2026-07-28 - Redesain kartu Survey Lokasi mobile

- Area: frontend
- Ringkasan: Mendesain ulang kartu Survey Lokasi dengan menghapus top border kuning dan mengganti isi kartu menjadi daftar informasi eksplisit: nama konsumen, ID konsumen, status, lokasi, jadwal awal survey, nomor telepon, kebutuhan kategori, serta tombol Maps bila tersedia.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/surveys/surveys-view.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Perlu cek visual di data nyata untuk memastikan label panjang tetap terbaca pada layar kecil.
- Agent: Codex

### 2026-07-28 - Sederhanakan header Survey Lokasi

- Area: frontend
- Ringkasan: Menghapus kartu header besar dan ikon pada halaman Survey Lokasi, lalu menggantinya dengan judul modern rata tengah yang lebih bersih.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/surveys/surveys-view.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Tidak ada risiko fungsional; perubahan hanya visual.
- Agent: Codex

### 2026-07-28 - Hapus ringkasan kecil header survey

- Area: frontend
- Ringkasan: Menghapus pill ringkasan jumlah survey/status aktif di header Survey Lokasi agar header lebih bersih dan fokus ke segmented tab status.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/surveys/surveys-view.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Tidak ada risiko fungsional; jumlah dan status tetap dibaca dari daftar serta tab status.
- Agent: Codex

### 2026-07-28 - Samakan tab survey dengan liquid glass bottom bar

- Area: frontend
- Ringkasan: Mendesain ulang tab status survey agar selaras dengan active state bottom navigation mobile. Segmented control survey kini memakai surface solid, active pill liquid glass, highlight kaca, moving blob, dan spring transition saat berpindah status.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/surveys/surveys-view.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Perlu cek visual di mobile untuk memastikan label empat status tetap terbaca pada layar kecil.
- Agent: Codex

### 2026-07-28 - Hapus underline active tab bottom nav

- Area: frontend
- Ringkasan: Menghapus indicator underline pada active tab bottom navigation mobile agar efek aktif cukup dibaca dari capsule liquid glass, icon, dan label.
- File berubah:
  - `E-REPORT-FRONTEND/src/components/layout/bottom-nav.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Tidak ada risiko fungsional; hanya penyesuaian visual.
- Agent: Codex

### 2026-07-28 - Perkuat liquid glass active tab bottom nav

- Area: frontend
- Ringkasan: Mempertegas efek active tab bottom navigation mobile menjadi lebih liquid glass dengan capsule translucent, highlight kaca, inner depth, moving blob dua sisi, dan spring transition yang lebih lembut saat berpindah tab.
- File berubah:
  - `E-REPORT-FRONTEND/src/components/layout/bottom-nav.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Perlu cek visual langsung di perangkat mobile agar efek blur/glow tidak terasa terlalu berat.
- Agent: Codex

### 2026-07-28 - Pulihkan efek liquid active tab bottom nav

- Area: frontend
- Ringkasan: Mengembalikan karakter transisi active tab pada bottom navigation mobile agar terasa lebih liquid. Active tab kini memakai moving blob, glow halus, dan indicator dot yang berpindah dengan spring transition.
- File berubah:
  - `E-REPORT-FRONTEND/src/components/layout/bottom-nav.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Animasi halus perlu dicek di perangkat mobile low-end agar tetap ringan.
- Agent: Codex

### 2026-07-28 - Perjelas pemisahan bottom navigation mobile

- Area: frontend
- Ringkasan: Mendesain ulang surface bottom navigation mobile agar tidak menyatu dengan background halaman. Nav kini memakai backing gradient, surface solid yang lebih kontras, active tab dengan panel halus, dan tombol plus dengan ring background agar hierarki visual lebih jelas.
- File berubah:
  - `E-REPORT-FRONTEND/src/components/layout/bottom-nav.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Perlu cek langsung pada halaman konsultasi mobile karena background grid gelap bisa memengaruhi persepsi kontras.
- Agent: Codex

### 2026-07-28 - Pisahkan warna toolbar konsultasi mobile

- Area: frontend
- Ringkasan: Menyesuaikan warna toolbar konsultasi agar kontrol mobile tidak lagi menyatu dalam satu tint primary. Toolbar/search/filter dibuat netral, status `Belum diajukan` memakai amber lembut, tombol `Export` memakai tone slate sekunder, dan `Lead Baru` tetap menjadi aksi primary.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/consultations/consultations-view.tsx`
  - `E-REPORT-FRONTEND/src/app/globals.css`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Perlu cek langsung di dark mode dan light mode mobile untuk memastikan kontras visual sesuai preferensi user.
- Agent: Codex

### 2026-07-28 - Ringkas toolbar filter konsultasi mobile

- Area: frontend
- Ringkasan: Merapikan toolbar halaman Daftar Konsultasi di mobile agar tombol filter, export, dan aksi utama tidak menumpuk. Status/reset/refresh kini berada dalam baris utilitas yang lebih rapat, sedangkan aksi file pada mobile digabung menjadi satu tombol `Export` berisi Import CSV, Excel, dan PDF. Desktop tetap mempertahankan tombol file terpisah.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/consultations/consultations-view.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Perlu cek visual di layar mobile sebenarnya untuk memastikan lebar tombol `Export` dan `Lead Baru` tetap seimbang di berbagai ukuran perangkat.
- Agent: Codex

### 2026-07-28 - Kurangi nuansa template pada UI operasional

- Area: frontend
- Ringkasan: Merapikan bahasa navigasi, menurunkan efek glow/glass berlebihan, menyederhanakan bottom navigation mobile, dan mengganti elemen dashboard/analytics yang terlalu gamified menjadi tampilan operasional yang lebih tenang.
- File berubah:
  - `E-REPORT-FRONTEND/src/components/layout/sidebar.tsx`
  - `E-REPORT-FRONTEND/src/components/layout/header.tsx`
  - `E-REPORT-FRONTEND/src/components/layout/bottom-nav.tsx`
  - `E-REPORT-FRONTEND/src/app/dashboard/dashboard-view.tsx`
  - `E-REPORT-FRONTEND/src/app/analytics/analytics-view.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Perlu cek visual di mobile dan desktop setelah build untuk memastikan proporsi bottom nav, kartu dashboard, dan filter analytics tetap nyaman.
- Agent: Codex

### 2026-07-28 - Izinkan origin dev Tailscale untuk Next.js dan Laravel

- Area: frontend / backend
- Ringkasan: Menambahkan IP dev `100.105.166.15` agar akses dari perangkat lain di jaringan Tailscale tidak lagi diperingatkan atau ditolak saat pengembangan. Frontend Next.js kini mengizinkan origin tersebut lewat `allowedDevOrigins`, sementara backend Laravel membuka CORS untuk subnet `100.64.0.0/10` pada port `3000` dan menambahkan `100.105.166.15` ke daftar stateful Sanctum.
- File berubah:
  - `E-REPORT-FRONTEND/next.config.js`
  - `E-REPORT-BACKEND_API/config/cors.php`
  - `E-REPORT-BACKEND_API/config/sanctum.php`
- Validasi:
  - `php -l` untuk `config/cors.php` dan `config/sanctum.php` berhasil.
- Risiko / follow-up: agar perubahan backend aktif di environment yang sedang berjalan, reload config aplikasi Laravel atau restart service dev/server yang memuat konfigurasi.
- Agent: Codex

### 2026-07-28 - Selaraskan desain panel Menu Lainnya di mobile

- Area: frontend
- Ringkasan: Menyatukan tampilan panel `Menu Lainnya` pada bottom navigation mobile dengan bahasa visual aplikasi yang lebih gelap, rapat, dan konsisten. Header panel, grid shortcut, grup pengaturan, dan tombol logout kini memakai surface, border, dan ritme spacing yang selaras dengan nav bawah serta kartu operasional lain.
- File berubah:
  - `E-REPORT-FRONTEND/src/components/layout/bottom-nav.tsx`
- Validasi:
  - `npm run build` lokal berhasil.
- Risiko / follow-up: Perubahan ini baru divalidasi lewat build. Jika ingin, langkah berikutnya paling pas adalah cek visual langsung di device mobile atau deploy ke server produksi.
- Agent: Codex

### 2026-07-27 - Konfigurasi Reverb produksi dan perbaikan layout konsultasi mobile

- Area: backend / frontend / ops
- Server: `mki@100.76.23.12:2201`, backend `/www/wwwroot/api-ereport.interiorcustom.id`, frontend `/www/wwwroot/e-report-frontend`.
- Ringkasan: Mengaktifkan Laravel Reverb di server produksi sebagai service systemd `ereport-reverb.service`, menghubungkan broadcast Laravel ke Reverb, menambahkan proxy Nginx untuk endpoint WebSocket `/app/*` dan `/apps/*`, serta memastikan queue Redis tetap aktif. Halaman daftar konsultasi mobile diperbaiki agar toolbar lebih ringkas dan daftar lead tampil sebagai kartu mobile; tabel penuh tetap dipakai di desktop.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/consultations/consultations-view.tsx`
  - Konfigurasi server produksi: `.env` backend/frontend, `/etc/systemd/system/ereport-reverb.service`, dan Nginx extension `reverb.conf`.
- Validasi:
  - `npm run build` lokal berhasil.
  - `npm run build` produksi berhasil.
  - `ereport-reverb.service` aktif dan listen di `127.0.0.1:8080`.
  - Handshake WebSocket publik berhasil dengan `101 Switching Protocols`.
  - `ereport-queue.service` aktif.
- Risiko / follow-up: Frontend saat ini belum memasang client Echo/Pusher untuk memakai event realtime di UI; infrastruktur Reverb sudah siap untuk disambungkan ke notifikasi live.
- Agent: Codex

### 2026-07-27 - Restore database produksi dari dump e_report.sql

- Area: database / ops
- Server: `mki@100.76.23.12:2201`, backend `/www/wwwroot/api-ereport.interiorcustom.id`, database `sql_api_ereport_interiorcustom_id`.
- Ringkasan: Mengganti isi database produksi E-Report dengan dump `C:\Users\Administrator\Downloads\e_report.sql` sesuai instruksi pemilik proyek. Aplikasi dimasukkan ke maintenance mode, queue `ereport-queue` dihentikan sementara, seluruh tabel lama di-drop, dump diimpor, cache Laravel dibersihkan, aplikasi diaktifkan kembali, dan queue dinyalakan lagi.
- Backup sebelum restore: `/home/mki/db-backups/e_report_server_before_restore_20260727-143510.sql.gz`.
- Hasil verifikasi setelah restore:
  - `accounts`: 20
  - `users`: 19
  - `consultations`: 285 total, 284 aktif, 1 soft-deleted
  - `migrations`: 60
  - `BROTO INTERIOR` berada pada `accounts.id = 15`, `account_group = NPP2`.
  - API publik `https://api-ereport.interiorcustom.id/api/v1/auth/me` merespons 401, menandakan aplikasi live dan auth guard aktif.
- File berubah: `E-REPORT-APP/10-Change-Log/Change-Log.md`.
- Risiko / follow-up: Login/user produksi mengikuti dump baru; bila perlu rollback, gunakan backup SQL di atas. File dump juga disalin ke server di `/home/mki/imports/e_report_restore_20260727.sql`.
- Agent: Codex

### 2026-07-27 - Deploy ke server produksi (aaPanel, serverone)

- Area: ops
- Server: `mki@100.76.23.12:2201` (Ubuntu 24.04, aaPanel). Backend `/www/wwwroot/api-ereport.interiorcustom.id`, frontend `/www/wwwroot/e-report-frontend`. Akses lewat kunci `ereport_deploy`; host key port 2201 diverifikasi cocok dengan fingerprint yang diberikan pemilik server (`SHA256:HH8Ilerb…AWrAxg`).
- **Jarak deploy jauh lebih besar dari perkiraan**: backend tertinggal **9 commit** (`1dc2b33` → `37448ed`), frontend **17 commit** (`4218b70` → `b854426`). Seluruh modul Survey belum pernah dideploy — `App\Models\Survey` tidak ada di server sebelum ini.
- Kendala yang ditemukan dan ditangani:
  - Working tree kedua repo **kotor** (artefak runtime `storage/app/imports/*`, `bootstrap/cache/.gitignore`, `package-lock.json`). `git pull` polos di `deploy.sh` akan berhenti. Diselesaikan dengan `git stash push -u` sebelum pull.
  - Repo dimiliki `www:www`; user `mki` tidak punya izin tulis ke `.git`. Semua perintah git/composer/artisan/npm dijalankan sebagai `www` lewat sudo.
  - `deploy.sh` di repo menyasar path **Hostinger** (`/home/u603012205/...`), bukan aaPanel. Tidak dipakai; langkahnya dijalankan manual dengan path yang benar.
- Langkah: backup DB → stash → pull → `composer install --no-dev` → `migrate --force` → `optimize:clear` → `config/route/view:cache` → `queue:restart` → reload `php-fpm-83`. Frontend: stash → pull → `npm ci` (935 paket) → `npm run build` → `pm2 restart e_report_frontend`.
- **Backup sebelum migrasi**: `~/backups/ereport-20260727-135723.sql.gz` (61 KB, 24 tabel, integritas gzip diverifikasi).
- **20 migrasi dijalankan** (19 baru + 1 sudah ada): seluruh tabel modul Survey, `widen_users_role_column`, `extend_users_role_enum`, `add_account_group_to_accounts_table`, `rename_pending_needs_category_label`, `sync_completed_survey_pipeline_status`. Sisa pending: 0.
- Verifikasi perbaikan keamanan aktif di server: `sanctum.expiration` = 720, `statuses.survey_entry` = `Request Survey`, `ConsultationStatusGroups::surveyEntryName()` jalan, `scopeNeedsSurveyRequest()` mengembalikan 11 (sebelumnya fatal `trim()` atas array), tabel `surveys` ada.
- Smoke test: API `/up` 200 (lokal + HTTPS publik), `/api/v1/auth/me` dan `/api/v1/export/csv` 401 (auth aktif). Frontend `127.0.0.1:3000/login` 200, `https://putrainterior.com` 200. Queue worker proses baru. Nol error backend baru.
- **Redis sehat**: write/read ok lewat Laravel, client `predis`, dipakai cache + queue + session. `redis-cli` tidak bisa diakses user `mki` (permission denied), jadi diuji lewat jalur aplikasi.
- **Data server TIDAK disentuh** — 209 lead tetap utuh. Lihat catatan di bawah.
- Dampak yang sudah diperkirakan: dari 7 token aktif, **6 lebih tua dari 12 jam** sehingga pengguna terkait diminta login ulang. Ini konsekuensi `sanctum.expiration` yang memang disengaja.
- Temuan sampingan (bukan akibat deploy): `e-report.kitchensetbandungbarat93.com` yang terdaftar di `FRONTEND_URLS` dan `SANCTUM_STATEFUL_DOMAINS` **tidak resolve di DNS** dan tidak punya vhost nginx. Entri usang; frontend live ada di `putrainterior.com`.
- Risiko / follow-up: stash di kedua repo belum di-drop (`git stash list`) — biarkan sebagai jaring pengaman sampai deploy dipastikan stabil. Pertimbangkan `.gitignore` untuk `storage/app/imports/*` supaya artefak runtime tidak lagi mengotori working tree tiap deploy.
- Agent: Claude

### 2026-07-27 - Lokal dan server adalah dua lingkungan berbeda, bukan salinan

- Area: data
- Ringkasan: permintaan "pertahankan data konsul lokal, timpa yang di server" **tidak dijalankan** karena premisnya tidak sesuai kenyataan.
- Bukti: server punya **39 akun / 209 lead**, lokal **20 akun / 284 lead**, dan penomoran akunnya berbeda total — id 17 = `BROTO INTERIOR` di server tapi `NACIRA STUDIO` di lokal; id 16 = `INTERIOR MODERN` di server tapi `FULLHOME ID` di lokal.
- Berkas xlsx yang dipakai untuk impor lokal berisi tepat **209 baris** — sama persis dengan jumlah lead server. Berkas itu **export dari server**, bukan dari lokal. Ini sekaligus menjelaskan kenapa importer menolak dan meregenerasi ID saat impor lokal: prefix ID mengikuti `account_id` server, yang menunjuk perusahaan lain di lokal.
- Konsekuensi bila tetap disalin: setiap lead menempel ke perusahaan yang salah, dan 209 lead produksi + 43 yang ter-soft-delete tertimpa. Tidak dapat dibatalkan tanpa backup.
- Keputusan: hanya kode yang dideploy. Data server dibiarkan utuh, menunggu konfirmasi ulang pemilik.
- Agent: Claude

### 2026-07-27 - Import 209 lead dari xlsx "data-leads-semua-akun-juli-2026 (15)"

- Area: data
- Ringkasan: berkas xlsx (hasil export app, sheet `Data Leads`, 209 baris data) diimpor lewat `ConsultationImportService` — service yang sama dengan tombol Import di aplikasi, supaya seluruh aturan app ikut berlaku. Hasil: **209 baris, 0 error, 16 lead baru, 193 dicocokkan ke lead yang sudah ada**. Total lead 268 → 284.
- Dua penyesuaian yang harus dikerjakan di luar importer:
  1. **Tanggal serial Excel** (mis. `46230`). `parseDate()` hanya mengenal string tanggal; serial mentah menghasilkan `null` dan baris jatuh ke `now()` — seluruh 209 lead akan kehilangan tanggal konsul aslinya tanpa error apa pun. Dikonversi ke `d/m/Y` sebelum impor. Terverifikasi: rentang tanggal di DB `2026-04-23 .. 2026-07-27`, hanya 3 baris bertanggal hari ini (memang begitu di berkas).
  2. **Status "Menunggu Bangunan"** (4 baris) tidak ada di `status_categories` — yang ada **"Nunggu Bangunan"**. Tanpa dipetakan, `$statusCategoryMap[...] ?? $defaultStatusId` membuatnya diam-diam jadi "Masih konsultasi". Dipetakan sebelum impor; terverifikasi 4/4 mendarat di "Nunggu Bangunan".
- **"Belum ada konfirmasi"** (30 baris kolom Jenis Kebutuhan) sengaja **tidak** dipetakan manual: importer sudah menanganinya lewat `NeedsCategory::PENDING_LEGACY_LABEL` → `Tidak konfirmasi`.
- Normalisasi wilayah berjalan sesuai aturan app — 84 nilai disesuaikan otomatis:
  - `DKI Jakarta` → `Jakarta` (34 baris)
  - `Belum ada konfirmasi` → `Belum konfirmasi` (13 provinsi + 18 kota)
  - Tanpa awalan → berawalan: `Bandung` → `Kota Bandung`, `Sidoarjo` → `Kab. Sidoarjo`, `Surabaya` → `Kota Surabaya`, dll.
  - Ejaan diperbaiki: `Kota B A T A M` → `Kota Batam`, `Kab. Mahakam Hulu` → `Kab. Mahakam Ulu`
- **ID konsultasi dari berkas tidak dipertahankan.** `normalizeImportedConsultationId()` mensyaratkan dua digit awal ID sama dengan `account_id`. Berkas memakai penomoran akun berbeda — mis. `17.2607.0126` berprefix 17 (= NACIRA STUDIO di DB ini) padahal kolom akunnya BROTO INTERIOR (id 15) — jadi ID ditolak dan diregenerasi sesuai akun yang benar. Ini aturan app bekerja: mempertahankan prefix asing justru merusak keterkaitan ID dengan akun.
- Verifikasi: rekonsiliasi seluruh berkas ke DB lewat nama + nomor WA — **209/209 cocok, 0 tidak ditemukan, 0 status meleset**. Selisih wilayah 84 seluruhnya adalah hasil normalisasi yang benar, bukan kesalahan.
- Catatan: dari 193 baris "diperbarui", hanya ~23 yang benar-benar mengubah data (`updated_at` tersentuh); sisanya menulis nilai identik karena berkas ini memang export dari basis data yang sama.
- Risiko / follow-up: tidak ada baris hilang atau terduplikasi. Bila ID asli dari berkas memang harus dipertahankan, itu memerlukan penyelarasan `account_id` antar lingkungan lebih dulu — bukan perubahan importer.
- Agent: Claude

### 2026-07-27 - SEC-002/003/004: token auth keluar dari URL export

- Area: backend, frontend
- Severity: **Critical** (insiden nyata) + **High** (cakupan middleware) + **Medium** (token tanpa masa berlaku)
- **Insiden**: token super admin (`id=36`, user 1, `superadmin@npp.com`) tertempel di URL export dan **keluar dari mesin**. `expires_at` kosong dan `sanctum.expiration` bernilai `null` — token itu berlaku selamanya. **Pencabutan belum dilakukan**: perintah hapus diblokir classifier izin, harus dijalankan manual (lihat "Tindakan tertunggak").
- Akar penyebab: `buildExportUrl()` menempelkan token ke query string supaya unduhan `window.open`/`<a href>` bisa menembus auth. Tujuh call site merendernya sebagai `href`, jadi URL bertoken masuk DOM, riwayat browser, header `Referer`, dan apa pun yang disalin atau di-screenshot.
- Mekanisme yang benar sebenarnya sudah ada — `api.downloadFile()` mengirim token di header `Authorization` — tapi baru dipakai **1 tempat** melawan **14** pemakai `buildExportUrl`.
- Perbaikan:
  - Tujuh anchor export pindah ke hook baru `useFileDownload()` → `api.downloadFile()`.
  - `buildExportUrl()` **dihapus** (CONFIRMED_UNUSED setelah call site terakhir dipindah). Membiarkannya tersedia berarti polanya bisa kembali. Satu pemakai non-export hanya menyambung base URL untuk gambar `/storage/` — jadi template string biasa.
  - `QueryTokenToHeader` dulu di-*prepend* ke seluruh grup `api`, jadi **semua 105 route** menerima `?token=`, termasuk yang menulis. Kini dibatasi **GET pada sembilan route export** saja. Middleware **dipertahankan, bukan dihapus**, karena tautan bertoken mungkin sudah beredar (bookmark, pesan, dokumen) — ditandai deprecated.
  - `sanctum.expiration` `null` → **720 menit (12 jam)**, dapat disetel lewat `SANCTUM_EXPIRATION`.
  - **401 tidak ditangani di mana pun.** `/auth/me` hanya jalan saat mount, jadi sesi yang ditolak di tengah pemakaian tidak pernah dibersihkan — pengguna hanya melihat permintaan gagal sampai memuat ulang sendiri. Tidak terasa selama token abadi; menjadi jalur yang pasti dilewati begitu token diberi masa berlaku. Kini 401 membersihkan sesi dan mengarahkan ke `/login`, kecuali di `/auth/login` (401 di sana = sandi salah, urusan form).
  - Kode mati ikut terbawa: `isMounted` di tiga view hanya ada untuk menunda `buildExportUrl` melewati hydration — handler `onClick` tidak jalan saat render, jadi dibuang.
- File berubah:
  - `E-REPORT-BACKEND_API/app/Http/Middleware/QueryTokenToHeader.php`, `config/sanctum.php`
  - `E-REPORT-FRONTEND/src/lib/hooks/useFileDownload.ts` (baru), `src/lib/api/client.ts`, `src/app/{analytics,consultations,accounts,rekap-jadwal-surveyor,report-attendances}/*-view.tsx`
- Verifikasi: `php -l` OK; delapan kasus middleware lulus (export GET diizinkan; GET non-export, POST, dan DELETE ditolak); sembilan route export dicocokkan ke allowlist lewat `route:list`; `sanctum.expiration` terbaca 720. Frontend: `npx tsc --noEmit` bersih, `npx eslint` bersih di semua path tersentuh.
- **Tindakan tertunggak (jalankan manual)**: cabut token yang terekspos —
  `php artisan tinker --execute="DB::table('personal_access_tokens')->where('id',36)->delete();"` lalu login ulang.
- **Catatan kebersihan riwayat**: commit frontend `b854426` ikut membawa 17 berkas pekerjaan AbortSignal milik agent Codex yang belum ter-commit (`src/lib/hooks/use*.ts`, `src/lib/prefetch.ts`), karena `git add -A src/` menyapu semuanya. Kodenya utuh dan terverifikasi — 15 hook masih memakai `signal` — tapi pesan commit itu hanya menjelaskan perubahan keamanan. Riwayat tidak ditulis ulang karena sudah ter-push. Cari perubahan AbortSignal di commit tersebut, bukan di commit tersendiri.
- Risiko / follow-up: `QueryTokenToHeader` masih menerima token di query untuk export GET — hapus setelah dipastikan tidak ada tautan lama beredar. Token 12 jam berarti login ulang tiap hari kerja; sesuaikan `SANCTUM_EXPIRATION` bila terlalu ketat.
- Agent: Claude

### 2026-07-27 - Batalkan request data saat query tidak lagi aktif

- Area: frontend/performance
- Ringkasan: Menambahkan dukungan `AbortSignal` pada API client frontend dan meneruskannya dari TanStack Query untuk request read-only. Query dashboard, analytics, consultations, survey, recap surveyor, audit logs, report attendance, notifications, master data, accounts, geo analytics, auth/me, online users, debug stats, bug reports, dan prefetch route kini bisa dibatalkan saat filter berubah cepat, halaman berpindah, atau komponen unmount.
- File berubah:
  - `E-REPORT-FRONTEND/src/lib/api/client.ts`
  - `E-REPORT-FRONTEND/src/lib/prefetch.ts`
  - `E-REPORT-FRONTEND/src/lib/hooks/use*.ts`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Perlu benchmark browser untuk melihat penurunan request selesai setelah navigasi cepat; mutasi tetap tidak dibatalkan otomatis agar aksi simpan/delete tidak berakhir setengah jelas.
- Agent: Codex

### 2026-07-27 - SEC-001: Stored XSS lewat import CSV (Critical)

- Area: backend
- Severity: **Critical** — stored XSS berujung pengambilalihan akun super admin.
- Ringkasan: `AnalyticsReportService::buildInsights()` menyusun kalimat insight dengan `sprintf` dan menyisipkan teks dari DB **tanpa escape**; frontend merendernya lewat `dangerouslySetInnerHTML` (`analytics-view.tsx:1021`). Sumber teksnya bisa berisi HTML karena `ConsultationImportService` menyimpan sel CSV apa adanya saat nama wilayah tak dikenali (`canonicalProvince($row[7]) ?? $row[7]`) dan importer **tidak punya aturan validasi sama sekali**. Route impor ada di grup `role:admin`, sedangkan Analitik dibaca super admin lintas akun — dan token auth ada di `localStorage`, jadi payload yang jalan di sana berarti akun super admin diambil alih.
- Catatan penting: seluruh jalur tulis lewat API **sudah** dijaga (`ConsultationRequest:187-190` regex wilayah, `/^[^<>]+$/` untuk alamat, `regex:/^[^\<\>]+$/` untuk nama user/kategori/status). Hanya jalur impor yang terlewat — jadi ini celah, bukan keputusan desain.
- Perbaikan dua lapis:
  1. `buildInsights()` — semua nilai dari DB kini lewat `e()`. Escape di argumen `sprintf`, bukan hasilnya, supaya `<mark>` tetap utuh. Ini menutup celah termasuk untuk data kotor yang sudah terlanjur tersimpan.
  2. `ConsultationImportService::stripHtmlDelimiters()` — `<` dan `>` dibuang dari `client_name`, `province`, `city`, `district`, `product_details`, `notes`. Karakter dibuang, bukan baris ditolak, supaya impor yang selama ini berhasil tidak gagal.
- File berubah:
  - `E-REPORT-BACKEND_API/app/Services/Reports/AnalyticsReportService.php`
  - `E-REPORT-BACKEND_API/app/Services/ConsultationImportService.php`
- Verifikasi: `php -l` OK. Uji reflection: 7 kalimat insight dengan payload `<img src=x onerror=...>` di enam field → **nol** `<img` mentah, 6 kali ter-escape, `<mark>` utuh; `stripHtmlDelimiters` membuang payload, aman untuk `null`, dan tidak menyentuh `Kab. Bandung Barat`. Smoke test data nyata Juli 2026: 9 insight, 245 lead, workbook 566 KB.
- Risiko / follow-up: nihil untuk data sah. **Data lama yang sudah tercemar tetap tersimpan** — lapis 1 membuatnya tidak berbahaya saat dirender, tapi belum ada pembersihan retroaktif. Pertimbangkan audit `SELECT` pada `province`/`city` yang memuat `<`.
- Agent: Claude

### 2026-07-27 - Fix: grup status survey dibaca sebagai string tunggal

- Area: backend
- Ringkasan: `statuses.survey` diperluas jadi tiga nama untuk pelaporan, tapi empat pemakainya masih membacanya sebagai string. Yang terlihat user: export Analitik Excel 500 — `escapeFormulaString(array)` TypeError. Di baliknya ada bug diam: `COUNTIF` di exporter hanya menghitung satu dari tiga status, jadi sel "Total Survey" **sudah salah hitung sebelum crash**. Tiga sisanya fatal di jalur yang belum sempat dijalankan: `Consultation::scopeNeedsSurveyRequest` (menyuapi badge notifikasi, filter daftar, notifikasi pasca-impor), guard `ConsultationController:411`, dan gerbang `SurveyController:87` — semuanya `trim()` atas array.
- Pemisahan konsep: grup pelaporan dan tahap pengajuan sempat jadi satu kunci, sehingga melebarkan grup ikut melebarkan gerbang — survey jadi bisa diajukan pada lead yang surveynya **sudah selesai**. Kini `statuses.survey` (grup pelaporan) dan `statuses.survey_entry` (tahap pengajuan) terpisah, keduanya lewat `ConsultationStatusGroups`. Lihat ADR 2026-07-27 di Decision Log.
- File berubah: `config/statuses.php`, `app/Support/ConsultationStatusGroups.php`, `app/Services/Reports/AnalyticsExcelExporter.php`, `app/Models/Consultation.php`, `app/Http/Controllers/Api/ConsultationController.php`, `app/Http/Controllers/Api/SurveyController.php`
- Verifikasi: `php -l` semua OK. Data nyata: workbook 566 KB terbentuk, formula survey menjumlah ketiga nama, `needsSurveyRequest` = 14, badge notifikasi jalan untuk super admin (14) dan admin (2), status id 2 = `Request Survey`.
- Risiko / follow-up: `Account.php`, `AccountController`, `AuditDatabase` **masih** membaca `config('statuses.deal')` langsung. Aman sekarang karena `deal` masih string tunggal, tapi rapuh dengan cara yang sama bila kelak dilebarkan.
- Agent: Claude

### 2026-07-27 - Absensi: filter khusus super admin + pengingat di dashboard

- Area: frontend
- Ringkasan: Panel filter absensi (tanggal, status, mode Harian/Rekap) adalah kontrol pemantauan — admin hanya melaporkan absensinya sendiri untuk hari ini — jadi kini khusus super admin. State yang dikendalikannya tetap di nilai bawaan, yang justru dibutuhkan kartu laporan admin. Ditambah banner pengingat di Dashboard untuk admin yang belum absen: halaman absensi tidak dilewati saat kerja normal, jadi yang lupa baru sadar setelah harinya lewat.
- `useReportAttendances` menerima opsi `enabled` supaya super admin — yang tidak pernah melihat banner — tidak ikut menembak endpoint.
- File berubah: `src/app/report-attendances/report-attendances-view.tsx`, `src/app/dashboard/dashboard-view.tsx`, `src/lib/hooks/useReportAttendances.ts`
- Verifikasi: `npx tsc --noEmit` OK.
- Agent: Claude

### 2026-07-27 - UI: tipografi, bahasa ikon, dan penyelarasan layout

- Area: frontend
- Ringkasan: beberapa perubahan tampilan yang saling terkait.
  - **Tipografi**: Plus Jakarta Sans → **Inter** (body) + **Space Grotesk** (heading), JetBrains Mono tetap.
  - **Bahasa ikon**: tabrakan metafora dibereskan — `Globe2` dan `MapPinned` sama-sama terbaca "peta", kini `Map` untuk Analisis Wilayah dan `ClipboardCheck` untuk Survey. `DatabaseZap` → `Database`, `FileClock` → `CalendarCheck`, `SlidersHorizontal` → `Settings`, `LayoutGrid` → `LayoutDashboard`. Akun memakai `Building2` yang sama dengan tile dashboard-nya. Stroke sidebar 2.2 → 1.75.
  - **KPI dashboard**: lima tile memakai chip amber identik sehingga barisnya tidak bisa dipindai; tiap metrik pendukung kini punya warna sendiri. Hero dan tile pendukung yang tadinya dua komponen hasil salin-tempel — dan sudah melenceng di tiap sambungan — digabung jadi satu komponen bervarian. Tiga aturan menahan kesejajaran: slot kanan-atas selalu `h-9`, aksen selalu hairline atas, konten dijangkar dari atas (dijangkar dari bawah membuat tile ber-badge naik satu baris).
  - **Mobile**: KPI jadi dua kolom; kartu login tidak lagi dipaksa `min-h-[600px]` di semua breakpoint padahal kolom aside hanya muncul di `lg`; bottom-nav admin kehilangan tombol "Lainnya" karena sheet-nya tinggal satu tujuan — `Pengaturan` sudah ada di popover profil header.
  - **Header**: tiga tombol aksi tersebar di tiga file dan ukurannya melenceng (36/36/40 px, radius berbeda) — geometrinya dipindah ke satu konstanta. Tinggi brand sidebar (`h-16`) dan header halaman (`h-14`) disamakan supaya border bawahnya menyambung.
  - **Bottom-nav**: dibangun ulang mengikuti bahasa material Apple — seleksi lewat tint, bukan kotak ber-border; spotlight neon, pill ber-border, dan cincin FAB berdenyut dibuang. Ikut terbawa: tab non-aktif tadinya `#9CA3AF` hardcoded (kontras jatuh di mode terang) kini pakai token `muted-foreground`.
  - **Geo map**: `will-change:transform;position:relative` dibuang dari root marker — properti itu mengganggu transform MapLibre.
  - Halaman `/debug` dihapus beserta entri navigasinya.
- File berubah: `src/app/layout.tsx`, `src/app/globals.css`, `src/app/dashboard/dashboard-view.tsx`, `src/app/login/page.tsx`, `src/app/geo-analytics/_components/geo-map.tsx`, `src/components/layout/{sidebar,bottom-nav,header,header-action,theme-toggle,pwa-install-button,notification-center}.*`, `src/app/debug/` (dihapus)
- Verifikasi: `npx tsc --noEmit` OK, `npx eslint src/components/layout` bersih. **Belum diverifikasi visual** — Browser pane tidak tampil sepanjang sesi, jadi perubahan tampilan ini belum dilihat hasil render-nya.
- Risiko / follow-up: warna aksen KPI bersifat tetap; bila user mengganti tema ke biru, aksen `sky` bisa berdekatan. Verifikasi visual masih tertunggak.
- Agent: Claude

### 2026-07-27 - Docs: README backend & frontend

- Area: docs
- Ringkasan: endpoint `GET /geo-analytics` ditambahkan ke tabel API README backend; fitur Geo Analytics dan folder `geo-analytics/` ditambahkan ke README frontend. Header kedua README dirapikan (judul rata tengah, badge ber-tautan, nav cepat, badge lisensi MIT).
- File berubah: `E-REPORT-BACKEND_API/README.md`, `E-REPORT-FRONTEND/README.md`
- Agent: Claude

### 2026-07-26 - Fix: marker peta hilang + posisi pin di laut

- Area: frontend
- Ringkasan: Pin marker provinsi hilang atau muncul di laut (Samudera Hindia) karena dua bug: (1) `@turf/centroid` pada MultiPolygon kepulauan menghasilkan titik geometris di laut — diganti `@turf/point-on-feature` yang menjamin titik selalu di dalam daratan; (2) centroid dihitung di `useEffect` (async setelah paint) sehingga race condition dengan `map.on('load')` — dipindah ke `useMemo` supaya tersedia sinkron sebelum effect pertama; (3) click handler pakai `selectedProvince` dari stale closure — diubah baca dari `dataRef.current`. Juga fix style provinsi line di basemap-switch handler yang inkonsisten.
- File berubah: `E-REPORT-FRONTEND/src/app/geo-analytics/_components/geo-map.tsx`
- Dependensi baru: `@turf/point-on-feature`
- Risiko / follow-up: Tidak ada. `pointOnFeature` lebih berat CPU dari `centroid` tapi di-memo sekali.
- Agent: Claude

### 2026-07-26 - Fix: downloadFile "Failed to fetch" (prefix /api/v1 hilang)

- Area: frontend
- Ringkasan: Tombol Export Excel user gagal "Failed to fetch". Akar: `api.downloadFile()` membangun URL `${baseUrl}${path}` TANPA `/api/v1`, sedangkan `request()` (get/post/dst) pakai `${baseUrl}/api/v1${path}`. Akibatnya request ke `:8000/master-data/users/export` → 404 tanpa header CORS → fetch ditolak browser. Fix: `downloadFile` kini menambah `/api/v1` konsisten dengan `request()`; caller mengirim path seperti `get()` (`/master-data/users/export`).
- File berubah: `E-REPORT-FRONTEND/src/lib/api/client.ts`
- Verifikasi: `npx tsc --noEmit` OK. `curl :8000/api/v1/master-data/users/export` → 401 (route hidup + auth guarded, bukan lagi network error).
- Agent: Claude

### 2026-07-26 - Export Excel daftar user (master data akun)

- Area: backend, frontend
- Ringkasan: Tombol **Export Excel** di tab "Akun Pengguna Admin" (master-data). Mengunduh xlsx berstyle berisi semua user: NO, Nama, Email, Role, Tautan Akun, Tanggal Dibuat. Menghormati filter pencarian aktif. Header teal, baris zebra, freeze pane.
- **KEAMANAN — kolom sandi DITOLAK**: permintaan awal minta "tampilkan sandi", tapi password disimpan hash bcrypt satu-arah — plaintext tidak ada di DB, tak bisa didekripsi/ditampilkan. Mengekspor hash pun berbahaya (crack offline bila bocor). Kolom sandi sengaja tidak ada. Alternatif aman sudah tersedia: endpoint `users.reset-password` (generate sandi sementara).
- File berubah:
  - `E-REPORT-BACKEND_API/app/Services/Reports/UsersExcelExporter.php` (baru, SpreadsheetML 2003 → xlsx)
  - `E-REPORT-BACKEND_API/app/Http/Controllers/Api/MasterDataController.php` (`exportUsers`)
  - `E-REPORT-BACKEND_API/routes/api.php` (`GET /master-data/users/export`, dalam grup `role:super_admin`)
  - `E-REPORT-FRONTEND/src/app/master-data/master-data-view.tsx` (tombol + `handleExportUsers` via `api.downloadFile`)
- Verifikasi: `php -l` OK; tinker: 19 user → xlsx valid (magic `504b`/PK). `route:list` menampilkan `api.master-data.users.export`. `npx tsc --noEmit` OK.
- Risiko / follow-up: akses super_admin only (grup master-data + `ensureSuperAdmin()`). Format xlsx via converter internal, bukan phpspreadsheet.
- Agent: Claude

### 2026-07-26 - Akses Geo Analytics dibatasi super_admin saja

- Area: frontend, backend
- Ringkasan: Halaman & API Geo Analytics sebelumnya boleh diakses `admin` (+ super_admin). Diubah jadi **super_admin saja**. Guard berlapis: (1) backend route `role:admin` → `role:super_admin` (mencegah bypass via panggilan API langsung); (2) `ROUTE_ROLES['/geo-analytics']` → `['super_admin']` (AuthGuard redirect non-super ke fallback); (3) sidebar `roles: ['super_admin']` & bottom-nav `superOnly: true` (sembunyikan menu).
- File berubah:
  - `E-REPORT-BACKEND_API/routes/api.php` (middleware `role:super_admin`)
  - `E-REPORT-FRONTEND/src/lib/auth/roles.ts` (ROUTE_ROLES)
  - `E-REPORT-FRONTEND/src/components/layout/sidebar.tsx`
  - `E-REPORT-FRONTEND/src/components/layout/bottom-nav.tsx`
- Verifikasi: `npx tsc --noEmit` OK. Middleware `role` cocokkan exact (pola sama dengan audit-logs/debug/accounts).
- Risiko / follow-up: role `admin` kehilangan akses peta wilayah — sesuai permintaan.
- Agent: Claude

### 2026-07-26 - Fix: pin marker hilang setelah klik wilayah / zoom (stale closure)

- Area: frontend
- Ringkasan: Bug — pin bernomor muncul saat awal load, lalu HILANG total setelah user klik provinsi (mis. Jawa Barat) yang memicu `flyTo`. Akar: `map.on('zoomend', renderMarkers)` dan `map.on('load', ...)` di-bind sekali di init-effect sehingga menangkap closure `renderMarkers`/`applyData` dengan `provinces`/`cities` KOSONG (data react-query datang belakangan). Zoom apa pun (termasuk `flyTo`) memanggil closure lama itu: marker lama dihapus, tak ada pengganti. Diverifikasi via repro statis di pane (marker DOM diposisikan lewat `project()` tanpa render loop) — `buildMarkerEl` terbukti render normal (ukuran 54×54 dst.), jadi biang bukan elemen marker melainkan closure. Fix: simpan `{ provinces, cities, selectedProvince }` di `dataRef` yang diperbarui tiap render; `renderMarkers`/`applyData` baca `dataRef.current` → handler map selalu pakai data terbaru.
- File berubah: `E-REPORT-FRONTEND/src/app/geo-analytics/_components/geo-map.tsx`
- Verifikasi: `npx tsc --noEmit` OK.
- Risiko / follow-up: pola — handler event MapLibre yang di-bind sekali WAJIB baca state dari ref, jangan dari closure prop/state.
- Agent: Claude

### 2026-07-26 - Fix: pin marker MapLibre loncat ke pojok saat hover

- Area: frontend
- Ringkasan: Bug — user lapor pin bernomor loncat ke pojok kanan atas saat di-hover dan tak bisa diklik. Akar: MapLibre memasang `transform: translate(...)` pada elemen ROOT marker untuk memposisikannya; handler hover menimpa dengan `el.style.transform = 'scale(...)'`, menghapus translate hingga pin pindah ke koordinat 0,0. Fix: bungkus visual pin di elemen anak (`inner`), hover-scale hanya di `inner`; root tak pernah disentuh transform-nya (cuma anchor posisi). Klik pulih otomatis karena pin tak lagi berpindah.
- File berubah: `E-REPORT-FRONTEND/src/app/geo-analytics/_components/geo-map.tsx` (`buildMarkerEl`)
- Verifikasi: `npx tsc --noEmit` OK. Visual dikonfirmasi user via screenshot (pin render); perbaikan hover deterministik.
- Risiko / follow-up: tak ada; jangan set `transform` di elemen root marker MapLibre mana pun.
- Agent: Claude

### 2026-07-26 - Geo Analytics: peta MapLibre + marker bernomor berjenjang

- Area: frontend
- Ringkasan: Mengganti peta Geo Analytics dari SVG choropleth ke **MapLibre GL JS** (`maplibre-gl@5.16.0`) atas permintaan: basemap raster OSM (toggle), fill kabupaten + garis provinsi, `NavigationControl`/`FullscreenControl`, dan **pin marker bernomor berjenjang** (provinsi saat zoom jauh, kabupaten saat zoom dekat; angka = total konsultasi; centroid via `@turf/centroid`).
- File berubah:
  - `E-REPORT-FRONTEND/src/app/geo-analytics/_components/geo-map.tsx` (rewrite ke MapLibre)
  - `E-REPORT-FRONTEND/src/app/geo-analytics/geo-analytics-view.tsx` (kirim `provinces` ke GeoMap)
  - `E-REPORT-FRONTEND/package.json` (+`maplibre-gl@5.16.0`, +`@turf/centroid`; `dev` diubah ke Webpack, tambah `dev:turbo`)
  - `E-REPORT-APP/11-Modules/Geo-Analytics.md`
- Verifikasi: `npx tsc --noEmit` OK. `npm run build` (Webpack) **sukses** — 22/22 halaman, `/geo-analytics` 2.23 kB (MapLibre lazy, tak masuk First Load). Worker/WebGL/blob-worker fungsional.
- Risiko / follow-up penting:
  1. **MapLibre WAJIB Webpack**. Turbopack men-drop worker MapLibre (peta hang). `dev` sekarang Webpack; `dev:turbo` merusak halaman peta. Produksi (`next build`) sudah Webpack — aman.
  2. **Render visual belum diverifikasi**: Claude Browser pane `hidden`/tak compositing → `requestAnimationFrame` tak fire → render loop MapLibre hang. Ini keterbatasan pane, bukan bug kode (SVG jalan karena tak butuh RAF). **User perlu cek visual di browser asli** (tab aktif) — pastikan basemap, fill, garis, dan pin bernomor tampil serta klik memfilter.
  3. Basemap OSM butuh internet keluar; offline → polygon tetap.
- Agent: Claude

### 2026-07-25 - Geo Analytics Fase 1 + penyatuan definisi status

- Area: backend, frontend
- Ringkasan: Implementasi modul Geo Analytics (choropleth per provinsi) sesuai rencana Fase 1. Sekaligus menyatukan definisi status survey/deal/pending/cancelled ke `ConsultationStatusGroups` dan mengisi ulang `config/statuses.php` dengan nama status nyata — memperbaiki bug lama KPI Pending/Cancel di Dashboard yang selalu 0 (kini 195/57).
- File berubah:
  - `E-REPORT-BACKEND_API/app/Support/ConsultationStatusGroups.php` (baru)
  - `E-REPORT-BACKEND_API/app/Support/RegionNameMatcher.php` (baru)
  - `E-REPORT-BACKEND_API/app/Services/Geo/GeoAnalyticsService.php` (baru)
  - `E-REPORT-BACKEND_API/app/Http/Requests/GeoAnalyticsRequest.php` (baru)
  - `E-REPORT-BACKEND_API/app/Http/Controllers/Api/GeoAnalyticsController.php` (baru)
  - `E-REPORT-BACKEND_API/config/statuses.php`, `routes/api.php`
  - `E-REPORT-BACKEND_API/app/Services/DashboardService.php`, `app/Services/Reports/AnalyticsReportService.php` (delegasi status)
  - `E-REPORT-FRONTEND/src/app/geo-analytics/**` (baru), `src/lib/hooks/useGeoAnalytics.ts` (baru)
  - `E-REPORT-FRONTEND/public/geo/idn-provinces.geojson` (baru, 328 KB)
  - `E-REPORT-FRONTEND/src/components/layout/sidebar.tsx`, `bottom-nav.tsx`, `src/lib/auth/roles.ts`
  - `E-REPORT-APP/11-Modules/Geo-Analytics.md`, `Module-Index.md`, `09-Decisions/Decision-Log.md`
- Verifikasi: Analitik tetap 245/15/0 setelah refactor. Geo total 245, 185+60=245 (tidak bocor), 15 provinsi 0 unmatched. Filter Jawa Barat→110, NPP2→156. Klik provinsi memfilter dashboard. `npx tsc --noEmit` OK. Mobile 390px tanpa overflow. 34 provinsi terwarnai, console bersih.
- Peta: awalnya MapLibre GL v6, tapi worker-nya **tak jalan di Next 15.5 + Turbopack** (GeoJSON source tak pernah parse, peta gelap). Diganti ke **SVG choropleth** (ringan, tanpa worker, tetap interaktif); `maplibre-gl` di-uninstall.
- Risiko / follow-up: Peta pakai 34 provinsi (GeoJSON pra-pemekaran Papua 2022) — cukup karena semua data terpakai ada di dalamnya, tapi 4 provinsi Papua baru belum terwakili bila nanti ada datanya. SVG tak punya pan/zoom native. Fitur berbasis titik butuh kolom lat/lng + map picker.
- Agent: Claude

### 2026-07-24 - Rencana modul Geo Analytics (belum dikerjakan)

- Area: docs
- Ringkasan: Menyusun rancangan Fase 1 Geo Analytics setelah memeriksa kesiapan data. Temuan utama: **tidak ada kolom koordinat sama sekali** di 30 tabel, `consultations.address` kosong 100%, `district` 100% placeholder — sehingga peta bermarker (cluster/heatmap/radius/polygon) mustahil tanpa pipeline koordinat baru. Rancangan dialihkan ke choropleth per provinsi memakai `province`/`city` yang sudah terisi. Implementasi ditunda atas permintaan.
- File berubah:
  - `E-REPORT-APP/11-Modules/Geo-Analytics.md` (baru)
  - `E-REPORT-APP/11-Modules/Module-Index.md`
  - `E-REPORT-APP/09-Decisions/Decision-Log.md`
- Verifikasi: tidak ada perubahan kode.
- Risiko / follow-up: Sebelum fitur berbasis titik dikerjakan, perlu kolom `latitude`/`longitude` + map picker di form konsultasi. Celah input `district`/`address` perlu ditutup lebih dulu — data master wilayah sendiri sudah lengkap.
- Agent: Claude

### 2026-07-24 - Filter grup akun di Analitik & Dashboard

- Area: backend, frontend
- Ringkasan: Menambahkan filter Grup Akun (PC / NPP 1 / NPP 2) di Analitik dan Dashboard, di-AND-kan dengan filter Akun yang sudah ada. Daftar grup disajikan endpoint baru `GET /api/v1/master-data/account-groups` yang diturunkan dari `App\Support\AccountGroup`, sehingga grup baru cukup ditambahkan di satu file PHP tanpa mengubah TypeScript.
- File berubah:
  - `E-REPORT-BACKEND_API/app/Services/Reports/AnalyticsReportService.php`
  - `E-REPORT-BACKEND_API/app/Http/Requests/AnalyticsReportRequest.php`
  - `E-REPORT-BACKEND_API/app/Http/Controllers/Api/MasterDataController.php`
  - `E-REPORT-BACKEND_API/routes/api.php`
  - `E-REPORT-FRONTEND/src/lib/hooks/useAnalytics.ts`
  - `E-REPORT-FRONTEND/src/app/analytics/analytics-view.tsx`
  - `E-REPORT-FRONTEND/src/app/dashboard/dashboard-view.tsx`
- Verifikasi: angka dicocokkan dengan query manual — NPP1=89, NPP2=156, PC=0, tanpa filter=245 (Juli 2026). Kombinasi grup+akun kontradiktif (grup PC + akun NPP2) benar menghasilkan 0. Validasi menolak grup asing dengan pesan Indonesia. `npx tsc --noEmit` OK.
- Risiko / follow-up: `AnalyticsReportService` menyimpan grup terpilih di property instance (`$accountGroup`) alih-alih meneruskannya lewat ~20 parameter method. Aman selama service di-resolve baru per request — sudah dipastikan tidak ada binding singleton dan tidak ada Octane. Jangan jadikan service ini singleton.
- Agent: Claude

### 2026-07-24 - Export absensi: warna otomatis dari data + satu tombol export

- Area: backend, frontend
- Ringkasan: Memperbaiki export Excel absensi yang **selalu 500** karena dua bug lama (`AccountGroup` dipakai tanpa `use`, dan eager load `'account:id,name,description'` tidak menyertakan kolom `account_group` yang dibaca setelahnya). Warna sel tidak lagi diambil dari klaim `report_category`, melainkan diturunkan otomatis dari data konsul dengan membandingkan `consultation_date` vs `DATE(created_at)` untuk memisahkan laporan tepat waktu dan susulan. Baris grid diubah dari per admin menjadi per akun. Dua tombol `Export PC`/`Export NPP` diganti satu tombol dengan daftar grup dinamis.
- File berubah:
  - `E-REPORT-BACKEND_API/app/Services/Reports/AdminReportAttendanceExcelExporter.php`
  - `E-REPORT-BACKEND_API/app/Http/Controllers/Api/ReportAttendanceController.php`
  - `E-REPORT-BACKEND_API/app/Support/AccountGroup.php`
  - `E-REPORT-FRONTEND/src/app/report-attendances/report-attendances-view.tsx`
  - `E-REPORT-FRONTEND/src/lib/hooks/useReportAttendances.ts`
- Verifikasi: 9 cabang aturan warna diuji satu per satu dan cocok dengan query manual. Grup baru `NPP3` ditambahkan sementara untuk membuktikan dropdown & export ikut otomatis, lalu dikembalikan. Rentang 8 hari → 8 kolom, lintas bulan → label `dd/mm`, 365 hari → dipotong 92 kolom.
- Risiko / follow-up: Peta `UMBRELLA` di `AccountGroup` diubah (`BISNIS PRIBADI 1/2` menggantikan `BISNIS PRIBADI (NPP 1)`), sehingga subtitle Rekap Jadwal Surveyor ikut berubah — disengaja.
- Agent: Claude

### 2026-07-24 - Analytics: 89 query jadi 31, plus dua bug perhitungan

- Area: backend
- Ringkasan: Lima blok analisa berat yang tidak dipakai di manapun (`comparisonMatrix`, `performanceAnalysis`, `cohortConversion`, `stageVelocity`, `onlyInquiryAnalysis`) dijadikan opt-in lewat filter `include`, default mati. `buildStageVelocity()` sebelumnya menarik **seluruh** tabel `consultation_status_histories` ke memori PHP tanpa filter tanggal — diperbaiki jadi dua tahap (ambil id konsultasi yang punya transisi di periode, baru tarik riwayatnya). `period_days` yang menghasilkan pecahan (`31,9999` untuk Juli) diperbaiki jadi bilangan bulat; nilai ini bocor ke kolom `avg_per_calendar_day` di export Excel.
- File berubah:
  - `E-REPORT-BACKEND_API/app/Services/Reports/AnalyticsReportService.php`
  - `E-REPORT-BACKEND_API/app/Http/Requests/AnalyticsReportRequest.php`
- Verifikasi: 89 → 31 query, 161 ms → 72 ms, seluruh angka UI identik sebelum/sesudah. Kesetaraan `buildStageVelocity` dibuktikan dengan 6 kasus batas (riwayat sebelum/di dalam/sesudah periode) — hasil lama dan baru sama persis. Ambang bucket tren tetap berpindah di 91 hari seperti sebelumnya.
- Risiko / follow-up: Bentuk response tidak berubah (key selalu ada, isinya kosong saat tidak diminta), jadi tidak ada konsumen yang perlu menyesuaikan.
- Agent: Claude

### 2026-07-24 - Filter periode Kustom + perbaikan bug binding SQL analitik

- Area: backend, frontend
- Ringkasan: Menambahkan tipe periode **Kustom** (rentang tanggal bebas) di Analitik dan Dashboard, di samping Mingguan/Bulanan/Tahunan. Sekaligus memperbaiki bug yang membuat **seluruh angka analitik tampil 0**: di `buildDataQuality()` array binding `$pendingLabels` di-merge 7 kali padahal placeholder `NOT IN` hanya ada di 6 tempat, sehingga binding berlebih menggeser posisi dan `whereBetween(consultation_date, …)` menerima string `'belum konfirmasi'` alih-alih tanggal.
- File berubah:
  - `E-REPORT-BACKEND_API/app/Services/Reports/ReportPeriodResolver.php`
  - `E-REPORT-BACKEND_API/app/Services/Reports/AnalyticsReportService.php`
  - `E-REPORT-BACKEND_API/app/Http/Requests/AnalyticsReportRequest.php`
  - `E-REPORT-FRONTEND/src/lib/hooks/useAnalytics.ts`
  - `E-REPORT-FRONTEND/src/app/analytics/analytics-view.tsx`
  - `E-REPORT-FRONTEND/src/app/dashboard/dashboard-view.tsx`
- Verifikasi: Kustom 1–31 Jul menghasilkan angka identik dengan filter Bulanan Juli di seluruh widget. Rentang lintas tahun (20 Des 2025 – 10 Jan 2026) benar label, jendela pembanding, dan jumlah bucket. Rentang > 90 hari otomatis beralih ke bucket bulanan.
- Risiko / follow-up: `cohortConversion` sengaja mengembalikan kosong untuk periode Kustom karena bentuknya terikat satu tahun kalender.
- Agent: Claude

### 2026-07-24 - Dropdown jam survey bisa dicari

- Area: frontend
- Ringkasan: Menambahkan komponen `TimeSearchSelect` untuk dropdown jam survey dengan input pencarian, list pendek/scrollable, warna dark cyan, dan dipakai pada modal `Jadwalkan Survey`, `Reschedule Survey`, serta `Ajukan Survey ke Manager Surveyor`.
- File berubah:
  - `E-REPORT-FRONTEND/src/components/ui/time-search-select.tsx`
  - `E-REPORT-FRONTEND/src/app/surveys/surveys-view.tsx`
  - `E-REPORT-FRONTEND/src/components/consultations/survey-request-card.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Verifikasi: `npx tsc --noEmit` OK, `npm run lint` OK.
- Risiko / follow-up: Perlu cek visual dropdown pada viewport kecil agar popover tidak menutup tombol aksi modal.
- Agent: Codex

### 2026-07-24 - Styling modal jadwal survey

- Area: frontend
- Ringkasan: Menyamakan styling modal `Jadwalkan Survey`, `Reschedule Survey`, dan `Ajukan Survey ke Manager Surveyor` dengan tema dark cyan. Field tanggal/jam pada modal jadwal survey diganti dari native `datetime-local` ke Calendar popover + select jam WIB agar popup abu-abu bawaan browser tidak muncul dan tampilan konsisten.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/surveys/surveys-view.tsx`
  - `E-REPORT-FRONTEND/src/components/consultations/survey-request-card.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Verifikasi: `npx tsc --noEmit` OK, `npm run lint` OK.
- Risiko / follow-up: Perlu cek visual modal di browser untuk memastikan dropdown jam dan calendar tidak terpotong pada viewport kecil.
- Agent: Codex

### 2026-07-24 - Stabilkan KPI absensi saat memuat data

- Area: frontend
- Ringkasan: KPI card absensi di `/report-attendances` sekarang tetap tampil saat data sedang dimuat dengan fallback angka 0, sehingga layout tidak hilang/bergeser ketika response belum tersedia.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/report-attendances/report-attendances-view.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Verifikasi: `npx tsc --noEmit` OK, `npm run lint` OK.
- Risiko / follow-up: Saat loading awal angka sementara tampil 0 sampai data API masuk.
- Agent: Codex

### 2026-07-24 - Redesign absensi report harian

- Area: frontend
- Ringkasan: Merapikan halaman `/report-attendances` menjadi lebih minimalis dan responsif, menambahkan filter status dropdown modern, menampilkan tanggal absen di tabel dan modal moderasi, menampilkan jam laporan di modal, menyesuaikan warna label status, serta mengganti native dropdown agar tidak muncul hover/highlight biru bawaan browser.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/report-attendances/report-attendances-view.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Verifikasi: `npx tsc --noEmit` OK, `npm run lint` OK dengan warning lama di file lain.
- Risiko / follow-up: Perlu cek visual di browser pada desktop dan mobile untuk memastikan komposisi KPI/filter sesuai preferensi desain.
- Agent: Codex

### 2026-07-23 - Urutan laporan Excel (terlama dulu) vs halaman Konsultasi (terbaru dulu)

- Area: backend
- Ringkasan: Laporan Excel/PDF kini disusun dari konsultasi **terlama ke terbaru**, sedangkan halaman Konsultasi tetap menaruh yang **terbaru di atas**. Dua kebutuhan yang memang berbeda: layar dipakai memantau lead yang baru masuk, laporan dibaca untuk menelusuri perkembangan berurutan waktu.
- `LeadsReportService::buildDetailRows()`: `orderByDesc('consultation_date')` jadi `orderBy('consultation_date')`. Pengurut kedua diganti dari `updated_at` ke `id` supaya baris dalam satu hari tidak berpindah-pindah tiap ada lead yang disunting. Sumber ini dipakai bersama oleh export Excel dan PDF, jadi keduanya ikut berubah.
- **Dua bug urutan yang terungkap saat memeriksanya**, keduanya baru berbahaya setelah import massal:
  1. `ConsultationController::index()` **tidak punya pemecah seri**. Import memberi ratusan lead `updated_at` yang nyaris identik, dan tanpa kolom kedua MySQL bebas menentukan urutan - antar halaman baris bisa terulang atau justru hilang. Ditambahkan `orderByDesc('id')` sebagai pengurut terakhir.
  2. **Default sort halaman adalah `updated_at`**, sehingga setelah import urutannya mencerminkan waktu import, bukan lead mana yang konsultasinya paling baru - lead tertanggal 22 Juli bisa muncul di bawah lead 7 Juli hanya karena urutan file impornya. Default diganti ke `consultation_date`. Parameter `sort`/`dir` dari klien tetap dihormati.
- File berubah: `app/Services/Reports/LeadsReportService.php`, `app/Http/Controllers/Api/ConsultationController.php`
- Verifikasi: 268 baris laporan urut menaik `2026-04-23` sampai `2026-07-23`; 3 halaman pertama daftar konsultasi berisi 30 id unik tanpa tumpang tindih, urutan stabil saat permintaan diulang, tanggal konsultasi menurun; `sort=asc` masih dihormati; `/consultations` HTTP 200.
- Catatan: `consultation_date` adalah kolom `date` tapi diserialkan API sebagai instant UTC (`2026-07-21T17:00:00Z` untuk tanggal DB `2026-07-22`). Saat ini tampil benar di UI karena browser mengembalikannya ke Asia/Jakarta, tapi rapuh untuk pembaca di zona waktu lain. Belum disentuh - di luar lingkup permintaan.
- Agent: Claude

### 2026-07-23 - Migrasi 268 lead lama dari 5 workbook ke sistem baru

- Area: backend / database / docs
- Ringkasan: Data lead lama dipindahkan ke sistem baru lewat endpoint import yang sudah ada, bukan importer khusus - seluruh logika pencocokan akun, normalisasi wilayah, penomoran ID, dan anti-duplikat sudah tersedia dan teruji di `ConsultationImportService`. Lima workbook (304 baris mentah) dikonversi jadi CSV template 14 kolom lebih dulu, lalu diunggah berurutan sebagai super admin. Hasil: **268 lead, 36 baris terdeteksi duplikat dan berperan sebagai update, 0 error**.

| Berkas | Baris | Baru | Update |
|---|---|---|---|
| export aplikasi (`data-leads-semua-akun-juli-2026 (11)`) | 193 | 193 | 0 |
| `LAPORAN PUTRA.INTERIOR 2 (1)` | 26 | 26 | 0 |
| `Kategori Master lap (8)` (HOME INTERIOR BANDUNG) | 27 | 27 | 0 |
| `LAPORAN RAYSA 23 JULI` | 19 | 15 | 4 |
| `DOC-20260722-WA0005` (HEYA INTERIOR) | 39 | 7 | 32 |

- **ID lead dibangkitkan aplikasi, bukan dibawa dari file.** ID lama memakai nomor akun lama (`39.2607-001`, `01.2604-001`) yang tidak berhubungan dengan id akun baru, dan formatnya pun beda (pemisah `-`, 3 digit). `normalizeImportedConsultationId()` memang sudah menolak bentuk itu; kolom ID di CSV dikosongkan supaya jelas niatnya. Diverifikasi: 268 ID berformat `NN.YYMM.NNNN`, prefiks cocok id akunnya, tidak ada yang kembar.
- **Anti-duplikat bersandar pada `Consultation::findDuplicateLead()`** yang mencocokkan nomor telepon dalam satu akun. `GROUP BY account_id, phone HAVING COUNT(*) > 1` kosong.
- **File `DOC-20260722-WA0005.xlsx` tidak punya nama akun sama sekali** - hanya "ADMIN : ragil sacikal", sementara RAGIL memegang 3 akun. Ditentukan sebagai HEYA INTERIOR lewat pencocokan nomor WA: 32 dari 39 baris memang sudah ada di HEYA INTERIOR. Ke-32 itu jadi update, 7 sisanya lead baru.
- Keputusan lain: `NACIRA` diganti nama jadi `NACIRA STUDIO` (bukan akun baru, supaya admin `omen@nacira.com` ikut terbawa dan leadnya tidak terpecah); `PALEM INTERIOR` dan `AKBAR INTERIOR` dibuat sebagai akun baru tanpa admin; pemetaan status `Penjadwalan`/`Masuk Survey` ke `Request Survey`, `Menunggu Bangunan` ke `Nunggu Bangunan`, sel kosong ke `Masih konsultasi`; status baru `Perbandingan Harga` ditambahkan (15 lead).

**Empat perbaikan kode yang lahir dari migrasi ini** (berlaku untuk semua import berikutnya, bukan sekali pakai):

0. **`parseDate()` menolak format tanggal Excel yang paling lazim** - ini bug terburuk dari semuanya dan baru ketahuan setelah import pertama: **seluruh 268 lead memakai tanggal hari ini**, tanggal konsultasi aslinya hilang. Penyebabnya `parseDate()` hanya menerima `d/m/Y`, `d-m-Y`, `Y-m-d`, `m/d/Y` dengan kecocokan bolak-balik persis, padahal sel tanggal Excel keluar sebagai `2026-07-16 00:00:00` (berjam) atau `16/07/26` (tahun dua digit). Keduanya ditolak, lalu `?? now()->toDateString()` diam-diam menutupinya - tidak ada error, tidak ada peringatan. Daftar format kini ditambah `Y-m-d H:i:s`, `Y-m-d H:i`, `Y/m/d`, `d/m/y`, `d-m-y`, dengan format bertahun 4 digit didahulukan supaya `01/02/2026` tidak keburu tertangkap pola dua digit. Konverter juga menormalkan sel tanggal jadi `YYYY-MM-DD` di hulu dan melaporkan sel yang gagal dibaca. Import diulang: 0 lead baru, 304 baris jadi update - **tanggal seluruh 268 lead diperbaiki di tempat tanpa satu pun duplikat**, membuktikan anti-duplikatnya bekerja. Sebaran akhir 34 hari berbeda, 2026-04-23 sampai 2026-07-23.


1. **Default status import salah** - `resolveDefaults()` memakai `orderBy('sort_order')->first()` yang mengembalikan **`Selesai Survey`** (`sort_order = 0`). Baris yang statusnya tak dikenali akan mengaku selesai disurvei tanpa punya record survey, sekaligus terkunci `surveyStatusConflict()` sehingga statusnya tidak bisa digeser lagi. Sekarang dicari eksplisit ke `Masih konsultasi`, pola yang sudah dipakai `$defaultCategory` tepat di bawahnya. `sort_order` `Selesai Survey` dan `Sedang Survey` yang sama-sama 0 juga dirapikan ke belakang.
2. **Kategori kebutuhan jamak** - sel seperti `"Kitchenset, Backdrop TV"` (format yang dipakai export aplikasi sendiri) dicocokkan utuh, tidak pernah ketemu, lalu jatuh ke kategori default. Helper baru `resolveNeedsCategoryIds()` memecah per koma dan `flushChunk()` men-`sync()` seluruh daftarnya ke pivot. 13 lead kini berkategori jamak.
3. **Alias wilayah** - `PROVINCE_ALIASES` ditambah singkatan lazim (`jabar`, `jateng`, `jatim`, `sumut`, dst.) dan konstanta baru `CITY_ALIASES` menampung salah eja (`tanggerang`, `tasik`, `Kota B A T A M`, `Kab. Mahakam Hulu` yang nama resminya `Kab. Mahakam Ulu`). `CITY_ALIASES` diperiksa **paling akhir** di `canonicalCity()` supaya tidak pernah mengalahkan nama asli dataset. Nilai `kbb` yang tertulis di kolom Provinsi ditangani konverter: nilai Provinsi yang ternyata nama kota dipindah ke kolom Kota.
- File berubah: `app/Services/ConsultationImportService.php`, `app/Support/WilayahNormalizer.php`
- Verifikasi tanggal: tanggal tiap lead dibandingkan **satu per satu** dengan file sumber lewat kunci akun+telepon - **268 cocok, 0 beda, 0 tidak ketemu**. Sebelumnya hanya sebarannya yang dilihat, dan itulah sebabnya bug tanggal sempat lolos.
- Verifikasi: 12 dari 13 pemeriksaan lolos langsung. Satu "gagal" ternyata ekspektasi saya yang salah - saya menghitung 21 lead Request Survey dari baris mentah, padahal baris yang tumpang tindih adalah lead yang sama. Hitung ulang dedup dari sisi CSV menghasilkan 268 lead dengan **sebaran status persis sama** dengan isi database (Hanya Tanya Tanya 134, Masih konsultasi 37, Kendala Anggaran 30, Tidak Ada Respon 27, Request Survey 15, Perbandingan Harga 15, Nunggu Bangunan 9, Selesai/Deal 1). Selain itu: seluruh nama provinsi dan kota lolos daftar kanonik `Wilayah`; tidak ada lead berstatus Sedang/Selesai Survey; semua lead punya akun, telepon, dan minimal satu kategori. `/consultations`, `/accounts`, `/dashboard` balas HTTP 200.
- Backup `mysqldump` diambil sebelum import ke scratchpad sesi.
- Risiko / follow-up:
  - **`PALEM INTERIOR` (2 lead) dan `AKBAR INTERIOR` (1 lead) belum punya admin** dan `account_group`-nya masih default `PC` - keduanya tidak akan muncul di rekap NPP sampai dilengkapi.
  - **7 baris dari file HEYA yang tidak cocok** dengan lead mana pun bisa saja sebenarnya milik akun RAGIL yang lain (`KITCHENSET BANDUNG BARAT` / `INTERIOR MODERN`). Perlu ditinjau di UI.
  - Baris yang tumpang tindih **ditimpa** oleh file yang diimpor belakangan. Urutan sudah diatur (export aplikasi lebih dulu, laporan menyusul) tapi bila ada laporan yang lebih tua dari export, kolomnya ikut mundur.
- Agent: Claude

### 2026-07-23 - Seeding 18 akun + 18 admin dari daftar NPP 1 / NPP 2

- Area: database
- Ringkasan: 18 akun beserta admin pemegangnya dibuat di DB lokal dari daftar yang diberikan. Sebaran: NPP 1 = 6 akun, NPP 2 = 12 akun. Semua admin berperan `admin`, terhubung ke account_id masing-masing, password default sama.
- **Format email**: `namaAdmin@namaAkun.com`, huruf kecil, spasi dan tanda baca dibuang. Contoh: `DIAN GARUT` di `HOME PUTRA INTERIOR` menjadi `diangarut@homeputrainterior.com`.
- **Satu orang bisa punya lebih dari satu akun user.** Kolom `users.account_id` tunggal, jadi RAGIL (3 akun), OMEN (4), YANWAR (2), YANWA (2), dan DENA (2) masing-masing mendapat user terpisah per akun. Emailnya otomatis berbeda karena memuat nama akun.
- **Baris "MEDIAN INTERIOR" pada NPP 1 tidak dibuat** - dikonfirmasi sebagai duplikat dari entri NPP 2 (admin ANTO). Nama akun divalidasi unik oleh `AccountRequest`, jadi dua entri bernama sama memang tidak mungkin.
- **Dua kolom grup diisi sekaligus.** `accounts.account_group` diisi `NPP1`/`NPP2` - taksonomi resmi yang dipakai rekap surveyor dan export absensi lewat `App\Support\AccountGroup`. `accounts.description` diisi label `NPP 1`/`NPP 2` karena kolom itulah yang tampil sebagai "Grup" di halaman Akun. Perlu diketahui: **dua kolom ini menyimpan hal yang sama dengan format berbeda**, dan form akun di UI hanya bisa mengubah `description`, tidak `account_group`.
- `account_group` dan `created_by` tidak fillable di model `Account`, begitu pula `role` di `User`; pengisian memakai `forceFill()`.
- Verifikasi: `Hash::check()` password default lolos untuk 18 admin; 0 admin tanpa `account_id`; 18 email seluruhnya unik; 0 akun tanpa admin; super admin tetap utuh.
- Risiko / follow-up: password default identik untuk 18 akun - wajib diganti sebelum dipakai di produksi. `account_group` belum bisa diubah lewat UI, jadi akun baru yang dibuat dari halaman Akun akan memakai default kolom `PC` dan tidak akan ikut terhitung di rekap NPP.
- Agent: Claude

### 2026-07-23 - Redesain UI/UX halaman Akun

- Area: frontend
- Ringkasan: Halaman Akun dirapikan setelah data dikosongkan - kondisi kosongnya memperlihatkan beberapa kelemahan yang selama ini tertutup data.
  1. **Empty state punya dua bentuk.** Sebelumnya hanya satu kalimat abu-abu "Tidak ada akun terdaftar." di tengah ruang kosong. Sekarang dibedakan: belum ada data sama sekali (ikon gedung + penjelasan peran akun + tombol "Buat Akun Pertama") versus filter tidak menemukan hasil (ikon `SearchX` + tombol "Bersihkan filter"). Dua kondisi ini butuh jalan keluar yang berbeda.
  2. **Bar paginasi tidak lagi muncul saat kosong.** Sebelumnya tetap tampil berisi "Menampilkan 0 dari 0 akun. Halaman 1 dari 1" dengan kedua tombol mati. Sekarang bar hanya muncul bila `meta.total > 0`, dan tombol navigasinya disembunyikan bila cuma satu halaman.
  3. **Loading pakai skeleton berbentuk tabel**, bukan spinner mengambang - tinggi halaman tidak melompat saat data masuk.
  4. **Layout kartu untuk layar sempit.** Tabel `min-w-[980px]` memaksa geser horizontal di ponsel. Di bawah `lg` kini tampil daftar kartu; tabel hanya untuk `lg` ke atas.
  5. **Toolbar dibungkus kartu** supaya tidak mengambang lepas dari daftar yang dikendalikannya, dan **jumlah akun dinaikkan ke header** sebagai badge sehingga skala data terbaca tanpa turun ke bawah tabel. Ikon judul dipindah ke kotak beraksen.
  6. Markup berulang diangkat jadi komponen kecil di file yang sama - `AccountLogo`, `GroupBadge`, `AdminBadges`, `ProgressBar`, dan helper `accountProgress()` - dipakai bersama tabel dan kartu mobile supaya keduanya tidak bisa berbeda tampilan.
  7. **Permukaan dibuat solid.** Semua kartu, toolbar, empty state, header tabel, dan input sebelumnya memakai alpha (`bg-card/40..60`, `bg-muted/20..60`, `dark:bg-zinc-900/30..60`, `dark:bg-zinc-950/40`) sehingga grid latar halaman tembus dan panelnya tidak terbaca sebagai bidang. Semuanya diganti warna penuh. `backdrop-blur-sm` pada kontainer tabel ikut dihapus - sejalan dengan konvensi performa proyek yang melarang blur pada kartu. Alpha yang tersisa hanya untuk lapisan di atas permukaan solid: zebra baris, hover, dan badge berwarna.
- File berubah: `src/app/accounts/accounts-view.tsx`
- Verifikasi: `npx tsc --noEmit` bersih; `/accounts` HTTP 200 tanpa build error.
- Risiko / follow-up: baru diperiksa pada kondisi data kosong (DB baru saja dibersihkan). Tampilan tabel dan kartu mobile saat berisi data belum dilihat langsung di browser.
- Agent: Claude

### 2026-07-23 - Istilah "Kategori" pada halaman Akun jadi "Grup"

- Area: frontend
- Ringkasan: Di halaman Akun, `accounts.description` dilabeli "Kategori" - membingungkan karena "kategori" di aplikasi ini sudah dipakai untuk kategori kebutuhan lead. Seluruh label UI-nya diganti "Grup": label form `Grup / Tagline`, placeholder filter dan opsi `Semua Grup`, judul kolom tabel `Grup`, tooltip `Bersihkan filter grup`, dan placeholder pencarian `Cari nama akun, grup, atau admin...`.
- Perubahan label saja. Kolom database (`accounts.description`), endpoint `/accounts/categories`, dan nama hook `useAccountCategories` **tidak** diubah - komentar hook diberi catatan supaya beda istilah ini tidak membingungkan nanti.
- File berubah: `src/app/accounts/accounts-view.tsx`, `src/lib/hooks/useAccounts.ts`
- Verifikasi: `npx tsc --noEmit` bersih; `/accounts` HTTP 200 tanpa build error.
- Agent: Claude

### 2026-07-23 - Pembersihan user + account, sisakan Super Admin (DB lokal)

- Area: database
- Ringkasan: Lanjutan pembersihan sebelumnya. Di DB lokal `e_report`, 35 dari 36 user dihapus - hanya `#1 Super Admin (superadmin@npp.com)` yang disisakan, dipilih lewat `role = super_admin` bukan ID hardcode. Seluruh 33 `accounts` dikosongkan; aman karena Super Admin memang tidak terikat account manapun (`account_id` NULL, dan tetap di-set NULL sebelum truncate sebagai pengaman). **Tanpa backup**, dipilih secara sadar.
- Ikut dikosongkan: `bug_reports` 2 (wajib - punya FK ke users), `audit_logs` 1053, `login_attempts` 104, `personal_access_tokens` 2, `account_user`, `sessions`, `push_subscriptions`. Pilihan "hapus semua jejak" diambil supaya tidak ada baris yatim yang `user_id`-nya menunjuk ID tidak ada.
- **Super Admin harus login ulang** - `personal_access_tokens` ikut terhapus.
- Master data tetap utuh: `status_categories` 8, `needs_categories` 38, `survey_statuses` 5. Cache aplikasi dibersihkan.
- Verifikasi: hitung sebelum/sesudah tiap tabel, plus daftar user tersisa dicetak ulang.
- Agent: Claude

### 2026-07-23 - Pembersihan data konsultasi + survey (DB lokal)

- Area: database
- Ringkasan: Atas permintaan, seluruh data transaksi konsultasi dan survey di DB lokal `e_report` (127.0.0.1) dikosongkan lewat `truncate` dengan `FOREIGN_KEY_CHECKS=0`. **Tanpa backup** - dipilih secara sadar. Yang dihapus: `consultations` 14, `surveys` 8, `survey_activity_logs` 17, `survey_notifications` 11, `survey_status_histories` 7, `survey_reschedules` 1, `consultation_needs_category` 30, `consultation_status_histories` 14, `consultation_imports` 9, `consultation_sequences` 397. Master data tidak disentuh dan diverifikasi utuh setelahnya: `accounts` 33, `users` 36, `status_categories` 8, `needs_categories` 38, `survey_statuses` 5. Cache aplikasi ikut dibersihkan.
- Catatan: `consultation_sequences` sengaja ikut direset, jadi ID lead berikutnya mulai dari 0001 lagi. Kalau ada dokumen lama yang memuat ID lama, nomornya akan terpakai ulang.
- Tidak disentuh: `audit_logs` (1053), `login_attempts` (104), `bug_reports` (2), `personal_access_tokens` (2) - bukan data konsultasi.
- Verifikasi: hitung ulang setelah truncate, ketiga tabel inti nol.
- Agent: Claude

### 2026-07-23 - Simpan edit lead kembali ke daftar

- Area: frontend
- Ringkasan: Semua jalan keluar halaman edit lead kini menuju `/consultations` (daftar), bukan halaman detail lead: redirect setelah simpan, tombol panah kembali di header, dan tombol Batal.
- File berubah: `src/app/consultations/[id]/edit/page.tsx`
- Verifikasi: `npx tsc --noEmit` bersih.
- Agent: Claude

### 2026-07-23 - Form pengajuan survey: link Google Maps + tanggal/jam format Indonesia

- Area: backend / frontend / docs
- Ringkasan:
  1. **Field "Kebutuhan yang disurvei" diganti "Link Google Maps".** Kebutuhan sudah tercatat di lead (kategori kebutuhan + detail produk), jadi mengetik ulang di form pengajuan cuma duplikasi. Yang benar-benar dibutuhkan surveyor adalah titik lokasinya. Kolom `surveys.google_maps_url` sebenarnya **sudah ada sejak migrasi awal dan sudah fillable**, hanya tidak pernah divalidasi maupun diisi oleh `store()` - sekarang keduanya dipasang (`nullable|url|max:2048` dengan pesan error Bahasa Indonesia). Link yang tersimpan tampil di kartu Status Survey sebagai tautan "Buka lokasi di Google Maps".
  2. **Tanggal pakai kalender Indonesia.** `<input type="date">` mengikuti locale sistem, jadi di mesin ber-locale Inggris tampil `07/23/2026`. Diganti Popover + komponen `Calendar` yang locale bawaannya sudah `date-fns/locale/id`, ditampilkan sebagai `23 Juli 2026`. Tanggal lampau dinonaktifkan lewat matcher `{ before: hari ini }`, menggantikan atribut `min`.
  3. **Jam pakai format 24 jam.** `<input type="time">` memunculkan pemilih AM/PM. Diganti `CustomSelect` berisi 48 pilihan tiap 30 menit berlabel `HH:mm WIB`, plus opsi "Belum ditentukan". Aturan backend `date_format:H:i` tidak berubah - diuji ulang: `05:09 PM` ditolak, `17:30` diterima.
  4. Skema URL dilengkapi di sisi frontend (`normalizeMapsUrl`) karena Google Maps kerap disalin sebagai `maps.app.goo.gl/...` tanpa `https://`, sementara aturan `url` menolaknya.
- File berubah: `app/Http/Controllers/Api/SurveyController.php`, `src/components/consultations/survey-request-card.tsx`, `src/lib/hooks/useSurveys.ts`
- Verifikasi: `php -l` bersih; `npx tsc --noEmit` bersih; `/consultations` HTTP 200 tanpa build error; 8 assertion validator lolos (4 bentuk URL + null, 2 format jam, 1 cek fillable).
- Risiko / follow-up: `requested_item` tidak lagi diisi dari UI tapi kolomnya tetap ada dan masih ikut dicari di `SurveyController::index()` - data lama tetap terbaca, data baru akan selalu NULL. Kalau memang tidak dipakai lagi, hapus di migrasi terpisah.
- Agent: Claude

### 2026-07-23 - Pengajuan survey dari halaman create + 3 bug form konsultasi

- Area: frontend / docs
- Ringkasan:
  1. **Lead dibuat langsung berstatus Request Survey kini diteruskan.** Sebelumnya alur pengajuan hanya punya satu pintu masuk (ubah status di halaman detail); lead yang lahir dengan status Request Survey dari halaman create langsung menggantung. Setelah tersimpan, admin diarahkan ke `/consultations/{id}?prompt_survey=1`; halaman detail membaca parameter itu, menaikkan `surveyPromptSignal`, lalu membersihkan query lewat `history.replaceState` supaya refresh tidak membuka modal lagi. Komponen `survey-request-card.tsx` dipakai ulang - tidak ada form kedua yang perlu dijaga.
  2. **Petunjuk di halaman create.** Form pengajuan butuh ID lead sehingga baru bisa terbuka setelah simpan. Saat status Request Survey dipilih muncul strip penjelas, supaya admin tidak mengira tahap pengajuannya hilang.
  3. **Bug: nomor telepon tidak bisa dihapus.** `formatPhoneInput` memformat ulang tiap ketikan. Untuk input parsial seperti `0323`, `AsYouType` menghasilkan `(0323)`; menekan backspace membuang `)` saja, digitnya utuh, jadi formatter memasangnya kembali dan field terkunci selamanya. Sekarang fungsi menerima nilai sebelumnya: bila panjang berkurang tapi susunan digit sama berarti yang terhapus separator, sehingga **digit terakhir** yang dibuang. Menghapus `+` dikecualikan agar tidak ikut memakan digit. Diuji: `(0323)`, `0831 3755 67834`, `+62 831 3755 6783`, dan `+1 213 373 4253` semuanya habis dikosongkan lewat backspace berulang - sebelumnya kasus pertama tidak pernah berhenti.
  4. **Bug: input tanggal tanpa border.** Aturan `.consultation-page :is(...)` hanya menyetel `border-color`, mengandalkan tiap kontrol membawa `border` sendiri. Trigger popover tanggal berupa `<button>` yang `border-width`-nya 0, jadi garisnya tak pernah tampak. `border-width: 1px` + `border-style: solid` ditambahkan ke aturan itu.
  5. Chip "Pilih minimal 1" di `consultation-form-actions.tsx` diganti menjadi informatif (`Kosong -> Tidak konfirmasi`) - sisa dari perubahan yang membuat kategori kebutuhan opsional.
- File berubah: `src/app/consultations/create/page.tsx`, `src/app/consultations/[id]/page.tsx`, `src/app/consultations/[id]/edit/page.tsx`, `src/lib/utils.ts`, `src/app/globals.css`, `src/components/consultations/consultation-form-actions.tsx`, `E-REPORT-APP/11-Modules/Survey.md`
- Verifikasi: `npx tsc --noEmit` bersih; `/consultations` dan `/consultations/create` balas HTTP 200 tanpa build error; simulasi backspace berulang pada 4 format nomor semuanya berakhir kosong.
- Risiko / follow-up: perubahan `border-width` berlaku untuk semua kontrol di `.consultation-page` - kontrol yang sudah membawa `border` sendiri hasilnya sama, tapi kalau ada yang sengaja tanpa garis akan ikut bergaris. Belum ditelusuri satu per satu lewat browser.
- Agent: Claude

### 2026-07-23 - Alur pengajuan survey oleh admin + penutupan celah flow survey

- Area: backend / frontend / database / docs
- Ringkasan: Admin sebelumnya **tidak punya cara sama sekali** mengajukan survey - endpoint dan hook `useRequestSurvey` ada tapi tidak pernah dipanggil, dan halaman detail lead nol UI survey. Flow survey dipetakan ulang lebih dulu; pemetaan menemukan 9 celah, 4 di antaranya struktural. Dikerjakan dalam 6 fase.
  1. **Celah struktural ditutup.** Migrasi `active_key` + unique index mencegah satu lead punya lebih dari satu survey aktif (**3 lead sudah terlanjur punya duplikat** dan dirapikan otomatis). `store()` mendapat kembali guard status lead = Request Survey (guard ini hilang saat repo di-clone) dan cek-lalu-tulisnya dibungkus transaksi + `lockForUpdate`. Endpoint baru `PATCH /surveys/{survey}/unassign` supaya manager bisa melepas penugasan tanpa membatalkan survey. `ConsultationController::updateStatus()` menolak status yang bertentangan dengan survey berjalan.
  2. **Relasi & query pendukung**: `Consultation::activeSurvey()` dan `scopeNeedsSurveyRequest()`, eager load di `index()`/`show()`, filter `?pending_survey=1`.
  3. **Web Push untuk peristiwa survey.** Penerima diambil dari `SurveyRealtimeUpdated::recipientsFor()` yang kini dipakai bersama notifikasi in-app - satu sumber, jadi push dan in-app tidak pernah menyasar orang berbeda. Semua dispatch disatukan lewat `SurveyController::notifySurvey()`.
  4. **Penanda lead belum diajukan**: `pending_survey_requests` menumpang payload `/notifications` yang sudah dipolling.
  5. **Modal pengajuan otomatis**: komponen baru `survey-request-card.tsx` - modal terbuka sendiri saat status diubah ke Request Survey pada lead yang belum punya survey, plus kartu Status Survey permanen sebagai jaring pengaman.
  6. **Badge + kesegaran**: tombol "Belum diajukan" di toolbar consultations, `refetchOnWindowFocus` pada hook notifikasi.
- **Penyimpangan yang disengaja dari permintaan awal.** Permintaan menyebut "notifikasi realtime". WebSocket Reverb **tidak** dipakai, dan interval polling **tidak** dipercepat. `useNotifications.ts` memuat catatan insiden: polling 3 detik menghasilkan ~1200 koneksi/jam per user dan menjatuhkan dashboard (MySQL shared-hosting dibatasi 500 koneksi/jam). Jalur seketika memakai Web Push yang sudah terpasang - sampai ke HP walau app ditutup. Biaya polling justru **diturunkan**: `SurveyNotification::count()` yang tadinya jalan mentah tiap request kini ikut cache 2 menit di `NotificationSummaryService`.
- File berubah:
  - Baru: `database/migrations/2026_07_23_000002_add_active_key_to_surveys_table.php`, `E-REPORT-FRONTEND/src/components/consultations/survey-request-card.tsx`
  - Backend: `app/Models/Survey.php` (hook `booted()` menjaga `active_key`), `app/Models/Consultation.php`, `app/Http/Controllers/Api/{Survey,Consultation,Notification}Controller.php`, `app/Events/SurveyRealtimeUpdated.php`, `app/Services/{NotificationSummaryService,ConsultationImportService}.php`, `routes/api.php`
  - Frontend: `src/lib/hooks/{useSurveys,useNotifications}.ts`, `src/types/index.ts`, `src/app/consultations/[id]/page.tsx`, `src/app/consultations/consultations-view.tsx`
  - Docs: `E-REPORT-APP/11-Modules/Survey.md`, `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Verifikasi: `php -l` bersih; `npx tsc --noEmit` bersih; **110 assertion di 10 skrip uji lolos semua**. Uji end-to-end memastikan pengajuan dari status salah ditolak 422, pengajuan sah tersimpan lengkap dengan tanggal/jam, seluruh manager dinotifikasi, pengajuan ganda ditolak, dan badge pending berkurang. Migrasi diuji lewat rollback + migrate ulang.
- Risiko / follow-up:
  - **Belum diuji lewat UI browser.** Perlu dicek: modal terbuka otomatis saat status diubah, modal **tidak** terbuka pada lead yang sudah punya survey, badge "Belum diajukan" muncul dan filternya bekerja.
  - **Guard baru bisa menolak kebiasaan lama.** Bila tim terbiasa menggeser status lead manual ke "Sedang/Selesai Survey" tanpa lewat aksi survey, alur itu kini gagal 422. Beri tahu tim sebelum rilis.
  - **Web Push butuh izin browser**; di iOS hanya jalan bila PWA di-install ke home screen. Manager yang menolak izin tetap bergantung polling 30 detik.
  - Celah "`submitResult` melompati `in_progress`" **sengaja tidak ditutup** - kemungkinan jalan pintas yang memang dipakai surveyor. Analitik kecepatan tahap perlu dibaca dengan pemahaman itu.
  - Notifikasi `request_created` tetap unread setelah survey dibatalkan (celah 8) - belum ditangani.

### 2026-07-23 - Sinkronisasi penamaan wilayah, kolom Kecamatan, dan nomor telepon global

- Area: backend / frontend / database / docs
- Ringkasan: Menyelaraskan penamaan wilayah aplikasi dengan master Excel tim, menambah kolom Kecamatan pada template import, mengubah nomor telepon ke E.164 lintas negara, serta memperbaiki beberapa bug yang ditemukan sepanjang pengerjaan. Dikerjakan dalam 9 fase.
  1. **Dataset wilayah dari master Excel.** Generator baru `tools/generate_wilayah.py` membaca `db_wilayah.xlsx` dan menulis `config/wilayah.php` + dua file di `resources/data/`. Hasil: **38 provinsi, 514 kota, 7283 pasangan kecamatan** dengan format `Kab. X` / `Jakarta` / `Yogyakarta` (sebelumnya 34 provinsi gaya `Kabupaten X` / `DKI Jakarta` dari emsifa). Spasi ganda pada sumber (`Kab.  Kepulauan Seribu`) dirapikan. `app:sync-wilayah` ditandai usang dan menolak jalan tanpa `--force`.
  2. **`WilayahNormalizer`** jadi satu sumber kebenaran ejaan wilayah, dipakai bersama form, importer, dan exporter. Tabel alias (`KBB`, `Jaksel`, dst) dipindah ke sini dari `LeadsExcelExporter`.
  3. **Kolom Kecamatan di template** - 14 kolom jadi 15, disisipkan di posisi J. Dropdown Kota dan Kecamatan kini bertingkat memakai `OFFSET/MATCH/COUNTIF`. Kecamatan sengaja **tidak mengikat** (`showErrorMessage=0`) sehingga boleh diketik manual; ejaannya dirapikan saat import.
  4. **Import mengikuti kolom baru** dan tetap menerima file lama 14 kolom lewat `hasDistrictColumn()`.
  5. **Command `consultations:backfill`** untuk menyelaraskan data lama. Mode bawaan pratinjau; menulis butuh `--apply`.
  6. **Label placeholder dipisah**: wilayah kosong jadi `Belum konfirmasi`, kategori kebutuhan kosong jadi `Tidak konfirmasi` (baris `needs_categories` di-rename lewat migrasi, id dipertahankan agar relasi pivot aman).
  7. **Master data tersinkron.** `NeedsCategory::scopeForConsultationOptions()` tidak lagi memfilter `whereIn('name', DISPLAY_NAMES)` - kategori yang ditambahkan lewat Master Data kini muncul di form dan template. Header `Cache-Control` endpoint master data diubah ke `private, no-cache, must-revalidate`.
  8. **Nomor telepon E.164** lewat `giggsey/libphonenumber-for-php` (paket baru). Validasi panjang mengikuti aturan per negara, dan kunci duplikat memakai E.164 sehingga `0831...` dan `+62 831...` terbaca sama.
  9. **UI kategori jamak** - detail konsumen menampilkan daftar badge, tabel leads menampilkan kategori pertama + `+N` dengan tooltip berisi semua nama.
- Bug yang ditemukan dan diperbaiki sepanjang pengerjaan:
  - `ConsultationImportService` menulis kolom **Domisili** ke `address`; 5 baris DB beralamat `"Dalam Kota"`/`"Luar Kota"`.
  - `district` di-hardcode `null` pada import, sehingga tidak pernah terisi.
  - Sanitasi anti-CSV-injection membuang `+` di awal nomor, membuat `+60123456789` tersimpan sebagai `+6260123456789`. Ditambah `sanitizePhoneCell()`.
  - `WilayahController::matchCity()` memakai `str_contains`, sehingga kota `Bandung` ikut menarik kecamatan milik `Bandung Barat`.
  - Migrasi rename kategori tidak membersihkan cache master data, sehingga API tetap menyajikan nama lama sampai 6 jam.
  - Pencarian lead tidak mencakup nama akun.
- File berubah:
  - Baru: `tools/generate_wilayah.py`, `app/Support/WilayahNormalizer.php`, `app/Support/PhoneNumber.php`, `app/Console/Commands/BackfillConsultations.php`, `database/migrations/2026_07_23_000001_rename_pending_needs_category_label.php`
  - Backend: `app/Support/PendingConfirmation.php`, `app/Models/NeedsCategory.php`, `app/Models/Consultation.php`, `app/Http/Requests/ConsultationRequest.php`, `app/Http/Controllers/Api/{Consultation,Wilayah,MasterData}Controller.php`, `app/Services/ConsultationImportService.php`, `app/Services/Reports/{LeadsExcelExporter,LeadsReportService,AnalyticsReportService,SpreadsheetXmlToXlsxConverter}.php`, `app/Console/Commands/SyncWilayah.php`, `config/wilayah.php`, `resources/data/wilayah_kota.php`, `resources/data/wilayah_kecamatan.php`, `composer.json`
  - Frontend: `src/lib/utils.ts`, `src/app/consultations/consultations-view.tsx`, `src/app/consultations/create/page.tsx`, `src/app/consultations/[id]/page.tsx`, `src/app/consultations/[id]/edit/page.tsx`, `src/components/consultations/panduan-pengisian.tsx`
  - Docs: `E-REPORT-APP/11-Modules/Wilayah.md`, `E-REPORT-APP/11-Modules/Consultations.md`, `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Verifikasi: `php -l` pada 20 file backend bersih; `npx tsc --noEmit` bersih; 84 assertion di 7 skrip uji lolos (normalizer, telepon, master data, matchCity, import baru, import lama, backfill). Uji API lewat HTTP: `/wilayah/provinces` mengembalikan 38 nama gaya Excel, `/wilayah/districts?city=Kota Bandung` tidak lagi bocor ke `Lembang`, template unduhan berisi 15 kolom dengan Kecamatan non-mengikat. `consultations:backfill --apply` dijalankan pada 13 lead dan terbukti idempoten.
- Risiko / follow-up:
  - **Belum diuji lewat UI browser.** Perlu dicek: form create tanpa kategori, dropdown bertingkat Provinsi-Kota-Kecamatan, tooltip kategori jamak, dan pencarian akun sebagai admin biasa (memastikan tidak melihat lead akun lain).
  - Template lama 14 kolom masih terbaca, tapi kolom Kecamatan hanya terisi dari template baru - tim sebaiknya mengunduh ulang.
  - Lead `04.2607.0008` memakai kecamatan `Citatah` yang tidak ada di dataset (itu desa di Kec. Cipatat). Dibiarkan apa adanya oleh backfill; perlu dibetulkan manual lewat UI.
  - Membuang daftar putih `DISPLAY_NAMES` membuat seluruh isi tabel `needs_categories` tampil di dropdown - tinjau dan hapus kategori uji coba sebelum rilis.
  - Import masih CSV-only; template xlsx harus di-Save As CSV.
  - `composer` melaporkan 22 advisory keamanan pada 5 paket lama (di luar cakupan perubahan ini) - jalankan `composer audit`.
- Agent: Claude

### 2026-07-23 - Audit flow survey: perbaiki 10 bug

- Area: backend / ops / docs
- Ringkasan: Audit menyeluruh flow survey menemukan 10 masalah, semuanya diperbaiki.
  1. **Kritis** - `BROADCAST_CONNECTION=reverb` + `QUEUE_CONNECTION=sync` sementara `laravel/reverb` dan `pusher/pusher-php-server` tidak terpasang. Broadcast jalan inline dan melempar `Error: Class "Pusher\Pusher" not found`, sehingga setiap aksi survey (`store`/`assign`/`reschedule`/`reschedule-assignment`/`start`/`result`/`cancel`) balas 500 padahal data sudah tersimpan. Key `BROADCAST_CONNECTION` dihapus dari `.env` agar default framework `env('BROADCAST_CONNECTION', 'null')` dipakai. Menulis `BROADCAST_CONNECTION=null` tidak bisa - Laravel mengubahnya jadi PHP `null`, bukan nama driver.
  2. **Tinggi** - `broadcastOn()` memakai `Channel` publik dan `routes/channels.php` tidak ada, jadi pesan berisi nama klien, wilayah, nama surveyor, dan jadwal bisa dibaca siapa pun yang terhubung ke WebSocket. Diganti `PrivateChannel`, dibuat `routes/channels.php`, didaftarkan lewat `withBroadcasting()` di `bootstrap/app.php`.
  3. `cancel()` tidak pernah mengisi `cancelled_at` / `cancellation_reason` walau kolomnya ada. Sekarang diisi, plus menerima `cancellation_reason` opsional.
  4. `cancel()` bisa dijalankan pada survey yang sudah `cancelled` (notifikasi dan activity log ganda). Ditambah guard 422.
  5. Cek bentrok jadwal di `assign()` dan `rescheduleAssignment()` hanya melihat state `scheduled`, sedangkan `availability()` menghitung `scheduled` + `in_progress` sebagai sibuk. Surveyor yang sedang `in_progress` bisa di-double-book (33 survey `in_progress` di DB saat audit). Kini keduanya memakai daftar state yang sama.
  6. Saat `rescheduleAssignment()` mengganti surveyor, hanya surveyor baru yang dinotifikasi. Ditambah parameter `extraRecipients` pada `SurveyRealtimeUpdated` dan surveyor lama ikut diberi tahu.
  7. History awal `null -> requested` tidak pernah tertulis karena `store()` mengisi `state` sebelum `save()`, membuat `transitionTo()` menganggap tidak ada perubahan (terbukti: 0 baris `survey_status_histories` dengan `from_state` NULL). `state` kini tidak diisi saat create - kolomnya sudah `default('requested')`.
  8. `submitResult()` mengisi `actual_start_at = now()` saat survey tidak pernah di-start, membuat durasi tercatat 0 menit dan merusak metrik. Kini dibiarkan NULL = tidak terukur.
  9. `flushDashboardCache()` hanya menghapus cache super admin yang sedang login. Kini menghapus untuk semua super admin, konsisten dengan `Survey::transitionTo()`.
  10. Notifikasi `completed` hanya ke `requested_by` + `assigned_by`. Manager surveyor kini ikut menerima.
- File berubah:
  - `E-REPORT-BACKEND_API/.env`
  - `E-REPORT-BACKEND_API/bootstrap/app.php`
  - `E-REPORT-BACKEND_API/routes/channels.php` (baru)
  - `E-REPORT-BACKEND_API/app/Events/SurveyRealtimeUpdated.php`
  - `E-REPORT-BACKEND_API/app/Http/Controllers/Api/SurveyController.php`
  - `E-REPORT-APP/11-Modules/Survey.md`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up:
  - Realtime broadcast **nonaktif**. Untuk mengaktifkan: `composer require laravel/reverb pusher/pusher-php-server`, set `BROADCAST_CONNECTION=reverb`, jalankan `php artisan reverb:start`. Frontend juga belum punya listener Echo sama sekali dan harus memakai `.private('survey.managers')` / `.private('survey.surveyor.{id}')`, bukan `.channel()`.
  - `cancel()` kini menerima body opsional `cancellation_reason`. Frontend belum mengirimkannya; tetap jalan karena ada teks default.
  - Survey lama yang `completed` tanpa pernah di-start tetap punya `actual_start_at` hasil isian lama; perubahan hanya berlaku untuk data baru.
  - Tersisa 2 deprecation PHP 8.5 dari `vendor/.../BroadcastEvent.php` (internal framework, bukan dari perubahan ini).
  - Verifikasi lewat `php -l`, dispatch event, dan uji transaksi rollback. **Belum diuji lewat UI.**
- Agent: Claude

### 2026-07-23 - Tambah Analitik Tim Survey di halaman Analytics

- Area: backend / frontend / docs
- Ringkasan: Menambah tiga metrik operasional survey ke payload `GET /analytics` dan menampilkannya sebagai section "Analitik Tim Survey" di halaman `/analytics`. Metrik: (1) leaderboard surveyor — survey selesai, durasi rata-rata, ketepatan waktu mulai, deal-rate; (2) statistik reschedule — total, sumber admin/manager, deal-rate setelah reschedule; (3) backlog antrian survey yang belum dijadwalkan dengan bucket lama tunggu. Semua memakai data yang sudah ada, tanpa migration baru.
- File berubah:
  - `E-REPORT-BACKEND_API/app/Services/Reports/AnalyticsReportService.php` (import model Survey/SurveyReschedule/SurveyStatus; metode `scopeSurveyByAccount`, `resolveSurveyDealStatusIds`, `buildSurveyorLeaderboard`, `buildRescheduleAnalytics`, `buildSurveyBacklog`; wire ke `compute()`)
  - `E-REPORT-FRONTEND/src/app/analytics/analytics-view.tsx` (variabel turunan + section UI)
  - `E-REPORT-APP/11-Modules/Reporting-and-Analytics.md`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Analytics memakai cache 3 menit — jalankan `php artisan cache:clear` agar field baru langsung muncul. Deal-rate bergantung nama status di tabel `survey_statuses`; data seed saat ini 33 dari 35 survey `completed` punya `result_status_id` NULL dan hanya 2 punya timestamp aktual, sehingga kolom deal-rate/durasi dapat tampil 0 atau `-`. Belum diuji visual di browser.
- Agent: Claude

### 2026-07-23 - Matikan hairline atas toolbar konsultasi

- Area: frontend
- Ringkasan: Mengganti border dark toolbar menjadi slate gelap, membuat top border transparan, dan menghapus campuran putih pada hover tombol utama agar toolbar tidak tampak putih.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/globals.css`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Hard refresh browser jika CSS lama masih tersimpan cache.
- Agent: Codex

### 2026-07-23 - Gelapkan toolbar konsultasi dark mode

- Area: frontend
- Ringkasan: Mengurangi tint cyan/primary, border, hairline, dan radial glow pada `consultation-toolbar` di dark mode agar tidak terlalu terang.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/globals.css`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Hard refresh browser jika CSS lama masih tersimpan cache.
- Agent: Codex

### 2026-07-23 - Perbaiki hover tombol Lead Baru

- Area: frontend
- Ringkasan: Mengubah hover tombol `Lead Baru` di toolbar konsultasi agar tetap cerah, teks kontras, dan tidak tertutup surface gelap toolbar.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/globals.css`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Hard refresh browser jika CSS lama masih tersimpan cache.
- Agent: Codex

### 2026-07-23 - Redupkan border toolbar konsultasi

- Area: frontend
- Ringkasan: Menurunkan intensitas border dan glow pada `consultation-toolbar` serta control di dalamnya agar tidak terlalu terang/neon di dark mode dan light mode.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/globals.css`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Hard refresh browser jika CSS lama masih tersimpan cache.
- Agent: Codex

### 2026-07-23 - Adaptifkan toolbar konsultasi light mode

- Area: frontend
- Ringkasan: Menyesuaikan `consultation-toolbar` agar light mode memakai gradient ice/cyan sesuai tema terang, sementara dark mode tetap navy/amber. Control di dalam toolbar ikut adaptif dan tombol Lead Baru tetap primary.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/globals.css`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Hard refresh browser jika CSS lama masih tersimpan cache.
- Agent: Codex

### 2026-07-23 - Sesuaikan gradient toolbar konsultasi

- Area: frontend
- Ringkasan: Menambahkan kelas `consultation-toolbar` agar toolbar filter konsultasi memakai gradient dark navy dan amber glow yang menyatu dengan tema aplikasi.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/consultations/consultations-view.tsx`
  - `E-REPORT-FRONTEND/src/app/globals.css`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Hard refresh browser jika toolbar masih memakai cache CSS lama.
- Agent: Codex

### 2026-07-23 - Hapus background icon login

- Area: frontend
- Ringkasan: Menghapus background, ring, dan shadow dari icon input login/modal. Icon kini tampil sebagai glyph amber saja di dalam field.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/login/page.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Hard refresh browser jika UI lama masih tersimpan cache.
- Agent: Codex

### 2026-07-23 - Tampilkan icon login di atas input

- Area: frontend
- Ringkasan: Menambahkan `z-10` pada wrapper icon input login dan modal lupa password agar icon tidak tertutup background input.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/login/page.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Hard refresh browser jika UI lama masih tersimpan cache.
- Agent: Codex

### 2026-07-23 - Perbaiki warna icon login

- Area: frontend
- Ringkasan: Memperbaiki tampilan icon pada input login dan modal lupa password. Icon email/password tidak lagi memakai background langsung di elemen SVG, tetapi memakai wrapper khusus dengan warna primary yang stabil dan kontras lebih jelas di light/dark mode.
- File berubah:
  - `E-REPORT-FRONTEND/src/app/login/page.tsx`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Jalankan ulang dev server atau hard refresh browser jika masih melihat cache tampilan lama.
- Agent: Codex

### 2026-07-23 - Rapikan jarak canvas knowledge map

- Area: docs
- Ringkasan: Menyusun ulang `E-Report Knowledge Map.canvas` agar tidak menumpuk. Group diperbesar dan disebar lebih jauh, posisi node diperlebar, dan koneksi dikurangi dari 36 menjadi 34 edge agar diagram lebih lega.
- File berubah:
  - `E-REPORT-APP/E-Report Knowledge Map.canvas`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Refresh/reopen canvas di Obsidian untuk melihat layout baru.
- Agent: Codex

### 2026-07-23 - Buat canvas knowledge map

- Area: docs
- Ringkasan: Menambahkan canvas Obsidian terstruktur untuk knowledge base E-Report. Canvas berisi 47 node dan 36 edge dengan cluster Core, Overview/Architecture, Data/API, Backend, Frontend, Security, Modules, dan Ops tanpa warna pink/magenta.
- File berubah:
  - `E-REPORT-APP/E-Report Knowledge Map.canvas`
  - `E-REPORT-APP/Home.md`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Buka ulang Obsidian atau refresh file explorer jika canvas belum muncul.
- Agent: Codex

### 2026-07-23 - Hapus pink dari palette Obsidian

- Area: docs
- Ringkasan: Mengganti palette graph dan CSS Obsidian agar tidak memakai warna pink/magenta. Modul/workflows sekarang memakai teal, security memakai clean red, glossary memakai slate, dan warna domain lain memakai palette modern profesional.
- File berubah:
  - `E-REPORT-APP/.obsidian/snippets/e-report-vault-colors.css`
  - `E-REPORT-APP/.obsidian/graph-modern-preset.json`
  - `E-REPORT-APP/.obsidian/graph.json`
  - `E-REPORT-APP/08-Ops/Obsidian-Modern-Color-Setup.md`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Reload Obsidian jika graph masih menampilkan cache warna lama.
- Agent: Codex

### 2026-07-23 - Apply graph color preset

- Area: docs
- Ringkasan: Menyalin preset warna modern dari `.obsidian/graph-modern-preset.json` ke `.obsidian/graph.json`. Graph sekarang memiliki 12 color groups berdasarkan folder/domain.
- File berubah:
  - `E-REPORT-APP/.obsidian/graph.json`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Jika Obsidian masih menampilkan warna lama, buka ulang vault atau reload Obsidian.
- Agent: Codex

### 2026-07-23 - Revisi styling modern Obsidian

- Area: docs
- Ringkasan: Mengganti CSS vault ke tema dark graphite modern, menghapus override global yang membuat node graph semua biru, menambahkan preset graph color groups modern, dan menambahkan panduan apply saat Obsidian menimpa `graph.json`.
- File berubah:
  - `E-REPORT-APP/.obsidian/snippets/e-report-vault-colors.css`
  - `E-REPORT-APP/.obsidian/graph-modern-preset.json`
  - `E-REPORT-APP/08-Ops/Obsidian-Modern-Color-Setup.md`
  - `E-REPORT-APP/08-Ops/Runbook.md`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: `graph.json` sedang ditimpa oleh proses Obsidian yang terbuka. Tutup Obsidian lalu apply preset agar warna node per-domain muncul.
- Agent: Codex

### 2026-07-23 - Styling warna vault Obsidian

- Area: docs
- Ringkasan: Menambahkan CSS snippet `e-report-vault-colors` dan color groups untuk graph Obsidian berdasarkan folder/domain agar knowledge graph lebih mudah dibaca.
- File berubah:
  - `E-REPORT-APP/.obsidian/snippets/e-report-vault-colors.css`
  - `E-REPORT-APP/.obsidian/appearance.json`
  - `E-REPORT-APP/.obsidian/graph.json`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Jika warna belum muncul, reload Obsidian atau aktifkan snippet manual di Settings -> Appearance -> CSS snippets.
- Agent: Codex

### 2026-07-23 - Perluas knowledge base menjadi modular

- Area: docs
- Ringkasan: Menambahkan knowledge map, module index, catatan detail per modul domain, security/auth/role matrix, testing guide, frontend UI/hook/cache notes, backend internals, deployment notes, dan glossary. Home dan AGENTS juga diperluas agar agent lain punya jalur baca lebih jelas.
- File berubah:
  - `E-REPORT-APP/Home.md`
  - `E-REPORT-APP/01-Overview/Knowledge-Map.md`
  - `E-REPORT-APP/03-Backend/Backend-Map.md`
  - `E-REPORT-APP/04-Frontend/Frontend-Map.md`
  - `E-REPORT-APP/05-Database/Data-Model.md`
  - `E-REPORT-APP/07-Workflows/Core-Workflows.md`
  - `E-REPORT-APP/08-Ops/Deployment-Notes.md`
  - `E-REPORT-APP/11-Modules/*`
  - `E-REPORT-APP/12-Security/*`
  - `E-REPORT-APP/13-Testing/Testing-Guide.md`
  - `E-REPORT-APP/14-Frontend-UI/*`
  - `E-REPORT-APP/15-Backend-Internals/*`
  - `E-REPORT-APP/16-Glossary/Domain-Glossary.md`
  - `AGENTS.md`
- Risiko / follow-up: Tetap perlu validasi berkala terhadap kode saat fitur berubah; graph Obsidian perlu refresh/reopen agar node baru tampil.
- Agent: Codex

### 2026-07-23 - Setup vault Obsidian knowledge base

- Area: docs
- Ringkasan: Mengubah vault `E-REPORT-APP` dari default Obsidian menjadi knowledge base E-Report. Menambahkan peta project, arsitektur, backend, frontend, data model, API catalog, workflow, runbook, decision log, dan template update.
- File berubah:
  - `E-REPORT-APP/Home.md`
  - `E-REPORT-APP/01-Overview/Project-Brief.md`
  - `E-REPORT-APP/02-Architecture/System-Architecture.md`
  - `E-REPORT-APP/03-Backend/Backend-Map.md`
  - `E-REPORT-APP/04-Frontend/Frontend-Map.md`
  - `E-REPORT-APP/05-Database/Data-Model.md`
  - `E-REPORT-APP/06-API/API-Catalog.md`
  - `E-REPORT-APP/07-Workflows/Core-Workflows.md`
  - `E-REPORT-APP/08-Ops/Runbook.md`
  - `E-REPORT-APP/09-Decisions/Decision-Log.md`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
  - `E-REPORT-APP/99-Templates/Change-Entry.md`
  - `AGENTS.md`
  - `CLAUDE.md`
- Risiko / follow-up: Perlu terus diperbarui saat kode berubah. Versi Next.js perlu diverifikasi karena README dan `package.json` berbeda.
- Agent: Codex

### 2026-07-23 - Redesain navbar/header aplikasi

- Area: frontend
- Ringkasan: Mendesain ulang komponen header dengan tampilan lebih premium, bersih, dan konsisten. Perubahan utama:
  - **Avatar inisial**: Menampilkan inisial user dengan `--primary-theme` color sebagai avatar lingkaran (fallback ke ikon User).
  - **Account badge** lebih ringkas: inisial akun di kotak warna, nama akun, badge SA untuk super_admin.
  - **Breadcrumb** lebih halus: dot separator menggantikan ChevronRight.
  - **Icon buttons** styling seragam (PWA, theme toggle, notification).
  - **Profile popover** lebih bersih: gradient header sesuai primary_color, info rows lebih rapi.
  - Tinggi header dikompres dari `h-16` ke `h-14` untuk proporsi lebih ramping.
- File berubah:
  - `E-REPORT-FRONTEND/src/components/layout/header.tsx`
- Risiko / follow-up: Verifikasi visual di semua role (admin, super_admin, surveyor, manager_surveyor).
- Agent: Codex
# 2026-07-28 - Tambahkan 23 akun surveyor operasional

- Menambahkan `SurveyorUserSeeder` yang idempotent untuk membuat 23 akun surveyor dari daftar tim operasional.
- Email login dinormalisasi dengan pola `{nama}@surveyor.ereport.local`; spasi pada nama menjadi titik.
- Seluruh akun memakai role `surveyor`, tidak terikat akun cabang, dan password awal wajib diberikan melalui `SEED_SURVEYOR_PASSWORD`.
- Seeder dapat dijalankan ulang tanpa membuat akun duplikat serta akan mengaktifkan kembali akun target yang sebelumnya dihapus secara lunak.
### 2026-07-30 - Buat video panduan login/reset password/lapor bug

- Area: docs/video
- Ringkasan: Menambahkan proyek HyperFrames berdurasi 20 detik untuk panduan cara login, meminta reset password melalui WhatsApp dukungan dari halaman login, dan melaporkan bug dari widget E-Report.
- File berubah:
  - `E-REPORT-APP/videos/e-report-login-reset-bug-20s/DESIGN.md`
  - `E-REPORT-APP/videos/e-report-login-reset-bug-20s/SCRIPT.md`
  - `E-REPORT-APP/videos/e-report-login-reset-bug-20s/STORYBOARD.md`
  - `E-REPORT-APP/videos/e-report-login-reset-bug-20s/index.html`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Render MP4 perlu diverifikasi ulang jika naskah atau branding aplikasi berubah.
- Agent: Codex
### 2026-08-03 - Auth gate server-side dan proxy API same-origin (opsi A)

- Area: frontend/auth/performance
- Ringkasan: Menambahkan mode opt-in `NEXT_PUBLIC_SAME_ORIGIN_API=true` agar request API, Sanctum CSRF, dan broadcasting memakai origin frontend melalui rewrite Next.js. Saat `NEXT_PUBLIC_AUTH_GATE=true`, middleware memvalidasi cookie sesi ke `auth/me` sebelum mengirim halaman privat; sesi hilang, kedaluwarsa, atau API gagal diarahkan ke `/login`.
- File berubah:
  - `E-REPORT-FRONTEND/src/middleware.ts`
  - `E-REPORT-FRONTEND/src/lib/api/client.ts`
  - `E-REPORT-FRONTEND/next.config.js`
  - `E-REPORT-FRONTEND/README.md`
- Kompatibilitas: mode direct API tetap default untuk local/dev. Deployment production harus mengatur `API_PROXY_TARGET`, `NEXT_PUBLIC_SAME_ORIGIN_API`, dan `NEXT_PUBLIC_AUTH_GATE` secara eksplisit.
- Verifikasi: redirect tanpa sesi, proxy CSRF/auth, lint, build, dan smoke browser dijalankan setelah konfigurasi gate diaktifkan.
- Risiko / follow-up: domain frontend production dan cookie Sanctum perlu diuji di staging; target API tidak boleh dicantumkan sebagai `NEXT_PUBLIC_*` saat mode proxy.
- Agent: Codex

### 2026-08-03 - Catat baseline spek server production

- Area: docs/ops
- Ringkasan: Menambahkan catatan spek server target production/home-lab untuk E-Report, termasuk OS, CPU, RAM, storage, jaringan, layanan yang cocok, batas aman operasional, rekomendasi environment, dan checklist verifikasi staging.
- File berubah:
  - `E-REPORT-APP/08-Ops/Production-Server-Spec.md`
  - `E-REPORT-APP/08-Ops/Runbook.md`
  - `E-REPORT-APP/08-Ops/Deployment-Notes.md`
  - `E-REPORT-APP/10-Change-Log/Change-Log.md`
- Risiko / follow-up: Perlu load test nyata di server tersebut sebelum dinyatakan siap untuk traffic production penuh.
- Agent: Codex

### 2026-08-04 - Redeploy production ke server aaPanel + refresh DB dari local

- Area: ops/deploy
- Ringkasan: Redeploy backend Laravel + frontend Next.js ke server produksi `mki@100.76.23.12:2201` (Ubuntu 24.04, aaPanel). Update kode dari GitHub origin/master (BE `37448ed`→`9773ed4`, FE `b854426`→`525e0de`), refresh database produksi dengan dump DB local `e_report` (timpa penuh, 31 tabel, 44 user), konfigurasi Redis (session/cache/queue), dan pastikan semua service auto-restart saat reboot.
- Backup (di harddisk `/data`, timestamp 20260804_1204):
  - `/data/db-backups/ereport-PROD-server_20260804_1204.sql.gz` (DB produksi lama, 70K)
  - `/data/db-backups/ereport-LOCAL-source_20260804_1204.sql.gz` (sumber dump local, 47K)
  - `/data/app-backups/ereport-backend_20260804_1204.tar.gz` (23M)
  - `/data/app-backups/ereport-frontend_20260804_1204.tar.gz` (204M)
  - `/data/app-backups/fe_env_prod_20260804_1204.bak` (`.env.production` FE produksi)
- Langkah: backup → dump+transfer DB local → `git reset --hard origin/master` (BE & FE) → `composer install --no-dev` → restore FE `.env.production` (config Reverb produksi) + `npm ci` + `npm run build` → import DB local (timpa) + `migrate --force` (Nothing to migrate) → `optimize:clear`+`config/route/view/event:cache` + `storage:link` → restart `ereport-queue`, `ereport-reverb`, reload nginx + php-fpm-83, `pm2 restart e_report_frontend` + `pm2 save`.
- Auto-restart boot terverifikasi enabled: `redis`, `mysqld`, `nginx`, `php-fpm-83`, `ereport-queue`, `ereport-reverb`, `pm2-root` (resurrect FE).
- Audit kesehatan: semua service active+enabled; `/sanctum/csrf-cookie`→204; `/api/v1/auth/me` guest→401; `/api/v1/auth/login` body kosong→422; FE `127.0.0.1:3000/login`→308; Redis `Cache::put/get`→ok; 0 error di `laravel.log` pasca-deploy; reverb error log bersih.
- Risiko / follow-up: (1) Data produksi lama TERGANTIKAN data local — pemulihan hanya via backup di atas. (2) Kredensial (SSH/DB/Redis/Telegram token/VAPID) sempat dilihat dalam plaintext — disarankan rotasi. (3) `.env.production` FE tetap tracked di git (berisi secret) — pertimbangkan untuk untrack.
- Agent: REMOX (Claude Opus 4.8)

### 2026-08-04 - Rotasi kredensial (DB/Redis/Reverb) + untrack .env.production FE

- Area: ops/security
- Ringkasan: Rotasi kredensial produksi yang sempat terekspos, dan menghapus `.env.production` FE dari tracking git. Kredensial baru tersimpan di `.env` server (backend) dan `.env.production` server (frontend). VAPID & SSH TIDAK dirotasi (keputusan user); Telegram bot token harus diregenerasi user via @BotFather.
- Perubahan:
  - **DB** `sql_api_ereport_interiorcustom_id`: `SET PASSWORD` ke nilai baru; `DB_PASSWORD` di backend `.env` diperbarui; password lama ditolak (Access denied).
  - **Redis** `requirepass`: diganti via `CONFIG SET` + `CONFIG REWRITE` (persist ke `redis.conf`, data tidak flush); `REDIS_PASSWORD` backend `.env` diperbarui.
  - **Reverb** `REVERB_APP_KEY` + `REVERB_APP_SECRET` (backend `.env`) + `NEXT_PUBLIC_REVERB_APP_KEY` (FE `.env.production`) diganti; FE di-rebuild karena key di-bake saat build.
  - **Untrack**: `git rm --cached .env.production` + `.gitignore` FE kini abaikan semua `.env*`. Commit `7ae88a0` di-push ke GitHub. Server FE di-`reset --hard` ke `7ae88a0` lalu file `.env.production` di-restore (kini untracked/ignored → deploy berikutnya tak akan menghapusnya).
- Penerapan: backend `optimize:clear`+`config:cache`, restart `php-fpm-83`+`ereport-queue`+`ereport-reverb`; FE `npm run build` + `pm2 restart e_report_frontend` + `pm2 save`.
- Backup env ter-rotasi: `/data/app-backups/fe_env_prod_ROTATED_20260804.bak`.
- Verifikasi PASS: app `Cache::put/get`→ok, `User::count()`=44; DB pass baru OK & pass lama ditolak; Redis `ping`→PONG dgn pass baru; `/sanctum/csrf-cookie`→204; `/api/v1/auth/login`→422; FE `/login`→308; Reverb LISTEN :8080, error log bersih.
- Follow-up: (1) Telegram bot token WAJIB diregenerasi user via @BotFather lalu update `.env`. (2) VAPID & SSH password belum dirotasi. (3) Nilai kredensial baru ada di `.env` server — simpan aman.
- Agent: REMOX (Claude Opus 4.8)

### 2026-08-04 - Fix login CSRF token mismatch (same-origin API) + matikan Cloudflare beacon

- Area: frontend/auth/ops
- Gejala: login di `putrainterior.com/login` gagal dengan `CSRF token mismatch` (419). Console juga error CSP block `static.cloudflareinsights.com/beacon.min.js` dan `GET /api/v1/auth/me 401` (yang terakhir NORMAL, bukan bug).
- Akar masalah CSRF: FE (`putrainterior.com`) dan API (`api-ereport.interiorcustom.id`) beda registrable-domain. `.env.production` FE tidak mengaktifkan same-origin sehingga `src/lib/api/client.ts` memakai `NEXT_PUBLIC_API_URL` absolut → panggilan lintas-domain → cookie `XSRF-TOKEN` diset di domain API dan tak terbaca JS di domain FE. Rebuild FE saat deploy 2026-08-04 menyingkap misconfig ini (build lama diduga sudah same-origin).
- Fix: tambah ke `.env.production` server FE lalu rebuild + `pm2 restart` + `pm2 save`:
  ```
  NEXT_PUBLIC_SAME_ORIGIN_API=true
  API_PROXY_TARGET=https://api-ereport.interiorcustom.id
  ```
  Client kini panggil `/api` & `/sanctum` same-origin → di-proxy Next.js (`next.config.js` rewrites) ke backend → cookie XSRF sedomain → CSRF cocok. Backend `.env` sudah benar (`SANCTUM_STATEFUL_DOMAINS=putrainterior.com,www.putrainterior.com`, `SESSION_SECURE_COOKIE=true`, `SESSION_DOMAIN` kosong).
- Verifikasi (curl cookie-jar, tanpa browser): `GET https://putrainterior.com/sanctum/csrf-cookie`→204 + cookie `XSRF-TOKEN` domain `putrainterior.com`; `POST /api/v1/auth/login` + header `X-XSRF-TOKEN`→401 (kredensial), BUKAN 419 → CSRF LOLOS.
- Cloudflare beacon: user set Web Analytics RUM ke **Disable** di dashboard (hostname `interiorcustom.id`) → beacon `beacon.min.js` tak lagi di-inject, error CSP hilang. Tak ubah kode.
- Pencegahan berulang: dicatat ke memory `csrf-same-origin-prod` + `deploy-ereport-server`. `.env.production` gitignored → var same-origin mudah hilang saat rebuild di env baru; WAJIB dicek sebelum tiap `npm run build`.
- Backup env: `/data/app-backups/fe_env_prod_SAMEORIGIN_20260804.bak`.
- Agent: REMOX (Claude Opus 4.8)

### 2026-08-04 - Fix avatar & logo akun 404 (proxy /storage + sync file storage)

- Area: frontend/assets/ops
- Gejala: foto profil (baru diupload) dan logo akun tak tampil (broken image) di `putrainterior.com`, walau upload "berhasil".
- Dua akar masalah:
  1. **Serving**: efek samping mode same-origin — `api.baseUrl` kosong, jadi `src/components/layout/header.tsx` render `<img src="/storage/..">` ke origin FE. `next.config.js` tak mem-proxy `/storage` → 404. Padahal backend serve `https://api-ereport.interiorcustom.id/storage/..`→200.
  2. **File hilang**: DB ditimpa data local (2026-08-04), sehingga `users.avatar_path` & `accounts.logo_path` merujuk file yang diupload di LOCAL (`avatars/`, `accounts/`) — file itu tak ada di server.
- Fix:
  1. Tambah rewrite `/storage/:path*` → `API_PROXY_TARGET` di `next.config.js` (commit `df38008`, di-push + deploy + rebuild FE + `pm2 restart`).
  2. Sync file storage local → server: `pscp` `storage/app/public/{avatars,accounts}` ke `/tmp`, lalu `cp` ke `storage/app/public/` + `chown www:www`. File upload server yang sudah ada (mis. avatar superadmin baru) tidak ditimpa.
- Verifikasi: `curl https://putrainterior.com/storage/avatars/<superadmin>` & `/storage/accounts/<logo>` → semua **200**.
- Pencegahan: dicatat ke memory `csrf-same-origin-prod` (rewrite /storage jangan dihapus) + `deploy-ereport-server` (kalau timpa DB, wajib sync file storage).
- Agent: REMOX (Claude Opus 4.8)
