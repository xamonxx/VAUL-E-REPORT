# Audit and Debug

Terkait: [[12-Security/Security-and-Access]], [[08-Ops/Runbook]], [[15-Backend-Internals/Backend-Internals]].

## Audit logs

- Controller: `AuditLogController`.
- Model: `AuditLog`, trait `Auditable`, observer `AuditObserver`.
- Route:
  - `GET /api/v1/audit-logs`
  - `GET /api/v1/audit-logs/{auditLog}`
  - `GET /api/v1/online-users`
- Akses audit logs: `super_admin`.

## Debug

- Controller: `DebugController`.
- Route prefix: `/api/v1/debug`.
- Endpoint:
  - `GET /stats`
  - `POST /generate-dummy`
  - `POST /clear-dummy`
  - `POST /clear-logs`
- Semua endpoint debug dibatasi `super_admin`.

## Frontend

- Route: `/audit-logs`, `/debug`.
- Hook: `useAuditLogs.ts`, `useDebug.ts`, `useOnlineUsers.ts`.

## Risiko

- `clear-logs` dan `clear-dummy` destructive.
- Debug generate dummy dapat memengaruhi performa dashboard/analytics dan data statistik.

