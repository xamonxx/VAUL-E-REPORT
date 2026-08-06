# Decision Log

Gunakan folder ini untuk ADR ringan. Satu keputusan besar sebaiknya satu file.

## Format singkat

- Tanggal:
- Status: proposed / accepted / superseded
- Konteks:
- Keputusan:
- Dampak:
- File terkait:

## Keputusan awal

### 2026-08-03 - Matikan runtime cache start URL PWA

- Status: accepted
- Konteks: `cacheStartUrl: false` saja masih menghasilkan `start-url` pada service worker karena perilaku default plugin untuk dynamic start URL.
- Keputusan: set `dynamicStartUrl: false` bersama pengecualian cache API, RSC, dan cross-origin; verifikasi hasil build dengan script statis.
- Dampak: start URL selalu diambil dari network; offline shell tidak bergantung pada cache halaman privat.

### 2026-08-03 - Tangani secret histori tanpa rewrite otomatis

- Status: accepted
- Konteks: secret produksi ditemukan pada histori Git, sedangkan working tree perlu segera dibersihkan tanpa mengubah clone/branch pengguna secara diam-diam.
- Keputusan: redaksi dokumentasi aktif segera; rotasi credential dan purge histori Git menjadi gate rilis terkoordinasi, bukan rewrite otomatis oleh audit.
- Dampak: temuan tetap open sampai bukti rotasi dan purge tersedia.

### 2026-08-03 - Notifikasi absensi memakai tabel sendiri dan role superadmin eksplisit

- Status: accepted
- Konteks: Absensi admin perlu memberi tahu superadmin, tetapi tidak boleh bercampur dengan notifikasi survey/catatan/reminder agar isi dan akses tidak salah dibaca atau bocor ke role lain.
- Keputusan: buat `attendance_notifications` sebagai channel persistensi khusus absensi. Penerima hanya diambil dari `User::cachedSuperAdminIds()`, summary/count/list/read/delete mengecek `isSuperAdmin()` dan `user_id` pemilik notifikasi.
- Dampak: payload panel dan Web Push absensi terpisah dari survey, cache notifikasi superadmin bisa di-invalidasi tepat, dan role admin/surveyor/manager surveyor tetap mendapat count 0/daftar kosong.
- File terkait: `E-REPORT-BACKEND_API/app/Http/Controllers/Api/ReportAttendanceController.php`, `E-REPORT-BACKEND_API/app/Http/Controllers/Api/NotificationController.php`, `E-REPORT-BACKEND_API/app/Services/NotificationSummaryService.php`, `E-REPORT-BACKEND_API/database/migrations/2026_08_03_000001_create_attendance_notifications_table.php`, `E-REPORT-FRONTEND/src/components/layout/notification-center.tsx`.

### 2026-07-31 - Survey admin biasa dibatasi ke admin pengaju

- Status: accepted
- Konteks: Admin biasa perlu melihat Data Konsumen Survey, tetapi jadwal survey yang diajukan admin lain tidak boleh ikut terekspos.
- Keputusan: Akses admin ke daftar/detail survey operasional memakai dua syarat: `account_id` sama dan `requested_by` sama dengan user login. Manager Surveyor tetap menjadi role lintas akun/jadwal untuk operasional survey.
- Dampak: Admin biasa dapat memantau konsumen survey yang dia ajukan sendiri tanpa melihat jadwal admin lain. Aksi maps, reschedule request, dan cancel admin ikut mengikuti batas yang sama.
- File terkait: `E-REPORT-BACKEND_API/app/Http/Controllers/Api/SurveyController.php`, `E-REPORT-BACKEND_API/app/Policies/SurveyPolicy.php`, `E-REPORT-FRONTEND/src/app/survey-consumers/survey-consumers-view.tsx`, `E-REPORT-FRONTEND/src/lib/auth/roles.ts`.

### 2026-07-28 - Daftar survey memakai summary-first dan urutan waktu operasional

- Status: accepted
- Konteks: kartu survey menampilkan seluruh data sebagai tabel dua kolom. Alamat dan kebutuhan panjang membuat kartu sulit dipindai, sementara jadwal, nama konsumen, dan WhatsApp tidak cukup menonjol. Endpoint juga mengurutkan semua state non-request berdasarkan `updated_at`.
- Keputusan: jadwal, identitas konsumen, WhatsApp, dan aksi utama menjadi ringkasan kartu; informasi lengkap dipindahkan ke dialog detail responsif. Urutan default API mengikuti waktu operasional tiap state, dengan `sort=latest` sebagai alternatif.
- Dampak: daftar tetap padat dan konsisten saat data banyak; surveyor melihat jadwal terdekat lebih dahulu tanpa kehilangan akses ke detail lengkap.
- File terkait: `E-REPORT-FRONTEND/src/app/surveys/surveys-view.tsx`, `E-REPORT-BACKEND_API/app/Http/Controllers/Api/SurveyController.php`, `E-REPORT-APP/11-Modules/Survey.md`.

### 2026-07-27 - Reverb produksi dijalankan sebagai service lokal di balik Nginx

- Status: accepted
- Konteks: backend Laravel sudah memiliki event `SurveyRealtimeUpdated` dan private broadcast channels, tetapi proses `php artisan reverb:start` belum berjalan, env Reverb kosong, dan port WebSocket belum diproxy ke domain API.
- Keputusan: jalankan Reverb sebagai systemd service `ereport-reverb.service` dengan bind lokal `127.0.0.1:8080`, lalu expose hanya lewat Nginx di `https://api-ereport.interiorcustom.id/app/*` dan `/apps/*`. Broadcast Laravel memakai `BROADCAST_CONNECTION=reverb`; frontend diberi env publik Reverb untuk koneksi klien berikutnya.
- Dampak: WebSocket tetap hidup setelah reboot dan tidak membuka port publik langsung. Handshake publik sudah valid `101 Switching Protocols`.
- File terkait: `.env` produksi backend/frontend, `/etc/systemd/system/ereport-reverb.service`, Nginx extension `reverb.conf`, `E-REPORT-APP/10-Change-Log/Change-Log.md`.

### 2026-07-27 - Token unduhan pindah ke header; `QueryTokenToHeader` dipertahankan sementara

- Status: accepted
- Konteks: unduhan export memakai `<a href>` bertoken lewat `buildExportUrl()`, didukung middleware `QueryTokenToHeader` yang menerima `?token=`. URL bertoken bocor ke DOM, riwayat browser, `Referer`, dan salinan pengguna — dan token Sanctum saat itu tidak punya masa berlaku. Satu token super admin sudah benar-benar keluar dari mesin lewat jalur ini.
- Keputusan: pindahkan seluruh unduhan ke `api.downloadFile()` (header `Authorization` + blob), hapus `buildExportUrl()`, dan **persempit** `QueryTokenToHeader` ke GET pada rute export — bukan menghapusnya.
- Alasan tidak langsung menghapus middleware: tautan bertoken bisa saja sudah beredar di bookmark, pesan, atau dokumen. Mematikannya seketika membuat tautan itu gagal tanpa penjelasan. Dipersempit dulu supaya token yang bocor tidak bisa dipakai menulis, dihapus setelah tautan lama habis umurnya. Middleware ditandai deprecated agar tidak dipakai endpoint baru.
- Konsekuensi yang harus ditangani bersamaan: memberi token masa berlaku (12 jam) berarti sesi bisa ditolak di tengah pemakaian, padahal 401 tidak ditangani di mana pun — `/auth/me` hanya jalan saat mount. Tanpa penanganan 401, perbaikan keamanan ini akan berubah jadi aplikasi yang rusak diam-diam setelah 12 jam. Karena itu penanganan 401 masuk dalam perubahan yang sama, bukan ditunda.
- Dampak: token tidak lagi pernah muncul di URL yang dihasilkan frontend. Pengguna login ulang kira-kira sekali per hari kerja.
- File terkait: `app/Http/Middleware/QueryTokenToHeader.php`, `config/sanctum.php`, `E-REPORT-FRONTEND/src/lib/api/client.ts`, `src/lib/hooks/useFileDownload.ts`.

### 2026-07-27 - Insight Analitik tetap HTML, tapi escaping jadi wajib di sumber

- Status: accepted
- Konteks: `buildInsights()` mengirim string HTML yang dirender frontend lewat `dangerouslySetInnerHTML`. Nilai dari DB disisipkan tanpa escape, dan importer CSV bisa menaruh HTML sembarang ke kolom wilayah — stored XSS yang berujung pengambilalihan akun super admin karena token ada di `localStorage`. Lihat SEC-001 di Change Log.
- Opsi yang dipertimbangkan:
  1. Ganti kontrak jadi `{template, values}` dan rakit di frontend tanpa `dangerouslySetInnerHTML`.
  2. Pertahankan `html`, wajibkan `e()` pada setiap nilai yang disisipkan.
- Keputusan: ambil opsi 2 untuk sekarang. Opsi 1 lebih benar secara struktural — menghilangkan HTML sink sepenuhnya — tapi mengubah kontrak API dan menyentuh frontend, sementara celahnya Critical dan perlu ditutup segera. Opsi 2 menutup celah dengan patch minimal dan tidak mengubah kontrak.
- Dampak: `buildInsights()` kini punya peringatan doc block bahwa nilainya adalah HTML sink dan setiap nilai wajib lewat `e()`, dengan catatan agar escaping diterapkan ke argumen `sprintf` — bukan hasilnya — supaya `<mark>` tetap utuh. Opsi 1 tetap terbuka sebagai perbaikan struktural berikutnya.
- File terkait: `app/Services/Reports/AnalyticsReportService.php`, `E-REPORT-FRONTEND/src/app/analytics/analytics-view.tsx`.

### 2026-07-27 - `statuses.survey` (grup pelaporan) dipisah dari `statuses.survey_entry` (tahap pengajuan)

- Status: accepted
- Konteks: satu kunci config dipakai untuk dua maksud berbeda — "status apa yang dihitung sebagai survey di laporan" dan "di status mana survey boleh diajukan". Ketika `survey` diperluas jadi tiga nama demi pelaporan, gerbang pengajuannya ikut melebar tanpa disengaja: survey jadi bisa diajukan pada lead yang surveynya **sudah selesai**, dan scope "menunggu pengajuan" jadi kabur. Empat pemakai juga masih membaca config itu sebagai string, tiga di antaranya fatal (`trim()` atas array).
- Keputusan: pisahkan jadi dua kunci. `statuses.survey` tetap grup pelaporan; `statuses.survey_entry` adalah satu status tempat survey diajukan. Keduanya diakses lewat `ConsultationStatusGroups` — tidak ada lagi `config('statuses.*')` langsung di consumer.
- Alternatif yang ditolak: memakai elemen pertama array `survey` sebagai tahap pengajuan. Implisit dan rapuh — urutan array jadi punya makna tersembunyi.
- Dampak: perilaku sebelum pelebaran grup dipulihkan persis (`survey_entry` = `Request Survey`). `scopeNeedsSurveyRequest` kini memfilter `status_category_id` (terindeks) alih-alih `whereHas` + `LOWER(TRIM(name))`.
- Follow-up: `Account.php`, `AccountController`, dan `AuditDatabase` masih membaca `config('statuses.deal')` langsung — aman selama `deal` string tunggal, rapuh bila kelak dilebarkan.
- File terkait: `config/statuses.php`, `app/Support/ConsultationStatusGroups.php`.

### 2026-07-23 - E-REPORT-APP dipakai sebagai vault Obsidian

- Status: accepted
- Konteks: folder `E-REPORT-APP` sudah ada sebagai vault default.
- Keputusan: gunakan folder ini sebagai knowledge base proyek.
- Dampak: agent lain harus mulai dari `Home.md` dan mencatat perubahan di `10-Change-Log/Change-Log.md`.

### 2026-07-24 - Blok analisa berat di Analytics dibuat opt-in, bukan dihapus

- Status: accepted
- Konteks: `AnalyticsReportService` menjalankan 89 query per request. Lima blok — `comparisonMatrix`, `performanceAnalysis`, `cohortConversion`, `stageVelocity`, `onlyInquiryAnalysis` — menghabiskan 59 di antaranya (66%) tapi **tidak dipakai di manapun**: bukan di UI web, bukan Excel, bukan PDF. Repo ini bukan git repo, jadi menghapus kode berarti hilang permanen.
- Keputusan: jadikan opt-in lewat filter `include` (mis. `?include=stage_velocity`), default mati. Bentuk response tetap sama — key selalu ada, isinya kosong saat tidak diminta, supaya tidak ada risiko `undefined index` di konsumen manapun.
- Dampak: 89 → 31 query, 161 ms → 72 ms. Angka yang dipakai UI identik. Konstanta `AnalyticsReportService::EXTRA_BLOCKS` jadi daftar resmi blok yang bisa diminta.
- File terkait: `app/Services/Reports/AnalyticsReportService.php`, `app/Http/Requests/AnalyticsReportRequest.php`.

### 2026-07-24 - config/statuses.php untuk pending & cancelled tidak cocok data nyata

- Status: accepted — **diperbaiki 2026-07-25** lewat `ConsultationStatusGroups` + isi ulang config. Dashboard pending 0→195, cancelled 0→57.
- Konteks: `config/statuses.php` memetakan `pending` ke alias `Pending`/`Masuk`/`Baru` dan `cancelled` ke `Batal`/`Cancelled`. **Tidak satu pun nama itu ada** di tabel `status_categories`. Akibatnya KPI "Total Pending" dan "Total Cancel" di Dashboard resolve ke `null` dan **selalu menampilkan 0 tanpa error** — bug diam yang belum pernah terlihat.
- Keputusan: isi ulang dengan nama status yang benar-benar ada, dan satukan definisi status ke satu kelas `ConsultationStatusGroups` supaya `AnalyticsReportService` dan `DashboardService` tidak lagi punya versi sendiri-sendiri yang isinya sudah berbeda.
- Dampak: **sudah dikerjakan 2026-07-25**. Dashboard pending 0 → 195, cancelled 0 → 57. (Baris ini sebelumnya tertulis "belum dikerjakan" sehingga bertentangan dengan status di judul entri — dikoreksi saat audit 2026-07-27.)
- File terkait: `config/statuses.php`, `app/Services/DashboardService.php`, `app/Services/Reports/AnalyticsReportService.php`.

### 2026-07-24 - Geo Analytics memakai choropleth, bukan peta bermarker

- Status: accepted (implementasi ditunda)
- Konteks: permintaan modul Geo Analytics dengan marker cluster, heatmap, radius, dan polygon analysis. Pemeriksaan DB: **tidak ada kolom koordinat sama sekali** di 30 tabel, `consultations.address` kosong 100%, `district` 100% berisi placeholder. Geocoding data lama tidak mungkin akurat — presisi maksimal titik pusat kota, membuat semua konsumen satu kota menumpuk di satu titik identik.
- Keputusan: Fase 1 memakai choropleth per provinsi memanfaatkan `province`/`city` yang sudah terisi. Fitur berbasis titik ditunda sampai ada pipeline koordinat (kolom lat/lng + map picker di form konsultasi).
- Dampak: Supercluster, Turf.js, dan Deck.GL tidak perlu dipasang. Satu-satunya dependensi baru `maplibre-gl`.
- File terkait: [[11-Modules/Geo-Analytics]].

### 2026-07-24 - Grup akun dijadikan filter lintas modul

- Status: accepted
- Konteks: `AccountGroup` (PC, NPP1, NPP2) sudah dipakai Rekap Jadwal Surveyor dan export Absensi, tapi belum bisa dipakai memfilter Analitik/Dashboard. Export Absensi bahkan masih memampatkan NPP1+NPP2 jadi satu ember lama `NPP`.
- Keputusan: daftar grup disajikan lewat endpoint `GET /api/v1/master-data/account-groups` yang diturunkan dari `App\Support\AccountGroup`, bukan konstanta yang disalin di frontend. Menambah grup baru cukup diurus di satu file PHP; dropdown ikut berubah tanpa ubah TypeScript.
- Dampak: filter grup tersedia di Analitik, Dashboard, dan export Absensi. Terverifikasi dengan menambah grup uji `NPP3` sementara — langsung muncul di endpoint dan export tanpa perubahan frontend.
- File terkait: `app/Support/AccountGroup.php`, `app/Http/Controllers/Api/MasterDataController.php`, `app/Services/Reports/AnalyticsReportService.php`.
### 2026-08-03 - Gunakan auth gate middleware + proxy same-origin secara opt-in

- Status: accepted
- Konteks: Halaman privat Next.js dapat mengunduh bundle besar sebelum client mengetahui sesi Sanctum tidak valid. API saat ini mendukung mode direct cross-origin untuk local/dev, sehingga perubahan global tanpa opt-in berisiko memutus cookie dan realtime.
- Keputusan: Tambahkan kombinasi `NEXT_PUBLIC_SAME_ORIGIN_API=true` dan `NEXT_PUBLIC_AUTH_GATE=true`. Client memakai URL relatif, rewrite Next.js meneruskan ke `API_PROXY_TARGET`, dan middleware memvalidasi cookie sesi ke `/api/v1/auth/me` sebelum merender route privat. Endpoint teknis/public tetap dapat diakses untuk login, CSRF, asset, dan broadcasting.
- Dampak: request tanpa sesi berhenti di redirect ringan dan tidak mengunduh private page bundle; target API tidak terekspos ke browser. Mode direct API tetap fallback sampai production staging tervalidasi.
- Batasan: ini bukan pengganti authorization Laravel. Semua endpoint backend tetap wajib memakai auth/policy; konfigurasi domain Sanctum dan cookie harus diuji di staging.
- File terkait: `E-REPORT-FRONTEND/src/middleware.ts`, `src/lib/api/client.ts`, `next.config.js`.
