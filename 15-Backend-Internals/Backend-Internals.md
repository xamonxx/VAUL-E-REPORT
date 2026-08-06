# Backend Internals

Terkait: [[03-Backend/Backend-Map]], [[15-Backend-Internals/Models-and-Relationships]], [[15-Backend-Internals/Services]], [[15-Backend-Internals/Policies]], [[15-Backend-Internals/Migrations-and-Indexes]].

## Entry point

- HTTP API: `routes/api.php`.
- App bootstrap: `bootstrap/app.php`.
- Providers: `bootstrap/providers.php`, `app/Providers/AppServiceProvider.php`.
- Public index: `public/index.php`.

## Layering

1. Route menerima request dan middleware.
2. Controller mengatur alur endpoint.
3. Form Request memvalidasi input.
4. Policy/middleware membatasi akses.
5. Service menampung proses domain/report/import.
6. Model mengelola persistence dan relationship.
7. Resource/response mengembalikan JSON atau file.

## Cross-cutting backend

- Audit: trait/model/observer.
- Cache: dashboard, analytics, notifications, schema check.
- Queue: import consultation.
- Export: PDF/Excel/CSV.
- Notification: web push dan Telegram.

