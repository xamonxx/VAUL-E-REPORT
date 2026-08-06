# API Catalog

Base path: `/api/v1`.

## Public

- `POST /auth/login`: login, throttle 10/min.
- `GET /auth/color-lookup`: lookup warna login by email, throttle 20/min.
- `POST /bug-reports`: submit bug publik, throttle 5/min.

## Auth

- `GET /auth/me`
- `POST /auth/logout`
- `POST /settings/profile`
- `POST /settings/theme`

## Dashboard dan analytics

- `GET /dashboard`: role admin.
- `GET /analytics`: role admin.

## Consultations

- `GET /consultations`
- `POST /consultations`
- `GET /consultations/{id}`
- `PUT/PATCH /consultations/{id}`
- `DELETE /consultations/{id}`
- `GET /consultations/id-preview`
- `GET /consultations/import/template`
- `POST /consultations/import`
- `PATCH /consultations/{id}/status`
- `POST /consultations/{id}/notes`
- `DELETE /consultations/{id}/notes/{note}`
- `POST /consultations/{id}/reminders`
- `DELETE /consultations/{id}/reminders/{reminder}`

## Survey

- `POST /consultations/{consultation}/survey`
- `GET /surveys`: mendukung `state`, `search`, `account`, `surveyor_id`, rentang tanggal, dan `sort=nearest|latest`. Urutan default mengikuti state: request FIFO, terjadwal berdasarkan jadwal terdekat, berlangsung berdasarkan waktu mulai terlama, dan selesai berdasarkan waktu selesai terbaru.
- `GET /surveys/{survey}`
- `GET /surveys/{survey}/history`
- `GET /surveys/recap`
- `GET /surveys/availability`
- `PATCH /surveys/{survey}/assign`
- `PATCH /surveys/{survey}/reschedule`
- `PATCH /surveys/{survey}/maps`
- `PATCH /surveys/{survey}/reschedule-assignment`
- `PATCH /surveys/{survey}/start`
- `PATCH /surveys/{survey}/result`
- `PATCH /surveys/{survey}/cancel`

## Master data

- `GET /master-data/needs-categories`
- `GET /master-data/status-categories`
- `GET /master-data/accounts`
- `GET /master-data/surveyors`
- `GET /master-data/survey-statuses`
- `GET /master-data/categories/list`
- `GET /master-data/statuses/list`
- Mutasi kategori/status/survey-status/users: `super_admin`.
- `apiResource /accounts`: `super_admin`.

## Report dan export

- `GET /report-attendances`
- `POST /report-attendances`
- `POST /report-attendances/upsert-by-super-admin`
- `GET /report-attendances/export`
- `GET /export/csv`
- `GET /export/leads/excel`
- `GET /export/leads/pdf`
- `GET /export/analytics/excel`
- `GET /export/analytics/pdf`
- `GET /export/surveys/recap/excel`: role `manager_surveyor`.

## Notifications

- `GET /notifications`
- `GET /notifications/summary`
- `PATCH /notifications/notes/{note}/read`
- `PATCH /notifications/reminders/{reminder}/read`
- `PATCH /notifications/surveys/{notification}/read`
- `DELETE /notifications/surveys/{notification}`
- `PATCH /notifications/attendances/{notification}/read`
- `DELETE /notifications/attendances/{notification}`
- `DELETE /notifications/clear`

## Maintenance

- `GET /audit-logs`
- `GET /audit-logs/{auditLog}`
- `GET /online-users`
- `GET /wilayah/provinces`
- `GET /wilayah/cities`
- `GET /wilayah/districts`
- `GET /debug/stats`: `super_admin`
- `POST /debug/generate-dummy`: `super_admin`
- `POST /debug/clear-dummy`: `super_admin`
- `POST /debug/clear-logs`: `super_admin`
