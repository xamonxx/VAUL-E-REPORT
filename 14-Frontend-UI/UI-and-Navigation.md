# UI and Navigation

Terkait: [[04-Frontend/Frontend-Map]], [[14-Frontend-UI/Frontend-Hooks]], [[14-Frontend-UI/Frontend-State-and-Cache]].

## Shell aplikasi

- `src/app/layout.tsx`
- `src/components/providers.tsx`
- `src/components/auth-guard.tsx`
- `src/components/layout/app-layout.tsx`
- `src/components/layout/sidebar.tsx`
- `src/components/layout/header.tsx`
- `src/components/layout/bottom-nav.tsx`
- `src/components/layout/notification-center.tsx`

## Route ke modul

| Route | Modul |
| --- | --- |
| `/login` | [[11-Modules/Auth-and-Users]] |
| `/dashboard` | [[11-Modules/Reporting-and-Analytics]] |
| `/consultations` | [[11-Modules/Consultations]] |
| `/surveys` | [[11-Modules/Survey]] |
| `/survey-consumers` | [[11-Modules/Survey]] |
| `/rekap-jadwal-surveyor` | [[11-Modules/Survey]] |
| `/accounts` | [[11-Modules/Accounts]] |
| `/analytics` | [[11-Modules/Reporting-and-Analytics]] |
| `/report-attendances` | [[11-Modules/Reporting-and-Analytics]] |
| `/master-data` | [[11-Modules/Master-Data]] |
| `/audit-logs` | [[11-Modules/Audit-and-Debug]] |
| `/bug-reports` | [[11-Modules/Bug-Reports]] |
| `/settings` | [[11-Modules/Auth-and-Users]] |
| `/debug` | [[11-Modules/Audit-and-Debug]] |
| `/offline` | PWA fallback |

## Komponen UI dasar

Folder `src/components/ui` berisi button, dialog, table, tabs, input, select, popover, tooltip, badge, calendar, card, dan komponen dasar lainnya.

## Catatan desain

- Aplikasi ini operasional/dashboard, jadi UI harus padat, jelas, dan cepat dipindai.
- Jangan mengubah pattern layout global tanpa cek semua halaman yang memakai `AppLayout`.

