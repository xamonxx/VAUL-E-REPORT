# Role Matrix

Terkait: [[11-Modules/Auth-and-Users]], [[11-Modules/Survey]], [[11-Modules/Consultations]], [[15-Backend-Internals/Policies]].

## Role

| Role | Label UI | Fungsi umum |
| --- | --- | --- |
| `super_admin` | Super Admin | Akses penuh, master data, users, accounts, audit, debug |
| `admin` | Admin | Kelola lead account sendiri, dashboard, analytics, attendance, request survey |
| `surveyor` | Surveyor | Melihat/mengerjakan survey miliknya, termasuk membatalkan jadwal miliknya sebelum selesai |
| `manager_surveyor` | Manager Surveyor | Assign survey, rekap jadwal, availability, cancel |

## Access matrix ringkas

| Modul | super_admin | admin | surveyor | manager_surveyor |
| --- | --- | --- | --- | --- |
| Dashboard | Ya | Ya | Tidak langsung | Tidak langsung |
| Consultations | Semua | Account sendiri | Tidak | Tidak |
| Notes/reminders | Semua | Account sendiri | Tidak | Tidak |
| Survey list | Semua | Diajukan sendiri | Miliknya | Semua |
| Survey assign | Ya | Tidak | Tidak | Ya |
| Survey start/result | Ya | Tidak | Miliknya | Ya |
| Survey cancel | Ya | Diajukan sendiri | Miliknya | Ya |
| Survey recap | Ya | Tidak | Tidak | Ya |
| Accounts CRUD | Ya | Tidak | Tidak | Tidak |
| Master data mutasi | Ya | Tidak | Tidak | Tidak |
| Audit logs | Ya | Tidak | Tidak | Tidak |
| Debug destructive | Ya | Tidak | Tidak | Tidak |
| Bug report management | Ya | Tidak | Tidak | Tidak |

## Catatan

- `RoleMiddleware` membolehkan `super_admin` melewati role route.
- Policy juga punya `before()` untuk `super_admin`.
- Admin scoping utama tetap memakai `account_id`; khusus survey operasional dan Data Konsumen Survey diperketat dengan `requested_by = user login` supaya jadwal admin lain tidak terekspos.
- Realtime dan push catatan memakai daftar penerima yang sama. Private channel `consultation-notes.user.{userId}` hanya dapat dibuka oleh admin/super admin pemilik user tersebut; role lain dan admin akun berbeda tidak menerima payload.
- Pembatalan survey lintas role tetap dibatasi policy: admin hanya survey yang dia ajukan sendiri, surveyor hanya survey yang ditugaskan, manager surveyor semua survey operasional, dan survey yang sudah selesai tidak dapat dibatalkan.
