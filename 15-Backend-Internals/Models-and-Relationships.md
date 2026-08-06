# Models and Relationships

Terkait: [[05-Database/Data-Model]], [[11-Modules/Consultations]], [[11-Modules/Survey]].

## User

- Belongs to Account.
- Has many consultations as creator.
- Has many report attendances.
- Has many consultation notes.
- Has many assigned surveys.
- Role cast ke `UserRole` enum.

## Account

- Has many users.
- Has many admins.
- Has many consultations.
- Menghitung lead count, conversion rate, target progress.

## Consultation

- Belongs to Account.
- Belongs to NeedsCategory legacy.
- Belongs to many NeedsCategory melalui pivot.
- Belongs to StatusCategory.
- Belongs to creator User.
- Has many timeline notes.
- Has many reminders.
- Has many surveys.
- Has many status histories.

## Survey

- Belongs to Consultation.
- Belongs to User sebagai surveyor.
- Belongs to User sebagai assigner.
- Belongs to User sebagai requester.
- Belongs to SurveyStatus sebagai result status.
- Has many histories.
- Has many reschedules.
- Has many activity logs.

## Notification-related

- Reminder belongs to consultation dan user.
- ConsultationNote belongs to consultation dan user.
- SurveyNotification references survey/consultation context.
- PushSubscription stores browser subscription.

