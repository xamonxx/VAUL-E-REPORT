# Backend Map

## Stack

- Laravel 11
- PHP platform composer: 8.3.30
- Sanctum 4.3
- MySQL/MariaDB
- DOMPDF untuk PDF
- Web Push via `minishlink/web-push`
- Redis opsional via `predis/predis`

## Folder penting

- `app/Http/Controllers/Api`: endpoint REST.
- `app/Http/Requests`: validasi input.
- `app/Models`: entity dan relationship Eloquent.
- `app/Policies`: authorization per resource.
- `app/Services`: service domain, import, dashboard, reports, telegram, push.
- `app/Support`: helper domain seperti wilayah, tanggal Indonesia, palette, account group.
- `database/migrations`: evolusi schema.
- `database/seeders`: data awal/demo.
- `routes/api.php`: semua route `/api/v1`.

## Controller domain

- `AuthController`: login, lookup warna login, profil user aktif, logout.
- `ConsultationController`: CRUD lead, preview ID, import, template import, update status.
- `SurveyController`: request survey, assign, reschedule, start, result, cancel, recap, availability, history.
- `DashboardController`: ringkasan dashboard.
- `AnalyticsController`: grafik dan agregasi analitik.
- `MasterDataController`: kategori kebutuhan/status, survey status, account list, surveyors, user management.
- `ReportAttendanceController`: report harian admin dan export attendance.
- `NotificationController`: unread count, summary, mark read, clear notifications.
- `BugReportController`: public submit, super admin management.
- `AuditLogController`: audit logs dan online users.
- `ExportController`: CSV, Excel, PDF, recap surveyor.
- `WilayahController`: provinsi, kota, kecamatan.

## Model inti

- `User`: role, account, auth identity.
- `Account`: cabang/akun operasional, logo, account group.
- `Consultation`: lead utama, soft delete, status, kategori kebutuhan, notes, reminders, survey, duplicate detection, ID generator.
- `Survey`: workflow survey dan transition state.
- `SurveyStatus`, `SurveyStatusHistory`, `SurveyActivityLog`, `SurveyNotification`, `SurveyReschedule`: ekosistem survey.
- `NeedsCategory`, `StatusCategory`: master pipeline dan kebutuhan.
- `ReportAttendance`: laporan harian admin.
- `Reminder`, `ConsultationNote`: follow-up lead.
- `AuditLog`, `BugReport`, `PushSubscription`, `ConsultationImport`.

## Business rule penting

- Consultation ID format: `AA.YYMM.NNNN`, dibuat atomik via tabel `consultation_sequences`.
- Admin hanya melihat data sesuai `account_id`; super admin melihat semua.
- Survey state: `requested`, `scheduled`, `in_progress`, `completed`, `cancelled`.
- Saat survey pindah ke `in_progress` atau `completed`, status konsultasi dapat disinkronkan ke `Sedang Survey` atau `Selesai Survey`.
- Endpoint debug destructive dibatasi `super_admin`.
- Bug report submit publik dibatasi throttle ketat.

## Catatan turunan

- [[15-Backend-Internals/Backend-Internals]]
- [[15-Backend-Internals/Models-and-Relationships]]
- [[15-Backend-Internals/Services]]
- [[15-Backend-Internals/Policies]]
- [[15-Backend-Internals/Migrations-and-Indexes]]
- [[12-Security/Security-and-Access]]
