# Policies

Terkait: [[12-Security/Role-Matrix]], [[11-Modules/Consultations]], [[11-Modules/Survey]].

## Policy files

- `ConsultationPolicy.php`
- `ConsultationNotePolicy.php`
- `ReminderPolicy.php`
- `SurveyPolicy.php`

## Pattern umum

- `before()` memberi bypass untuk `super_admin`.
- Admin biasanya dibatasi `account_id`.
- Surveyor dibatasi assignment ke dirinya.
- Manager surveyor punya authority untuk penjadwalan survey.

## Consultation policy

- Admin dapat view/create/update/delete lead hanya untuk account sendiri.
- Admin dapat add note/reminder hanya untuk consultation account sendiri.

## Survey policy

- `viewAny`: survey team atau admin.
- `view`: manager semua, surveyor miliknya, admin account sendiri.
- `assign`: manager surveyor.
- `reschedule`: admin account sendiri.
- `rescheduleAssignment`: manager surveyor.
- `start`: manager atau surveyor yang ditugaskan.
- `submitResult`: manager atau surveyor yang ditugaskan.
- `cancel`: manager surveyor.

## Saat ubah policy

- Update [[12-Security/Role-Matrix]].
- Test 403/allowed path.
- Cek UI menu/route guard supaya tidak menampilkan aksi yang akan ditolak backend.

