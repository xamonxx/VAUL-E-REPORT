# Notifications

Terkait: [[11-Modules/Consultations]], [[11-Modules/Survey]], [[12-Security/Security-and-Access]].

## Tujuan

Menampilkan dan mengirim notifikasi untuk notes, reminders, survey, dan absensi admin. Mendukung notification center dan web push.

## Backend

- Controller: `NotificationController`, `PushSubscriptionController`.
- Service: `NotificationSummaryService`, `WebPushService`.
- Event realtime catatan: `ConsultationNoteCreated`.
- Model: `ConsultationNote`, `Reminder`, `SurveyNotification`, `AttendanceNotification`, `PushSubscription`.
- Command: `PushDueReminders`.

## API

- `GET /notifications`
- `GET /notifications/summary`
- `PATCH /notifications/notes/{note}/read`
- `PATCH /notifications/reminders/{reminder}/read`
- `PATCH /notifications/surveys/{notification}/read`
- `DELETE /notifications/surveys/{notification}`
- `PATCH /notifications/attendances/{notification}/read`
- `DELETE /notifications/attendances/{notification}`
- `DELETE /notifications/clear`
- `GET /push/public-key`
- `POST /push/subscribe`
- `POST /push/unsubscribe`

## Frontend

- Component: `notification-center.tsx`, `push-toggle.tsx`.
- Hook: `useNotifications.ts`.
- PWA support: manifest, icons, worker.
- Catatan baru diterima lewat private channel `consultation-notes.user.{userId}`. Listener memperbarui badge, detail konsultasi, dan menampilkan toast saat aplikasi terbuka.
- Event `consultation-notes.changed` pada channel yang sama menyinkronkan edit, hapus pilihan, dan bersihkan percakapan tanpa mengirim toast baru.
- Saat aplikasi/PWA tertutup, `WebPushService` mengirim payload ke service worker. Pengguna harus menekan `Aktifkan HP` satu kali pada setiap perangkat dan memberi izin notifikasi.
- Saat admin mengirim absensi harian, backend membuat `attendance_notifications` dan Web Push hanya untuk super admin. Payload berisi nama admin, akun, tanggal, dan kategori absensi.

## Akses catatan

- Penerima hanya super admin dan admin dengan `account_id` yang sama dengan konsultasi, selain pembuat catatan.
- Surveyor, manager surveyor, dan admin akun lain mendapat hitungan catatan `0`, daftar kosong, tidak dapat membuka channel catatan, dan tidak menjadi penerima Web Push.
- Notifikasi absensi hanya untuk role `super_admin`; role lain mendapat hitungan absensi `0`, daftar kosong, dan ditolak saat mencoba read/delete notifikasi absensi.
- Channel privat dibuat per user, bukan per akun, agar payload isi catatan tidak tersebar ke subscriber lain.
- Logout mencabut langganan push perangkat untuk mencegah pesan akun lama muncul pada perangkat bersama.

## Cache

`NotificationSummaryService` cache:

- Detail: `api_notif:v2:detail:{userId}`
- Counts: `api_notif:v2:counts:{userId}`
- TTL: 2 menit

## Risiko perubahan

- Jika menambah jenis notifikasi baru, update backend summary, frontend type `NotificationSummary`, UI notification center, dan clear/read endpoint.
- Push notification perlu key dan subscription valid di environment.
