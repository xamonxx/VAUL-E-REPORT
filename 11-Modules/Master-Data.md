# Master Data

Terkait: [[11-Modules/Consultations]], [[11-Modules/Survey]], [[11-Modules/Auth-and-Users]], [[12-Security/Role-Matrix]].

## Tujuan

Master data menyimpan data referensi yang dipakai form, filter, status pipeline, survey status, account, surveyor, dan user.

## Backend

- Controller: `MasterDataController`.
- Model: `NeedsCategory`, `StatusCategory`, `SurveyStatus`, `User`, `Account`.
- Seeder: `AccountGroupSeeder`, seeders kategori/status.

## Endpoint read umum

- `GET /master-data/needs-categories`
- `GET /master-data/status-categories`
- `GET /master-data/accounts`
- `GET /master-data/surveyors`
- `GET /master-data/survey-statuses`
- `GET /master-data/categories/list`
- `GET /master-data/statuses/list`

## Endpoint mutasi super admin

- Kategori kebutuhan: create/update/delete.
- Status lead: create/update/delete/reorder.
- Survey status: create/update/delete/reorder.
- Users: list/create/update/delete/reset password.

## Frontend

- Route: `/master-data`.
- Hook: `useMasterData.ts`.
- Dipakai juga oleh form consultations, survey assignment, settings, filters.

## Risiko perubahan

- Status lead bernama `Sedang Survey`, `Selesai Survey`, dan `Request Survey` dipakai oleh sinkronisasi survey.
- Reorder status memengaruhi tampilan pipeline.
- User role harus konsisten dengan enum backend dan type frontend.

