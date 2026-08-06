# Frontend State and Cache

Terkait: [[14-Frontend-UI/Frontend-Hooks]], [[04-Frontend/Frontend-Map]], [[11-Modules/Notifications]].

## State global

- `authStore.ts`: user aktif, auth state, token lifecycle.
- `sidebarStore.ts`: state sidebar.

## Server state

TanStack Query dipakai untuk:

- List/detail data.
- Pagination/filter server-side.
- Cache dashboard/notifications.
- Invalidation setelah mutation.

## Invalidation penting

- Consultation create/update/delete/status/import: invalidate consultations dan dashboard.
- Note/reminder create/delete/read: invalidate consultation detail dan notifications.
- Survey action: invalidate surveys, survey detail, dashboard, kadang consultations.
- Settings profile/theme: invalidate auth/me atau update local store.
- Master data mutation: invalidate master data list dan form dropdown terkait.

## Risiko

- Query key filter object harus stabil.
- Mutation tanpa invalidation membuat UI stale.
- Optimistic update belum didokumentasikan; hati-hati menambah tanpa rollback.

