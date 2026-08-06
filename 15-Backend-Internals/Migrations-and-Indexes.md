# Migrations and Indexes

Terkait: [[05-Database/Data-Model]], [[11-Modules/Consultations]], [[11-Modules/Survey]].

## Kelompok migration

- Base tables: accounts, users, needs categories, status categories, consultations, notes, reminders.
- Auth: personal access tokens, login attempts index.
- Performance: search indexes, date indexes, compound indexes.
- Consultation evolution: district/address, product details, pivot needs categories, sequences, imports, soft deletes.
- Attendance/report: report attendances dan category.
- Notifications: push subscriptions, reminder pushed at.
- Survey: role widening, survey statuses, surveys, histories, workflow expansion, activity logs, notifications, reschedules, actual timestamps.
- Security audit improvements.

## Index penting

- Consultation search/filter indexes.
- Date indexes untuk consultation.
- Compound indexes untuk query dashboard/list.
- Login attempts lookup index.

## Saat ubah schema

1. Buat migration baru, jangan edit migration lama kecuali masih belum dipakai.
2. Update model fillable/casts/relationships.
3. Update request validation.
4. Update frontend types/forms.
5. Update [[05-Database/Data-Model]] dan modul terkait.

