# Auth Flow

Terkait: [[11-Modules/Auth-and-Users]], [[02-Architecture/System-Architecture]], [[04-Frontend/Frontend-Map]].

## Flow aktual dari kode frontend

1. User login di `/login`.
2. Frontend memanggil `POST /api/v1/auth/login`.
3. Backend mengembalikan token atau sesi yang diterima client.
4. Frontend menyimpan token di `localStorage` dengan key `e_report_auth_token`.
5. API client menambahkan `Authorization: Bearer <token>`.
6. API client juga mengirim `credentials: include`.
7. Backend route memakai `auth:sanctum`.
8. Frontend memanggil `/auth/me` untuk profil aktif.

## Kompatibilitas lama

`getCsrfCookie()` masih ada di `client.ts`, tetapi komentar kode menyebut dipertahankan untuk backward compatibility.

## Logout

- Frontend memanggil `/auth/logout`.
- Token localStorage harus dihapus.
- Auth store dibersihkan.

## Hal yang harus dicek saat ubah auth

- `AuthController`
- `LoginRequest`
- `src/lib/api/client.ts`
- `src/lib/stores/authStore.ts`
- `src/lib/hooks/useAuth.ts`
- `AuthGuard`
- CORS/Sanctum config backend

