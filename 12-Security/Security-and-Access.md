# Security and Access

Terkait: [[12-Security/Auth-Flow]], [[12-Security/Role-Matrix]], [[15-Backend-Internals/Policies]], [[11-Modules/Auth-and-Users]].

> Audit terakhir: [[12-Security/Pentest-Audit-2026-07-31]] — audit kode + verifikasi produksi live + pentest aktif lokal (8 vektor). Nihil kerentanan; sisa 1 tindak lanjut (HSTS di Cloudflare).

## Lapisan keamanan

- Authentication: `auth:sanctum`.
- Authorization route: `RoleMiddleware`.
- Authorization object: policies.
- Validation: Form Request classes.
- Throttle: login, color lookup, public bug report.
- Mass assignment protection: `User::$fillable` tidak memuat `role`.
- Data scoping: admin hanya account sendiri; data survey operasional admin dibatasi lagi ke survey yang dia ajukan sendiri.
- Sensitive endpoints: debug dan audit hanya `super_admin`.

## Middleware penting

- `RoleMiddleware`: super admin bypass; role lain harus match role route.
- `SecurityHeaders`: header keamanan HTTP.
- `UpdateLastSeen`: presence user.
- `QueryTokenToHeader`: membantu auth token dari query untuk download/export.
- `PreventBackHistory`: mencegah navigasi balik setelah logout pada area tertentu.

## Area rawan

- Token di localStorage rentan XSS; jaga semua rendering user-generated content.
- Export URL bisa membawa token query.
- Public bug report upload harus tetap throttle dan validasi ketat.
- Debug endpoint destructive.
