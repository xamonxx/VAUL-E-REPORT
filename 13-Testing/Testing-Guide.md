# Testing Guide

Terkait: [[08-Ops/Runbook]], [[03-Backend/Backend-Map]], [[04-Frontend/Frontend-Map]].

## Backend

Command umum:

```bash
cd C:\laragon\www\E-REPOR_v0.1\E-REPORT-BACKEND_API
php artisan test
php artisan route:list
php artisan migrate:status
```

Jika mengubah API:

- Tes validasi request.
- Tes authorization role/policy.
- Tes response shape untuk frontend.
- Tes cache invalidation bila menyentuh dashboard/notifications.

## Frontend

Command umum:

```bash
cd C:\laragon\www\E-REPOR_v0.1\E-REPORT-FRONTEND
npm run lint
npm run build
```

Jika mengubah UI:

- Verifikasi route render.
- Verifikasi form validation.
- Verifikasi loading/error/empty state.
- Verifikasi mobile layout.
- Verifikasi query invalidation setelah mutation.

## Smoke test manual

1. Login sebagai `super_admin`.
2. Buka dashboard.
3. Buka consultations list dan detail.
4. Buat/edit lead test.
5. Request survey dari lead.
6. Login manager surveyor, assign survey.
7. Login surveyor, start dan submit result.
8. Cek notification center.
9. Export salah satu laporan.

## Catatan

- Belum ada ringkasan test suite lengkap di vault. Agent yang menambahkan test baru harus menulis catatan di changelog.

