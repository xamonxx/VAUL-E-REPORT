# Frontend Map

## Stack

- Next.js App Router, package aktual `next ^15.2.0`.
- React 19.2.4
- TypeScript
- Tailwind CSS v4
- TanStack Query
- Zustand
- React Hook Form + Zod
- Recharts
- Lucide React
- PWA via `@ducanh2912/next-pwa`

## Struktur

- `src/app`: halaman per route.
- `src/components/layout`: shell aplikasi seperti sidebar, header, nav bawah, notification center.
- `src/components/ui`: komponen dasar shadcn/Base UI.
- `src/components/consultations`: komponen khusus konsultasi.
- `src/components/login`: background/login visual.
- `src/lib/api`: API client dan query keys.
- `src/lib/hooks`: hook TanStack Query per domain.
- `src/lib/stores`: Zustand store auth dan sidebar.
- `src/types/index.ts`: tipe domain utama.

## Route aplikasi

- `/login`
- `/dashboard`
- `/consultations`
- `/consultations/create`
- `/consultations/[id]`
- `/consultations/[id]/edit`
- `/surveys`
- `/survey-consumers`
- `/rekap-jadwal-surveyor`
- `/accounts`
- `/analytics`
- `/audit-logs`
- `/master-data`
- `/report-attendances`
- `/bug-reports`
- `/settings`
- `/debug`
- `/offline`

## Auth frontend

- Token key: `e_report_auth_token`.
- API base: `NEXT_PUBLIC_API_URL`, default `http://localhost:8000`.
- Jika browser berjalan di hostname lain, API client mengganti hostname localhost dengan hostname browser dan port backend.
- Semua request API melewati `api` dari `src/lib/api/client.ts`.

## Catatan PWA

- Ada worker di `worker/index.js`.
- Asset manifest dan icon ada di `public`.
- Halaman offline ada di `src/app/offline/page.tsx`.

## Catatan turunan

- [[14-Frontend-UI/UI-and-Navigation]]
- [[14-Frontend-UI/Frontend-Hooks]]
- [[14-Frontend-UI/Frontend-State-and-Cache]]
- [[12-Security/Auth-Flow]]
