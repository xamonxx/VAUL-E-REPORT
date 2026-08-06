# Accounts

Terkait: [[11-Modules/Auth-and-Users]], [[11-Modules/Consultations]], [[11-Modules/Reporting-and-Analytics]], [[05-Database/Data-Model]].

## Tujuan

Account mewakili cabang/akun operasional. Lead, user admin, target, dan reporting banyak bergantung pada `account_id`.

## Backend

- Controller: `AccountController`.
- Model: `Account`.
- Routes:
  - `GET /api/v1/accounts/categories`
  - `apiResource /api/v1/accounts`
  - `GET /api/v1/master-data/accounts`
- Akses mutasi account: `super_admin`.

## Field penting

- `name`
- `logo_path`
- `description`
- `city`
- `province`
- `target_leads`
- `account_group` melalui migration tambahan.

## Relasi

- Account has many users.
- Account has many admin users.
- Account has many consultations.
- Consultation ID memakai account padded sebagai prefix `AA`.

## Frontend

- Route: `/accounts`.
- Hook: `useAccounts.ts`.
- Master data account dipakai untuk filter dan form consultation.

## Catatan

- Account conversion rate dihitung dari jumlah consultation dengan status deal.
- Perubahan account bisa berdampak ke dashboard, analytics, filter lead, dan access control admin.

