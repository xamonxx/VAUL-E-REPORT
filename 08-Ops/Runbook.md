# Runbook

## Menjalankan backend lokal

```bash
cd C:\laragon\www\E-REPOR_v0.1\E-REPORT-BACKEND_API
composer install
php artisan migrate --seed
php artisan serve
php artisan queue:work
```

## Menjalankan frontend lokal

```bash
cd C:\laragon\www\E-REPOR_v0.1\E-REPORT-FRONTEND
npm install
npm run dev
```

Default:

- Backend: `http://localhost:8000`
- Frontend: `http://localhost:3000`

## Environment penting

Backend:

- `APP_URL`
- `FRONTEND_URL`
- `SANCTUM_STATEFUL_DOMAINS`
- `DB_CONNECTION`, `DB_HOST`, `DB_PORT`, `DB_DATABASE`, `DB_USERNAME`, `DB_PASSWORD`
- `QUEUE_CONNECTION`
- `CACHE_STORE`
- Web push keys bila push notification aktif.

Frontend:

- `NEXT_PUBLIC_API_URL`
- `NEXT_PUBLIC_APP_NAME`

## Verifikasi cepat

Backend:

```bash
php artisan route:list
php artisan migrate:status
```

Frontend:

```bash
npm run lint
npm run build
```

## Risiko operasional

- Debug endpoints dapat membuat/menghapus data dummy dan clear logs; hanya untuk `super_admin`.
- Import lead membutuhkan queue worker agar tidak macet.
- Export file dapat memakai token query via `buildExportUrl`; hati-hati share URL export.
- Dokumentasi versi Next harus mengikuti package aktual, bukan asumsi README lama.

## Obsidian

- Styling modern vault: [[08-Ops/Obsidian-Modern-Color-Setup]]

## Server production

- Baseline spek server dan rekomendasi sizing: [[08-Ops/Production-Server-Spec]]
