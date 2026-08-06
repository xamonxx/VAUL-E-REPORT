# Auth and Users

Terkait: [[12-Security/Auth-Flow]], [[12-Security/Role-Matrix]], [[15-Backend-Internals/Models-and-Relationships]], [[04-Frontend/Frontend-Map]].

## Tujuan

Mengelola identitas user, login, logout, session/token, profil, warna tema, dan role.

## Backend

- Controller: `AuthController`, `SettingsController`, sebagian `MasterDataController`.
- Model: `User`, `LoginAttempt`, `Account`.
- Enum: `App\Enums\UserRole`.
- Middleware: `auth:sanctum`, `RoleMiddleware`, `UpdateLastSeen`.
- Route:
  - `POST /api/v1/auth/login`
  - `GET /api/v1/auth/color-lookup`
  - `GET /api/v1/auth/me`
  - `POST /api/v1/auth/logout`
  - `POST /api/v1/settings/profile`
  - `POST /api/v1/settings/theme`
  - `GET/POST/PUT/DELETE /api/v1/master-data/users`

## Frontend

- Store: `src/lib/stores/authStore.ts`.
- Hook: `src/lib/hooks/useAuth.ts`, `useSettings.ts`, `useMasterData.ts`.
- API client menyimpan token di `localStorage` key `e_report_auth_token`.
- Route UI: `/login`, `/settings`, `/master-data`.

## Role

- `super_admin`: bypass role middleware, kelola users.
- `admin`: akses lead, dashboard, report.
- `surveyor`: akses survey miliknya.
- `manager_surveyor`: akses jadwal, assignment, rekap surveyor.

## Provisioning Surveyor

- Seeder operasional: `database/seeders/SurveyorUserSeeder.php`.
- Email surveyor memakai pola `{nama}-srvy@npp.com`.
- Spasi pada nama dinormalisasi menjadi titik, misalnya `INDRA DADO` menjadi `indra.dado-srvy@npp.com`.
- Akun surveyor tidak terikat `account_id` agar dapat menerima penugasan lintas akun.
- Password awal diberikan saat seeder dijalankan melalui environment sementara `SEED_SURVEYOR_PASSWORD`; nilainya tidak disimpan di source code maupun vault.
- Seeder memigrasikan email format lama dan mengaktifkan kembali akun target yang sebelumnya dihapus secara lunak tanpa membuat akun duplikat.

## Hal yang rawan

- `role` tidak ada di `$fillable` model `User` untuk mencegah privilege escalation mass assignment.
- Dokumentasi auth lama menyebut cookie Sanctum, tetapi client aktif memakai Bearer token.
- Jangan menyimpan password awal surveyor di Git, changelog, atau catatan Obsidian.
- Perubahan auth wajib update [[12-Security/Auth-Flow]] dan [[12-Security/Role-Matrix]].
