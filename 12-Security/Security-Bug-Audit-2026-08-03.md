# Security and Bug Audit 2026-08-03

Tanggal audit: 3 Agustus 2026  
Baseline: working tree saat audit, termasuk perubahan user yang belum di-commit  
Stack: Laravel 12.64, Next.js 15.5.22, React 19.2.4, MySQL/MariaDB, Sanctum cookie auth, Reverb, queue, Web Push PWA  
Metode: static review, dependency/secret scan, database sintetis, feature test, Playwright smoke, dan produksi pasif

## Verdict

| Target | Verdict | Alasan |
|---|---|---|
| Source saat ini | **HOLD FOR SECRET ROTATION** | Source aktif sudah bebas dari kredensial yang terdeteksi, tetapi histori Git memuat kredensial database produksi yang harus dirotasi dan dibersihkan melalui proses repository yang disetujui. |
| Produksi aktif | **HOLD / DEPLOY AND REVERIFY** | Domain HTTP masih memberi 200, HSTS frontend belum terlihat, dan `sw-v2.js` masih dicache publik 4 jam. Protected API sudah menolak tanpa sesi. |

Tidak ada mutation, brute force, upload, login, SQLi aktif, atau push nyata yang dilakukan ke produksi.

## Ringkasan Temuan

| ID | Severity | Status | Temuan |
|---|---:|---|---|
| SEC-2026-04 | Critical* | Open / source redacted | Kredensial database produksi pernah masuk histori Git pada `DEPLOYMENT.md`; nilai tidak dicetak dalam laporan. |
| SEC-2026-01 | High 8.1 | Fixed | Endpoint Web Push bebas memungkinkan authenticated SSRF dan subscription tanpa batas. |
| SEC-2026-02 | High 8.0 | Fixed | Service worker masih memiliki cache GET `cross-origin` yang dapat menyimpan response API privat. |
| SEC-2026-03 | High 7.5 | Fixed | Plugin PWA tetap menghasilkan runtime cache `start-url` walau `cacheStartUrl` false; konfigurasi dinamis belum dimatikan. |
| DEP-2026-01 | High 7.0 | Fixed | `sharp 0.34.5` terdampak GHSA-f88m-g3jw-g9cj. |
| DEP-2026-02 | High 7.5 | Fixed | `brace-expansion <1.1.17` pada dependency development terdampak ReDoS. |
| CICD-2026-01 | High 8.1 | Fixed | SSH host verification dimatikan, YAML heredoc invalid, dan failure deploy penting disembunyikan. |
| OPS-2026-01 | High 8.1 | Source fixed, prod open | HTTP produksi tidak redirect ke HTTPS; HSTS dan security header terbaru belum terpasang. |
| DATA-2026-01 | Medium 6.5 | Mitigated | Backup SQL lokal berisi email/password hash tidak di-ignore dan berisiko ter-commit. |
| BUG-2026-01 | Medium 5.3 | Fixed | Fresh seed gagal setelah `User::$fillable` diperketat. |
| BUG-2026-02 | Medium 5.3 | Fixed | Parameter tanggal absensi invalid dapat menghasilkan HTTP 500. |
| UX-2026-01 | Medium 4.3 | Open | Gangguan jaringan dibungkus sebagai `ApiError(status=0)`, tetapi auth guard hanya mengenali `TypeError`; UI dapat menganggap sesi hilang. |
| SEC-2026-05 | Medium 4.8 | Open | CSP frontend masih membutuhkan `unsafe-inline` untuk script/style. Tidak ditemukan HTML sink aktif pada audit ini. |
| NOTIF-2026-01 | Low 3.1 | Open | Status baca catatan bersifat global pada record, sehingga aksi bersihkan dapat memengaruhi visibilitas admin lain dalam akun yang sama. |

## Temuan High

### SEC-2026-04: Kredensial database produksi di histori Git

- Severity: Critical*; CVSS: bergantung pada validitas dan privilege credential saat ini; gunakan CVSS 9.1 bila credential masih aktif dan memiliki akses produksi.
- CWE: CWE-798, CWE-540
- OWASP: A02 Cryptographic Failures, A07 Identification and Authentication Failures
- Bukti: histori Git `DEPLOYMENT.md` pada commit `aa78aac` memuat password database produksi dalam dokumentasi tracked. Nilai rahasia sengaja tidak ditampilkan.
- Dampak: akses langsung ke database produksi bila credential masih aktif atau dapat dipakai ulang; potensi baca/ubah data dan pivot.
- Perbaikan source: nilai pada `DEPLOYMENT.md` saat ini sudah diganti placeholder dan tidak menampilkan rahasia.
- Tindakan wajib sebelum ship: rotasi password database produksi, audit penggunaan credential, lalu purge histori Git melalui prosedur repository resmi. Rewrite histori tidak dilakukan otomatis karena dapat merusak clone/branch dan harus dikoordinasikan.
- Status: **Open until rotation and history purge are evidenced**.

### SEC-2026-03: PWA runtime cache `start-url`

- CWE: CWE-525
- OWASP: A01 Broken Access Control, A05 Security Misconfiguration
- Bukti: hasil build awal tetap membuat cache `start-url` walau `cacheStartUrl: false`; default `dynamicStartUrl` plugin masih aktif.
- Dampak: dokumen start URL privat dapat masuk Cache Storage perangkat bersama.
- Perbaikan: `dynamicStartUrl: false` ditambahkan di `next.config.js`; cache API/RSC/cross-origin tetap dikeluarkan.
- Regression: build baru dan `npm run test:pwa-security` lulus; service worker tidak mengandung `start-url`, `/api/v1`, atau `/sanctum`.

### SEC-2026-01: Web Push SSRF dan resource abuse

- CWE: CWE-918, CWE-400
- OWASP: A10 SSRF, A04 Insecure Design
- Bukti awal: user terautentikasi dapat menyimpan URL HTTPS arbitrer sebagai endpoint push; service kemudian melakukan pengiriman server-side ke endpoint tersebut. Ukuran key dan jumlah perangkat juga tidak dibatasi.
- Dampak: probing layanan internal, request ke target yang dikendalikan attacker, pertumbuhan tabel, dan beban queue/pengiriman.
- Perbaikan:
  - allowlist exact host FCM, Mozilla, Apple dan suffix Windows yang boundary-safe;
  - wajib HTTPS, tolak credential URL dan IP literal;
  - batasi key, content encoding, 10 subscription per user, dan throttle 20/menit;
  - validasi ulang serta hapus subscription legacy sebelum pengiriman;
  - log hanya hash endpoint, bukan URL lengkap.
- Lokasi: `app/Support/PushEndpoint.php`, `app/Http/Controllers/Api/PushSubscriptionController.php:29`, `app/Services/WebPushService.php:37`, `config/webpush.php:22`, `routes/api.php:185`.
- Regression: target localhost/private/attacker dan suffix confusion ditolak 422; provider resmi diterima; subscription ke-11 menghapus yang tertua; endpoint legacy invalid dipangkas.

### SEC-2026-02: Cache PWA dapat menyimpan API privat

- CWE: CWE-525
- OWASP: A01 Broken Access Control, A05 Security Misconfiguration
- Bukti awal: service worker hasil build memiliki route `NetworkFirst` dengan `cacheName:"cross-origin"`. API produksi berada pada origin berbeda dari frontend.
- Dampak: response GET pengguna dapat tertinggal di Cache Storage, menjadi stale, dan berisiko terlihat pada sesi berikutnya di perangkat bersama.
- Perbaikan: keluarkan `cross-origin`, `apis`, `next-data`, `pages`, `pages-rsc`, dan prefetch RSC dari runtime caching; start URL dinamis juga dimatikan.
- Lokasi: `E-REPORT-FRONTEND/next.config.js:16-20`.
- Regression build: `sw-v2.js` tidak mengandung cache privat di atas maupun token `/api/v1` dan `/sanctum`.

### DEP-2026-01 dan DEP-2026-02: dependency High

- `sharp` dipaksa ke `0.35.3`, di atas versi patched `0.35.0` pada [GitHub Advisory GHSA-f88m-g3jw-g9cj](https://github.com/advisories/GHSA-f88m-g3jw-g9cj).
- `npm audit fix` memperbarui jalur development `brace-expansion`; versi aktif terendah sekarang `1.1.18`.
- `npm audit` penuh dan `npm audit --omit=dev` sama-sama melaporkan 0 vulnerability.
- Next.js aktif `15.5.22`, melampaui patch line `15.5.21` yang tercantum pada [rilis keamanan resmi Next.js](https://nextjs.org/blog).

### CICD-2026-01: integritas deployment

- CWE: CWE-295, CWE-703
- OWASP: A08 Software and Data Integrity Failures
- Bukti awal: dua workflow memakai `StrictHostKeyChecking=no`; satu known-host adalah placeholder; kegagalan Composer, npm, dan migration ditutup dengan `|| true`; satu heredoc YAML invalid.
- Perbaikan: host key wajib dari GitHub Secret, `StrictHostKeyChecking=yes`, permission file ketat, failure command menjadi fatal, dan YAML tervalidasi parser.
- Lokasi: `.github/workflows/deploy-interiorcustom.yml`, `.github/workflows/deploy.yml`.
- Prasyarat deploy: isi `INTERIORCUSTOM_SSH_KNOWN_HOSTS` dan `HOSTINGER_SSH_KNOWN_HOSTS` dengan key yang diverifikasi out-of-band.

### OPS-2026-01: HTTPS produksi belum enforced

- CWE: CWE-319
- Bukti 2026-08-03:
  - `http://api-ereport.interiorcustom.id` -> 200, bukan redirect;
  - `http://putrainterior.com` -> 200, bukan redirect;
  - response HTTPS frontend tidak memiliki HSTS yang terlihat dan hanya membawa cache header publik;
  - source baru sudah menambah redirect 308 berdasarkan `X-Forwarded-Proto` dan HSTS.
- Dampak: halaman frontend yang dimuat lewat HTTP dapat dimodifikasi di jaringan sebelum user diarahkan ke transport aman.
- Status: source fixed, produksi tetap High sampai deployment dan retest pasif membuktikan HTTP -> HTTPS serta HSTS aktif.

## Attack Surface dan Matriks Akses

- 115 route Laravel dipetakan. Seluruh route bisnis memakai `auth:sanctum`.
- Public yang disengaja: login/CSRF bootstrap, lookup warna terbatas, dan submit bug report yang di-throttle.
- `super_admin`: dashboard lintas akun, master data, audit/debug, absensi lintas admin, dan notification attendance milik penerima.
- `admin`: consultation dalam scope account, survey hanya yang dia ajukan, attendance hanya diri sendiri.
- `surveyor`: survey yang ditugaskan; dashboard/master data administratif ditolak.
- `manager_surveyor`: operasi penjadwalan survey lintas akun; consultation administratif ditolak.
- Private broadcast channel memeriksa user, account/pemilik, dan role; channel publik untuk payload bisnis tidak ditemukan.

## Notifikasi Absensi

- Record attendance dibuat idempotent melalui unique insert; submit kedua menghasilkan 422 tanpa notifikasi duplikat.
- Penerima diturunkan hanya dari `cachedSuperAdminIds()` dan setiap penerima memiliki row sendiri.
- Endpoint count/summary/read/delete attendance memeriksa `isSuperAdmin()` dan `user_id` pemilik notification.
- Admin, surveyor, dan manager tidak memperoleh record/API payload attendance.
- Web Push memakai ID superadmin yang sama; subscription terikat `user_id` dan unsubscribe dibatasi ke pemilik.
- Exception push dicatat dan tidak menggagalkan penyimpanan absensi.

## Database, Queue, dan PWA

- Database `e_report_audit` dibuat dari 63 migration dan fixture sintetis; tidak ada data produksi yang disalin.
- Seluruh migration berstatus Ran dan `migrate:fresh --seed` berhasil setelah seeder diperbaiki.
- Unique attendance mencegah duplicate submit; notification dibuat hanya setelah insert sukses.
- Scheduler terdeteksi untuk cleanup harian dan `reminders:push-due` setiap menit.
- Kegagalan Web Push dibatasi di service/controller dan tidak mematahkan mutation utama.
- `sw-v2.js` produksi memang tersedia; `/sw.js` bukan nama worker proyek. Deploy aktif masih memberi cache 4 jam, sedangkan source baru memberi `no-cache, no-store`.

## Produksi Pasif

| Check | Hasil |
|---|---|
| API root HTTPS dan `/up` | 200 |
| Protected `/api/v1/auth/me` tanpa sesi | 401 JSON, private/no-cache |
| `/.env`, `/storage/logs/laravel.log`, `/_ignition/health-check` | 404 |
| Frontend dan `manifest.json` | 200 |
| `sw-v2.js` | 200, tetapi cache policy deploy lama |
| HTTP -> HTTPS | Gagal, masih 200 |
| HSTS/CSP/header source terbaru | Belum aktif penuh |

## Verification Gate

| Gate | Hasil |
|---|---|
| `php artisan test` | 13 test, 63 assertion, lulus |
| PHP syntax scan | 0 error |
| Route cache/list | lulus, 115 route |
| Migration fresh/status/seed | lulus pada `e_report_audit` |
| Composer validate/audit | valid, 0 advisory |
| `npm run lint` | lulus, 0 warning setelah generated PWA di-ignore |
| `npm run build` | lulus, 21 page, middleware dan PWA terbangun |
| `npm audit` full/prod | 0 vulnerability |
| Browser smoke desktop 1440x900 | protected route redirect ke `/login`; 0 console error tak terduga/5xx |
| Browser smoke Android 412x915 | protected route redirect ke `/login`; 0 console error tak terduga/5xx |
| PWA cache regression | lulus; tidak ada private runtime cache/API path |

## Residual Risk dan Manual Verification

Item berikut tidak dapat dibuktikan dari repository atau pengujian pasif:

- **MANUAL/HIGH:** deploy source terbaru lalu verifikasi redirect HTTPS, HSTS, CSP, dan cache `sw-v2.js` pada Cloudflare/origin.
- **MANUAL/CRITICAL:** rotasi credential database produksi yang pernah masuk histori, audit akses, dan purge histori Git; source redaction saja belum cukup.
- **MANUAL/HIGH:** verifikasi GitHub known-host secrets secara out-of-band sebelum workflow dijalankan.
- **MANUAL:** backup restore drill, enkripsi/retensi backup, firewall origin, cron, queue worker, Reverb service, alerting, uptime, dan log retention.
- **MANUAL:** Web Push nyata pada Android PWA terpasang dengan VAPID produksi; audit hanya membuktikan service worker build, subscription contract, routing penerima, dan failure isolation.
- Authenticated browser smoke dan Android Web Push nyata belum dijalankan pada turn ini karena credential fixture lokal yang valid tidak tersedia; tetap perlu dijalankan memakai akun sintetis.
- Medium/Low terbuka (`UX-2026-01`, `SEC-2026-05`, `NOTIF-2026-01`) perlu backlog terpisah dan tidak menutup blocker Critical di atas.

## Release Checklist

1. Isi dan verifikasi dua secret SSH known-host.
2. Deploy backend/frontend dari source audit ini.
3. Purge cache Cloudflare untuk HTML dan `sw-v2.js`.
4. Ulangi check HTTP redirect, HSTS, CSP, cookie Secure/SameSite, dan protected 401.
5. Uji satu absensi sintetis production-safe sesuai prosedur operasional dan pastikan hanya akun superadmin test menerima panel/Web Push.
6. Baru ubah verdict produksi menjadi **CLEAR TO SHIP**.
