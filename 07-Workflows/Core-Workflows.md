# Core Workflows

## Login

```mermaid
sequenceDiagram
  participant U as User
  participant FE as Frontend
  participant API as Laravel API
  U->>FE: Isi login
  FE->>API: POST /api/v1/auth/login
  API-->>FE: user + token/session
  FE->>FE: Simpan e_report_auth_token
  FE->>API: GET /api/v1/auth/me
  API-->>FE: AuthUser
```

## Lead/konsultasi

1. Admin membuat lead lewat `/consultations/create` atau import.
2. Backend validasi lewat `ConsultationRequest`.
3. Backend generate `consultation_id` format `AA.YYMM.NNNN`.
4. Lead punya account, status, kebutuhan, lokasi, detail produk, catatan, reminders.
5. Status dapat diubah manual atau tersinkron oleh workflow survey.

## Import lead

1. User download template.
2. User upload file import.
3. Backend mencatat `consultation_imports`.
4. Job `ProcessConsultationImportJob` memproses data via queue.
5. Duplicate detection memakai nomor telepon ternormalisasi atau profile key.

## Survey

```mermaid
stateDiagram-v2
  [*] --> requested
  requested --> scheduled: assign / reschedule assignment
  scheduled --> in_progress: start
  in_progress --> completed: submit result
  requested --> cancelled: cancel
  scheduled --> cancelled: cancel
  in_progress --> cancelled: cancel
```

Catatan:

- Transition ditulis ke `survey_status_histories` dan `survey_activity_logs`.
- State `in_progress` dapat menyetel status lead ke `Sedang Survey`.
- State `completed` dapat menyetel status lead ke `Selesai Survey`.
- Cancel dari status survey tertentu dapat mengembalikan lead ke `Request Survey`.

## Notifications

- Notes, reminders, dan survey notifications muncul lewat `/notifications/summary`.
- Web push memakai `PushSubscriptionController` dan `WebPushService`.
- Reminder due dapat diproses lewat command `PushDueReminders`.

## Detail modul

- [[11-Modules/Auth-and-Users]]
- [[11-Modules/Consultations]]
- [[11-Modules/Survey]]
- [[11-Modules/Notifications]]
- [[11-Modules/Reporting-and-Analytics]]
