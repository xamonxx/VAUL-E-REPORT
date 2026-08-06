# Bug Reports

Terkait: [[12-Security/Security-and-Access]], [[14-Frontend-UI/UI-and-Navigation]], [[08-Ops/Runbook]].

## Tujuan

Memungkinkan user atau calon user melaporkan bug, termasuk dari halaman login. Super admin dapat melihat dan mengelola laporan.

## Backend

- Controller: `BugReportController`.
- Request: `BugReportRequest`.
- Model: `BugReport`.
- Route public submit: `POST /api/v1/bug-reports`, throttle 5/min.
- Route management: index/show/update/delete, role `super_admin`.

## Frontend

- Route management: `/bug-reports`.
- Widget: `components/bug-report/bug-report-widget.tsx`.
- Hook: `useBugReports.ts`, `useReportBug.ts`.

## Data sensitif

Screenshot bug report bisa mengandung PII, karena itu list/detail hanya untuk `super_admin`.

## Risiko perubahan

- Perubahan upload/storage screenshot harus cek validasi, ukuran file, path storage, dan privacy.

