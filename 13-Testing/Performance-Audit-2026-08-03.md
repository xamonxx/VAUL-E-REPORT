# Performance Audit 2026-08-03

Tanggal: 3 Agustus 2026  
Baseline: working tree saat audit  
Frontend: Next.js 15.5.22, React 19.2.4  
Backend: Laravel 12.64, PHP 8.5 built-in server lokal, database lokal  
Harness: Playwright Chromium, desktop 1440x900, Android 412x915, 3 run per route/viewport

## Verdict

**Baseline UI sehat untuk shell publik, tetapi audit authenticated belum lengkap.** LCP lokal berada di bawah 500 ms dan CLS 0. Temuan utama adalah transfer awal sekitar 433-519 KB dan chunk Geo Analytics sekitar 970 KB uncompressed. Angka API lokal sekitar 240 ms dipengaruhi `php -S`, sehingga belum menjadi SLA produksi.

## Hasil Browser

| Skenario | Desktop | Android | Catatan |
|---|---:|---:|---|
| `/login` transfer | 432.7 KB | 432.7 KB | P50 dan P95 sama pada cache-disabled context |
| `/login` LCP | 272 ms P50, 356 ms P95 | 216 ms P50, 232 ms P95 | CLS 0 |
| Protected shell transfer | 437.8-437.9 KB | 437.8-437.9 KB | Route unauthenticated kembali ke `/login` |
| Protected shell LCP | 304-396 ms P50 | 260-272 ms P50 | Bukan authenticated data view |
| `/consultations/1` transfer | 518.5 KB | 518.5 KB | Route chunk ikut terprefetch sebelum redirect |
| HTTP 5xx / console error | 0 | 0 | Tidak ada pada run audit |

`wall_ms` harness sengaja menunggu 1.5 detik untuk settle state dan tidak dipakai sebagai metrik first paint.

## Backend

- Dashboard superadmin cold cache pada fixture lokal: 14 query, 22.69 ms total DB time, payload 6.7 KB.
- Analytics dengan `includeRawRows=true`: 32 query, 38.46 ms total DB time pada dataset lokal.
- CSRF cookie: rata-rata 241.6 ms, p95 262.7 ms, 20 request.
- `/api/v1/auth/me` tanpa sesi: rata-rata 240.5 ms, p95 263 ms, 20 request.
- Dua angka API terakhir berasal dari PHP built-in server lokal dan tidak mewakili PHP-FPM/queue/cache produksi.
- Dashboard dan analytics memakai cache; relasi dashboard utama eager-loaded; endpoint list memakai pagination.

## Temuan

### PERF-2026-01 - Transfer awal private shell cukup besar

- Severity: Medium
- Bukti: protected route unauthenticated tetap mengirim sekitar 438 KB sebelum client auth guard mengarah ke login; login sendiri sekitar 433 KB.
- Dampak: first visit perangkat Android dan jaringan lambat membayar download route private walau sesi tidak valid.
- Perbaikan: tersedia auth gate server-side opt-in berbasis validasi cookie Sanctum melalui middleware, dengan proxy API same-origin untuk production.
- Status: Implemented opt-in. Belum aktif pada production sampai env frontend/API dan domain cookie tervalidasi di staging.

### PERF-2026-02 - Geo Analytics membawa chunk MapLibre besar

- Severity: Medium
- Bukti: chunk MapLibre sekitar 970 KB uncompressed pada `.next/static/chunks/05f6971a...js`; GeoMap sudah `dynamic()` tetapi tetap harus diukur pada authenticated route.
- Dampak: masuk Geo Analytics pada device rendah dapat menaikkan parse/compile time dan konsumsi memori.
- Rekomendasi: pertahankan lazy boundary, tambahkan load-on-interaction/viewport, ukur gzip/brotli production, dan pertimbangkan peta lebih ringan bila fungsi tidak memerlukan MapLibre penuh.
- Status: Open.

### PERF-2026-03 - Font dan shared chunk mendominasi request awal

- Severity: Low
- Bukti: request terbesar login mencakup chunk 54.7 KB, font 48.7 KB, chunk 48.6 KB, chunk 46.2 KB, dan CSS 43.7 KB transfer.
- Rekomendasi: audit subset/weight font, pastikan hanya weight yang dipakai, dan ukur ulang setelah optimasi.
- Status: Open.

### PERF-2026-04 - Authenticated data flow belum terukur

- Severity: Medium / Measurement gap
- Bukti: fixture credential browser valid belum tersedia; protected route pada audit ini redirect ke login sehingga latency dashboard, analytics, notification, report attendance, dan query data nyata belum terukur dari browser.
- Tindakan: jalankan harness dengan akun sintetis superadmin/admin pada database audit, ukur login, `/dashboard`, `/analytics`, `/notifications`, `/report-attendances`, dan navigation tab.
- Status: Manual pending.

## Yang Sudah Baik

- LCP lokal berada di bawah 500 ms pada semua shell yang diukur.
- CLS 0 pada desktop dan Android.
- Long task hanya sporadis dan pendek pada login desktop; tidak ada error 5xx.
- Route halaman besar sudah banyak memakai `dynamic()`; MapLibre tidak masuk shared route chunk.
- Dashboard cache 5-15 menit dan analytics cache 3 menit mengurangi query berulang.
- Query list utama memakai `paginate()` dan relasi dashboard memakai eager loading.

## Quick Wins Diterapkan

- Font mono tidak lagi dipreload pada first paint. Remeasure menunjukkan transfer awal hanya berubah sekitar 23 byte, jadi ini optimasi kecil, bukan pengurang bundle utama.
- Master-data kategori/status sekarang memakai cache 5 menit; mutation tetap meng-invalidate key sehingga perubahan admin tetap muncul tanpa refetch setiap mount/focus.
- Notification summary tidak lagi refetch pada setiap window focus; membuka panel tetap mengaktifkan query dan polling fallback berjalan sesuai aktivitas user.
- Build, lint, browser smoke, dan backend security suite tetap lulus setelah perubahan.

## Rekomendasi Urutan

1. Jalankan authenticated benchmark dengan fixture sintetis valid.
2. Profilkan Geo Analytics di device Android nyata atau emulasi CPU/network throttling.
3. Aktifkan dan uji auth gate server-side pada staging; lanjutkan bundling boundary privat bila transfer authenticated masih tinggi.
4. Audit font weight/subset dan ukur bundle gzip/brotli di CDN.
5. Ukur API production-safe melalui PHP-FPM/queue worker dengan p50/p95/p99 dan error rate.

## Artefak

- Harness: `E-REPORT-FRONTEND/scripts/performance_audit.py`
- Command: `npm run test:performance`
- Data: `E-REPORT-FRONTEND/artifacts/performance-audit.json` (lokal, di-ignore Git)
- Build: `npm run build` lulus; shared First Load JS 106 KB, dashboard 154 KB, detail konsultasi 306 KB, login 216 KB.
