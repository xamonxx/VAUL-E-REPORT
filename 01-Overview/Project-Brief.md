# Project Brief

E-Report adalah aplikasi operasional untuk manajemen lead konsultasi, survey lokasi, reminder follow-up, absensi/report harian admin, analitik, export laporan, audit log, dan bug report.

## Komponen utama

- Backend API: Laravel 11, PHP 8.2/8.3, Sanctum, MySQL/MariaDB, queue database atau Redis, DOMPDF, export spreadsheet.
- Frontend web: Next.js App Router, React 19, TypeScript, Tailwind CSS v4, TanStack Query, Zustand, shadcn/Base UI, PWA.
- Vault knowledge: folder `E-REPORT-APP`, dibuka dengan Obsidian.

## Aktor dan role

- `super_admin`: akses penuh, master data, accounts, users, audit logs, debug destructive, bug report management.
- `admin`: lead/konsultasi, dashboard, analytics, report attendance, export leads/analytics.
- `surveyor`: terlibat dalam assignment dan eksekusi survey.
- `manager_surveyor`: mengatur jadwal/assignment survey dan export rekap jadwal surveyor.

## Modul bisnis

- Konsultasi/leads: CRUD, filter, import massal, ID otomatis, status pipeline, notes, reminders.
- Survey: request dari konsultasi, assign surveyor, reschedule, start, submit result, cancel, notifications, activity log.
- Master data: account/cabang, kategori kebutuhan, kategori status lead, survey status, users.
- Reporting: dashboard, analytics, attendance, export CSV/PDF/Excel, rekap jadwal surveyor.
- Maintenance: audit logs, online users, debug data dummy, clear logs, bug reports.

## Catatan penting

- Dokumentasi frontend lama menyebut Next.js 16, tetapi `package.json` saat ini memakai `next ^15.2.0`. Agent harus cek versi aktual dari package sebelum mengikuti dokumentasi eksternal.
- Komentar di backend route menyebut Sanctum cookie dan Bearer token. Implementasi frontend saat ini menyimpan Bearer token di `localStorage`, tetap mengirim `credentials: include`, dan masih punya `getCsrfCookie()` untuk kompatibilitas lama.

