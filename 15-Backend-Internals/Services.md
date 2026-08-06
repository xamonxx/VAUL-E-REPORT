# Services

Terkait: [[03-Backend/Backend-Map]], [[11-Modules/Reporting-and-Analytics]], [[11-Modules/Notifications]], [[11-Modules/Consultations]].

## Domain services

- `ConsultationImportService`: parsing dan proses import lead.
- `DashboardService`: agregasi dashboard.
- `NotificationSummaryService`: cache dan ringkasan notification.
- `WebPushService`: kirim web push.
- `TelegramService`, `TelegramNotifier`: integrasi Telegram.

## Report services

- `LeadsReportService`: data export leads.
- `LeadsExcelExporter`: Excel leads.
- `AnalyticsReportService`: data analytics.
- `AnalyticsExcelExporter`: Excel analytics.
- `AdminReportAttendanceExcelExporter`: export attendance.
- `SurveyorScheduleRecapService`: rekap jadwal surveyor.
- `SurveyorScheduleRecapExcelExporter`: export recap surveyor.
- `ReportPeriodResolver`: resolver periode report.
- `SpreadsheetXmlToXlsxConverter`: konversi XML spreadsheet ke XLSX.

## Saat menambah service

- Pastikan controller tipis.
- Pastikan service punya input/output jelas.
- Jika service dipakai export, verifikasi format file.
- Update modul domain terkait dan [[15-Backend-Internals/Backend-Internals]].

