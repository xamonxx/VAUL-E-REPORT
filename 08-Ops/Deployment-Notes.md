# Deployment Notes

Terkait: [[08-Ops/Runbook]], [[08-Ops/Production-Server-Spec]], [[12-Security/Security-and-Access]], [[13-Testing/Testing-Guide]].

## Backend

File terkait:

- `render.yaml`
- `deploy.sh`
- `start.sh`
- `scripts/deploy-hostinger.sh`
- `DEPLOYMENT.md`

Hal yang perlu dicek sebelum deploy:

- `.env` production.
- DB credential dan migration.
- Queue worker.
- Cache config.
- Storage symlink bila upload/screenshot/logo dipakai.
- Web push keys.
- CORS/Sanctum domain.

## Frontend

File terkait:

- `.env.production`
- `next.config.js`
- `package.json`
- `public/manifest.json`
- `worker/index.js`

Hal yang perlu dicek:

- `NEXT_PUBLIC_API_URL`.
- Build berhasil.
- PWA asset/icon valid.
- API URL tidak menunjuk localhost.

## Post-deploy smoke

- Login.
- `/auth/me`.
- Dashboard.
- Consultations list.
- Create/edit lead.
- Survey flow minimal.
- Notification count.
- Export file.

## Server target

- Spek production awal yang dicatat: Ubuntu 24.04.4 LTS, Intel Core i3-7100U 2 core / 4 thread, RAM 15 GiB, SSD NVMe 238.5 GB, HDD 465.8 GB, Ethernet aktif, Tailscale aktif.
- Host ini layak untuk production kecil/internal dengan limit worker konservatif dan monitoring CPU/queue/slow query.
- Detail kapasitas dan rekomendasi tuning ada di [[08-Ops/Production-Server-Spec]].
