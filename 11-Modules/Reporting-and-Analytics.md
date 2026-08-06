# Reporting and Analytics

Terkait: [[11-Modules/Consultations]], [[11-Modules/Survey]], [[11-Modules/Accounts]], [[08-Ops/Runbook]].

## Tujuan

Memberi ringkasan kinerja lead, dashboard operasional, analytics, absensi admin, export file, dan rekap jadwal surveyor.

## Backend

- Controller: `DashboardController`, `AnalyticsController`, `ReportAttendanceController`, `ExportController`.
- Service:
  - `DashboardService`
  - `AnalyticsReportService`
  - `LeadsReportService`
  - `LeadsExcelExporter`
  - `AnalyticsExcelExporter`
  - `AdminReportAttendanceExcelExporter`
  - `SurveyorScheduleRecapService`
  - `SurveyorScheduleRecapExcelExporter`
  - `ReportPeriodResolver`
  - `SpreadsheetXmlToXlsxConverter`
- Views PDF: `resources/views/reports/pdf`.

## Frontend

- Route: `/dashboard`, `/analytics`, `/report-attendances`, `/rekap-jadwal-surveyor`.
- Hook: `useDashboard.ts`, `useAnalytics.ts`, `useReportAttendances.ts`, `useSurveyorScheduleRecap.ts`.
- Visualisasi memakai Recharts dan export cards.

## API

- `GET /dashboard`
- `GET /analytics`
- `GET /report-attendances`
- `POST /report-attendances`
- `GET /report-attendances/export`
- `GET /export/csv`
- `GET /export/leads/excel`
- `GET /export/leads/pdf`
- `GET /export/analytics/excel`
- `GET /export/analytics/pdf`
- `GET /export/surveys/recap/excel`

## Analitik Tim Survey

Payload `GET /analytics` menyertakan tiga metrik operasional survey (dibangun di `AnalyticsReportService`):

- `surveyorLeaderboard`: per surveyor pada periode terpilih — `completed`, `avg_duration_min` (dari `actual_start_at`/`actual_finish_at`), `on_time_rate` (mulai maksimal 30 menit setelah `scheduled_at`), `deals`, `deal_rate`.
- `rescheduleAnalytics`: `total`, `by_admin`, `by_manager`, `rescheduled_surveys`, `rescheduled_deal_rate` — sumber data tabel `survey_reschedules`.
- `surveyBacklog`: survey `state=requested` yang belum dijadwalkan; `total_pending`, `oldest_days`, `avg_wait_days`, dan bucket `<1 hari` / `1-3 hari` / `>3 hari`. Kondisi terkini, tidak terikat periode.

Status hasil survey yang dihitung sebagai deal ditentukan `resolveSurveyDealStatusIds()` (cocokkan nama: Deal, Selesai/Deal, Closing, Closing Deal).

UI ada di section "Analitik Tim Survey" pada `/analytics` (paling bawah halaman).

Catatan data: survey hasil seed banyak yang `result_status_id` NULL dan tanpa timestamp aktual, sehingga `deal_rate` dan `avg_duration_min` bisa kosong/0 meski metrik benar.

## Filter periode

`ReportPeriodResolver` mengenal empat tipe: `weekly`, `monthly`, `yearly`, dan `custom`.

`custom` memakai `start_date` + `end_date` bebas. Periode pembanding dihitung sebagai jendela sama panjang tepat sebelum `start_date`. Dua catatan:

- `buildTrendSeries()` beralih dari bucket harian ke bulanan bila rentang **lebih dari 90 hari**, berlaku untuk semua tipe periode (bukan lagi khusus `yearly`).
- `cohortConversion` sengaja mengembalikan kosong untuk `custom`, karena bentuknya terikat satu tahun kalender (`whereYear`).

## Filter grup akun

`account_group` (PC / NPP1 / NPP2) tersedia di `GET /analytics` dan di-AND-kan dengan `account`. Hanya super admin; admin sudah terkunci ke akunnya sendiri.

Daftar grup diambil dari `GET /master-data/account-groups` yang diturunkan dari `App\Support\AccountGroup` — bukan konstanta yang disalin di frontend, sehingga grup baru cukup ditambahkan di satu file PHP.

Implementasi menyimpan grup terpilih di property instance `AnalyticsReportService::$accountGroup`, bukan meneruskannya lewat ~20 parameter method. **Aman selama service di-resolve baru per request** — sudah dipastikan tidak ada binding singleton dan tidak ada Octane. Jangan jadikan service ini singleton.

## Blok analisa opt-in

Lima blok berat tidak dihitung kecuali diminta eksplisit lewat filter `include`:

`comparison_matrix`, `performance_analysis`, `cohort_conversion`, `stage_velocity`, `only_inquiry`

Contoh: `GET /analytics?include=stage_velocity,comparison_matrix`. Daftar resminya ada di konstanta `AnalyticsReportService::EXTRA_BLOCKS`.

Alasan: kelimanya menghabiskan 59 dari 89 query per request tapi tidak dipakai UI web, Excel, maupun PDF. Setelah dimatikan: 31 query, 72 ms. Bentuk response **tidak berubah** — key selalu ada, isinya kosong saat tidak diminta, jadi konsumen lama tidak perlu menyesuaikan.

## Cache

- Dashboard/analytics memakai cache dan invalidasi saat data consultation/survey berubah.
- Cache key `analytics:report:{userId}:{lastUpdated}:{md5(filters)}` — seluruh `$filters` ikut di-hash, jadi filter baru otomatis membedakan cache tanpa perubahan kode.
- Perubahan filter atau tanggal harus dipastikan tidak memakai cache yang salah.

## Risiko perubahan

- Export memakai file download; frontend `buildExportUrl()` bisa menyisipkan token query.
- PDF/Excel harus diverifikasi setelah perubahan struktur data.
- KPI **Pending** dan **Cancel** di Dashboard saat ini selalu 0 — alias di `config/statuses.php` tidak cocok satu pun nama di `status_categories`. Lihat [[09-Decisions/Decision-Log]]. Jangan pakai dua angka itu sebelum diperbaiki.
- Definisi status survey/deal saat ini terduplikasi di `AnalyticsReportService` dan `DashboardService` dengan isi yang **sudah berbeda**. Rencana penyatuan ada di [[11-Modules/Geo-Analytics]].

