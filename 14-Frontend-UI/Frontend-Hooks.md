# Frontend Hooks

Terkait: [[04-Frontend/Frontend-Map]], [[14-Frontend-UI/Frontend-State-and-Cache]], [[06-API/API-Catalog]].

## Hook per domain

- `useAuth.ts`: login/logout/me.
- `useAccounts.ts`: account CRUD.
- `useAnalytics.ts`: analytics data.
- `useAuditLogs.ts`: audit logs.
- `useBugReports.ts`: bug report management.
- `useReportBug.ts`: public bug report submit.
- `useConsultations.ts`: lead CRUD, import, notes, reminders.
- `useDashboard.ts`: dashboard stats.
- `useDebug.ts`: debug tools.
- `useMasterData.ts`: categories, statuses, users, surveyors.
- `useNotifications.ts`: notification count/summary/read/clear.
- `useOnlineUsers.ts`: presence.
- `useReportAttendances.ts`: attendance.
- `useSettings.ts`: profile/theme.
- `useSurveys.ts`: survey list/detail/actions.
- `useSurveyorScheduleRecap.ts`: recap jadwal surveyor.

## Pattern umum

- Query key berasal dari `src/lib/api/queryKeys.ts`.
- Mutation harus invalidate query terkait.
- List query memakai filter object sebagai bagian query key.
- Detail query `enabled` jika id valid.

## Saat menambah endpoint

1. Tambah type di `src/types/index.ts`.
2. Tambah query key jika perlu.
3. Tambah hook domain.
4. Update UI route/component.
5. Update [[06-API/API-Catalog]] dan modul terkait.

