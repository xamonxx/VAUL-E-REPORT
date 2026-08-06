# Survey

Terkait: [[11-Modules/Consultations]], [[11-Modules/Notifications]], [[12-Security/Role-Matrix]], [[07-Workflows/Core-Workflows]].

## Tujuan

Modul survey mengatur permintaan survey dari lead, penugasan surveyor, jadwal, reschedule, mulai survey, hasil survey, pembatalan, histori, dan rekap jadwal.

## Backend

- Controller: `SurveyController`.
- Model: `Survey`, `SurveyStatus`, `SurveyStatusHistory`, `SurveyActivityLog`, `SurveyNotification`, `SurveyReschedule`.
- Policy: `SurveyPolicy`.
- Report service: `SurveyorScheduleRecapService`, `SurveyorScheduleRecapExcelExporter`.
- Event: `SurveyRealtimeUpdated`.

## State

- `requested`: admin mengajukan survey.
- `scheduled`: manager surveyor menetapkan surveyor dan jadwal.
- `in_progress`: surveyor memulai survey.
- `completed`: surveyor submit hasil.
- `cancelled`: admin, manager surveyor, atau surveyor pemilik jadwal membatalkan survey sebelum selesai.

## API

- `POST /consultations/{consultation}/survey`
- `GET /surveys`
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

## Frontend

- Route: `/surveys`, `/survey-consumers`, `/rekap-jadwal-surveyor`.
- Hook: `useSurveys.ts`, `useSurveyorScheduleRecap.ts`.
- Types: `Survey`, `SurveyState`, `SurveyFilters`, `SurveyActivity`, `SurveyStatusItem`.
- Daftar `/surveys` memakai kartu summary-first: jadwal survey, nama konsumen, WhatsApp, dan tombol Google Maps menjadi informasi utama; data akun, alamat lengkap, seluruh kebutuhan, waktu aktual, serta catatan tersedia lewat dialog detail. Tombol Maps tetap menampilkan state nonaktif bila link belum tersedia.
- Grid daftar responsif: satu kolom pada mobile, dua pada tablet kecil, tiga pada desktop menengah, dan empat kolom compact mulai breakpoint `xl`.
- Untuk state selain `requested`, jadwal utama kartu berasal dari `scheduled_at`. Jadwal pengajuan (`requested_date`/`requested_time`) tetap ditampilkan sebagai pembanding di dialog detail.
- Pencarian daftar memakai filter server `search`; opsi `sort=nearest|latest` mengubah urutan tanpa mengubah bentuk response.

## Role rules

- Super admin bypass.
- Admin dapat mengajukan survey dari lead account-nya. Pada daftar/detail survey operasional dan Data Konsumen Survey, admin hanya melihat dan mengelola survey yang dia ajukan sendiri agar jadwal admin lain tidak terekspos.
- Surveyor melihat dan mengeksekusi survey miliknya.
- Surveyor dapat membatalkan survey yang ditugaskan kepadanya selama belum selesai.
- Manager surveyor melihat semua, assign, reschedule assignment, cancel, availability, recap.

## Sinkronisasi lead

Saat `Survey::transitionTo()`:

- `in_progress` dapat mengubah status consultation menjadi `Sedang Survey`.
- `completed` dapat mengubah status consultation menjadi `Selesai Survey`.
- `cancelled` dari status tertentu bisa mengembalikan consultation ke `Request Survey`.
- Cache dashboard/analytics dihapus atau ditandai refresh.

## Realtime & notifikasi

- Event `SurveyRealtimeUpdated` menulis `SurveyNotification` di constructor, lalu broadcast. Notifikasi DB tetap jalan walau broadcast mati.
- Broadcast memakai **channel privat**: `survey.managers`, `survey.surveyor.{id}`, dan `survey.account.{account_id}`, diotorisasi di `routes/channels.php` (didaftarkan lewat `withBroadcasting()` di `bootstrap/app.php`). Frontend wajib memakai `.private(...)`, bukan `.channel(...)`.
- Penerima notifikasi per aksi: `request_created`/`started` ke manager + super admin; `scheduled`/`rescheduled_by_manager` ke surveyor; `rescheduled_by_admin` ke manager + super admin; `completed` ke pengaju, assigner, dan manager; `cancelled` ke surveyor + pengaju. Parameter `extraRecipients` dipakai saat surveyor lama perlu diberi tahu karena digantikan.
- Realtime memakai Laravel Reverb/Echo bila `BROADCAST_CONNECTION=reverb`, env `NEXT_PUBLIC_REVERB_*` terisi, dan `php artisan reverb:start` berjalan. Jika WebSocket putus, frontend tetap memakai fallback polling ringan: badge count saat tab aktif dan detail panel saat dropdown notifikasi dibuka.

## Alur pengajuan oleh admin

Ada dua pintu masuk, keduanya bermuara ke modal yang sama:

- **Halaman detail** - admin mengubah tahap pipeline lead ke **Request Survey**.
- **Halaman create** - admin membuat lead baru langsung berstatus Request Survey. Setelah tersimpan, admin diarahkan ke `/consultations/{id}?prompt_survey=1`; halaman detail membaca parameter itu, membuka modal, lalu membersihkan query supaya refresh tidak membukanya lagi.

1. Status lead berada di tahap **Request Survey**.
2. Bila lead belum punya survey aktif, modal pengajuan **terbuka otomatis** (`src/components/consultations/survey-request-card.tsx`). Isinya tanggal (wajib), jam, link Google Maps, catatan untuk manager.

Tanggal memakai Popover + `Calendar` dan jam memakai select 24 jam berlabel WIB - **bukan** `<input type="date">`/`type="time"`, yang mengikuti locale browser dan menampilkan `07/23/2026` serta AM/PM di mesin ber-locale Inggris. Link Maps disimpan di `surveys.google_maps_url`; skema `https://` dilengkapi frontend karena tautan Maps kerap disalin tanpa itu.
3. `POST /consultations/{consultation}/survey` membuat survey `requested`, lalu seluruh Manager Surveyor + Super Admin menerima notifikasi in-app **dan** Web Push.
4. Bila request survey sudah telanjur diajukan tanpa link Google Maps, admin akun terkait dapat melengkapi atau memperbaiki `google_maps_url` lewat `PATCH /surveys/{survey}/maps` dari kartu **Status Survey**. Perubahan ini membuat data survey ter-refresh untuk role terkait dan mengirim pembaruan survey.
5. Admin akun terkait juga dapat mengubah **Jadwal Awal Admin** (`requested_date`/`requested_time`) selama survey belum berjalan/selesai/batal lewat `PATCH /surveys/{survey}/reschedule`. Perubahan ini mencatat riwayat reschedule dan mengirim notifikasi ke Manager Surveyor untuk validasi ulang jadwal.
6. Kartu **Status Survey** tetap ada di halaman detail sebagai jaring pengaman bila modal ditutup. UI membedakan **Jadwal Awal Admin** dari **Jadwal Ditetapkan** (`scheduled_at`); jadwal yang belum ada ditampilkan sebagai `-`.

Guard penting: survey **hanya** bisa diajukan ketika status lead = `config('statuses.survey')`. Satu lead hanya boleh punya satu survey aktif - dijaga transaksi + `lockForUpdate` sekaligus unique index `surveys.active_key`.

`active_key` berisi `consultation_id` selama survey aktif dan `NULL` saat cancelled/soft-deleted. Nilainya dijaga `Survey::booted()`, bukan controller - jangan diisi manual.

## Lead yang belum diajukan

`Consultation::scopeNeedsSurveyRequest()` = status lead Request Survey **dan** belum punya survey non-cancelled. Dipakai tiga tempat:

- `pending_survey_requests` pada payload `/notifications` (ikut cache 2 menit)
- filter `GET /consultations?pending_survey=1`
- notifikasi Web Push setelah import selesai

Import **tidak** membuat survey otomatis - tanggal survey belum diketahui, dan manager tidak bisa menjadwalkan tanpa itu. Lead-nya ditandai, admin yang menindaklanjuti.

## Notifikasi

Penerima ditentukan `SurveyRealtimeUpdated::recipientsFor()`, dipakai bersama notifikasi in-app dan Web Push supaya keduanya tidak pernah berbeda. Semua pengiriman lewat `SurveyController::notifySurvey()` - jangan memanggil `SurveyRealtimeUpdated::dispatch()` langsung, nanti push-nya terlewat.

Jalur seketika memakai **Web Push** (`WebPushService`), bukan WebSocket. Interval polling notifikasi sengaja **tidak** dipercepat: polling 3 detik pernah menjatuhkan dashboard karena batas 500 koneksi/jam di shared hosting.

## Aturan state yang mudah terlewat

- Bentrok jadwal dihitung terhadap state `scheduled` **dan** `in_progress`, sama dengan `availability()`.
- Urutan daftar operasional mengikuti state: `requested` memakai FIFO `requested_at`; `scheduled` memakai `scheduled_at` terdekat; `in_progress` memakai `actual_start_at` terlama (fallback `scheduled_at`); `completed` memakai `actual_finish_at` terbaru. `sort=latest` tersedia untuk melihat data yang baru diperbarui.
- `cancel()` bisa dipanggil oleh admin akun terkait, manager surveyor, atau surveyor yang ditugaskan. Endpoint menolak survey yang sudah `completed` maupun yang sudah `cancelled`, lalu mengisi `cancelled_at` + `cancellation_reason`.
- `submitResult()` boleh dipanggil dari `scheduled` (tanpa `start`). Dalam kasus itu `actual_start_at` dibiarkan NULL supaya durasi tidak tercatat 0 menit.
- `store()` tidak mengisi `state` saat create; kolomnya `default('requested')` sehingga `transitionTo()` bisa menulis history awal `null -> requested`.

## Risiko perubahan

- Jangan ubah state string tanpa update frontend type, policy, UI badge, filter, migration/seed status, dan notification.
- Assignment logic memengaruhi jadwal surveyor dan recap export.
