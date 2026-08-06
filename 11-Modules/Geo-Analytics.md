# Geo Analytics

> **STATUS: FASE 1 AKTIF** (2026-07-25). Choropleth per provinsi berjalan. Fitur berbasis titik (marker/heatmap/radius/polygon/territory/AI) tetap di luar cakupan — butuh koordinat yang belum ada.
>
> Route: `GET /api/v1/geo-analytics`. Halaman: `/geo-analytics` (role admin, super_admin lolos).

Terkait: [[11-Modules/Wilayah]], [[11-Modules/Reporting-and-Analytics]], [[11-Modules/Consultations]], [[09-Decisions/Decision-Log]].

## Tujuan

Modul Business Intelligence geografis: menjawab wilayah mana paling banyak konsultasi/survey/deal, wilayah mana belum tersentuh, produk apa paling diminati per wilayah, dan akun mana paling produktif.

Di luar cakupan secara eksplisit: omset, revenue, profit, margin, harga, nilai proyek. Modul ini murni operasional dan geografis.

## Temuan penting sebelum merancang

Spesifikasi awal mengasumsikan skema yang **tidak ada** di aplikasi ini. Hasil pemeriksaan langsung ke database:

| Spek mengasumsikan | Kenyataan |
|---|---|
| `customers` dengan `latitude`/`longitude` | `consultations` — **nol kolom koordinat** di seluruh 30 tabel |
| `marketing`, `products`, `activity_logs` | `users`(role admin)/`accounts`, `needs_categories`, `consultation_status_histories` |
| Prisma ORM | Laravel Eloquent |
| kecamatan + kelurahan terisi | `consultations.district` = **100% "Belum konfirmasi"**, kolom kelurahan tidak ada |
| 100.000+ konsumen | 268 konsultasi |
| MapLibre + Supercluster + Turf + Deck.GL | belum satu pun terpasang |

Kualitas kolom lokasi `consultations` (268 baris, per 2026-07-24):

| Kolom | Terisi | Nilai unik | Placeholder "Belum konfirmasi" |
|---|---|---|---|
| `province` | 268/268 | 16 | 61 (23%) |
| `city` | 268/268 | 47 | 84 (31%) |
| `district` | 268/268 | **1** | 268 (100%) |
| `address` | **0/268** | 0 | - |

Catatan penting: data referensi wilayah **sudah lengkap** di aplikasi (38 provinsi, 514 kota, 7283 kecamatan — lihat [[11-Modules/Wilayah]]). Yang kosong adalah **pengisian** di form konsultasi, bukan datanya. `district` dan `address` adalah celah input, bukan celah data master.

Konsekuensi: peta bermarker (cluster, heatmap, radius, polygon, territory) mustahil tanpa lebih dulu membangun pipeline koordinat.

## Keputusan Fase 1

| Hal | Keputusan |
|---|---|
| Bentuk peta | **Choropleth** (batas wilayah diwarnai), bukan marker |
| Cakupan | Peta, KPI, Sidebar Filter, Ranking Wilayah, Popup detail |
| "Marketing" | **Akun/cabang** — `created_by` tidak andal (lihat di bawah) |
| Level wilayah | **Provinsi + Kota saja** |
| Polygon | Hanya provinsi; kota tampil sebagai daftar peringkat |
| Dependensi baru | Hanya `maplibre-gl`. Supercluster/Turf/Deck.GL tidak dipasang |
| Basemap | Tanpa tile eksternal (hindari API key); polygon di atas latar bertema |

### Kenapa "marketing" = akun, bukan admin

Dari 268 konsultasi, `created_by` terisi penuh (268/268) tapi **nol** di antaranya milik admin dari akun yang sama. Ranking per admin akan tampil kosong atau salah. Ranking per akun akurat karena konsultasi memang terikat `account_id`.

### Kenapa KPI Pending/Cancel tidak disalin mentah

`config/statuses.php` memetakan `pending` ke alias `Pending`/`Masuk`/`Baru` dan `cancelled` ke `Batal`/`Cancelled`. **Tidak satu pun nama itu ada** di tabel `status_categories`. Akibatnya dua KPI ini di Dashboard sekarang **selalu 0 tanpa disadari** — lihat [[09-Decisions/Decision-Log]].

Status yang benar-benar ada (10): Hanya Tanya Tanya, Request Survey, Kendala Anggaran, Tidak Ada Respon, Selesai/Deal, Masih konsultasi, Nunggu Bangunan, Perbandingan Harga, Sedang Survey, Selesai Survey.

## Implementasi aktual (Fase 1)

File yang dibuat:

- `app/Support/ConsultationStatusGroups.php` — satu sumber `surveyIds/dealIds/pendingIds/cancelledIds`, dibaca dari `config/statuses.php`. `AnalyticsReportService` dan `DashboardService` sekarang delegasi ke sini.
- `app/Support/RegionNameMatcher.php` — `matchProvince()`, `provinceForCity()`, `missingProvinces()`. Acuan dari `config/wilayah.php` + `resources/data/wilayah_kota.php`. Alias: `dki jakarta`/`di yogyakarta`/`bangka belitung`.
- `app/Services/Geo/GeoAnalyticsService.php` — agregasi GROUP BY: kpi, statusBreakdown, provinces, cities, accountRanking, productByRegion, unmatchedRegions, unlocatedProvince. Grup terpilih diteruskan lewat `$filters` (bukan property instance seperti AnalyticsReportService).
- `app/Http/Requests/GeoAnalyticsRequest.php`, `app/Http/Controllers/Api/GeoAnalyticsController.php`.
- Frontend: `src/app/geo-analytics/**`, `src/lib/hooks/useGeoAnalytics.ts`, `public/geo/idn-provinces.geojson` (34 provinsi, 328 KB, `feature.id` = region_id kanonik).

### Peta: MapLibre GL JS (per 2026-07-26)

Awalnya SVG choropleth (karena MapLibre worker mati di Turbopack). Atas permintaan user, dipindah ke **MapLibre GL JS** (`maplibre-gl@5.16.0`) untuk basemap peta jalan + pin marker bernomor. `geo-map.tsx` sekarang: basemap raster OSM (toggle), source GeoJSON `kabkota` + `provinces` (`promoteId: region_id`), layer `fill` kabupaten (warna dari `feature-state` join `cities`) + `line` batas provinsi, `NavigationControl` + `FullscreenControl`, dan **marker bernomor berjenjang** (lihat bagian Marker di bawah).

**WAJIB Webpack, bukan Turbopack.** Turbopack men-drop Web Worker MapLibre (bug diketahui: [vercel/next.js#86495](https://github.com/vercel/next.js/issues/86495)) — peta hang, GeoJSON tak parse. Karena itu script `dev` di `package.json` diubah dari `next dev --turbopack` ke `next dev` (Webpack); `dev:turbo` disediakan untuk yang butuh turbopack (tapi halaman peta akan rusak di sana). `next build` sudah Webpack, jadi **produksi aman** — dibuktikan `npm run build` sukses (22/22 halaman, `/geo-analytics` 2.23 kB, MapLibre lazy).

`maplibre-gl@5.16.0` dipin (bukan 5.17) karena 5.17.0 punya bug worker `_define_property` ([maplibre-gl-js#7105](https://github.com/maplibre/maplibre-gl-js/issues/7105)). `@turf/centroid` dipasang untuk hitung centroid marker.

**Marker bernomor berjenjang**: centroid tiap wilayah dihitung sekali dari GeoJSON (`@turf/centroid`). Zoom < 6 → satu pin per **provinsi** (angka = `provinces[].total`); zoom ≥ 6 → pin per **kabupaten** (angka = `cities[].total`). Hanya wilayah `total > 0` yang dapat pin. Klik pin/fill → filter provinsi + `flyTo`. Ukuran & warna pin dari magnitude. Marker = `new maplibregl.Marker({element})` dengan `<div>` lingkaran+angka.

**Batas verifikasi**: render visual MapLibre **tidak bisa diuji di Claude Browser pane** — pane `hidden`/tak compositing, `requestAnimationFrame` tak fire, jadi render loop MapLibre (yang memicu event `load`) hang. Bukan bug kode (SVG jalan karena tak butuh RAF). Verifikasi visual harus di browser asli (tab aktif). Yang terbukti dari sisi kami: `tsc` bersih, `next build` (Webpack) sukses, worker/WebGL/blob-worker semua fungsional di pane.

Angka terverifikasi (Juli 2026, super admin): total 245, provinces 15, unlocated 60, 185+60=245 (tidak bocor), 0 unmatched. Filter `province=Jawa Barat` → 110. `account_group=NPP2` → 156 (sama dengan Analitik). Klik provinsi di peta memfilter seluruh dashboard.

**KPI Pending/Cancel yang dulu selalu 0 kini benar** sebagai efek samping penyatuan status: Dashboard pending 0→195, cancelled 0→57 (cocok query manual). Angka Analitik tidak bergeser (tetap 245/15/0).

## Rancangan Backend

### `app/Support/ConsultationStatusGroups.php` (baru)

Satu sumber kebenaran untuk "status apa yang dihitung sebagai survey / deal / pending / batal".

Saat ini logika itu tersebar di dua tempat sebagai method privat — `AnalyticsReportService::surveyStatusAliases()/dealStatusAliases()/resolveStatusIds()` dan `DashboardService::statusAliases()/resolveStatusId()` — dan isinya **sudah berbeda**. Menambah tempat ketiga akan membuat angka antar halaman berpotensi tidak sama.

Menyediakan `surveyIds()`, `dealIds()`, `pendingIds()`, `cancelledIds()`. `AnalyticsReportService` dan `DashboardService` didelegasikan ke sini, wajib diverifikasi angkanya tidak bergeser.

### `config/statuses.php` — lengkapi dengan nama nyata

```php
'deal'      => 'Selesai/Deal',
'survey'    => ['Request Survey', 'Sedang Survey', 'Selesai Survey'],
'pending'   => ['Masih konsultasi', 'Nunggu Bangunan', 'Perbandingan Harga', 'Hanya Tanya Tanya'],
'cancelled' => ['Kendala Anggaran', 'Tidak Ada Respon'],
```

### `app/Support/RegionNameMatcher.php` (baru)

Mencocokkan nama wilayah DB ke fitur GeoJSON. Kunci pencocokan mengikuti pola `normalizeLocation()` yang sudah ada di `AnalyticsReportService` (lowercase, ascii, non-alfanumerik jadi spasi, squish).

**Wajib ada lapisan alias** karena penamaan kanonik aplikasi sengaja berbeda dari BPS/GeoJSON umum — lihat [[11-Modules/Wilayah]]: aplikasi memakai `Jakarta` (bukan `DKI Jakarta`), `Kab. Bandung` (bukan `Kabupaten Bandung`).

Daftar acuan diambil dari `config/wilayah.php` + `resources/data/wilayah_kota.php` yang **sudah ada**, bukan bikin daftar baru. Mapping kota→provinsi di `wilayah_kota.php` juga bisa dipakai mengelompokkan kota tanpa bergantung kolom `province` yang 23% placeholder.

Fungsi `unmatched()` mengembalikan nama yang gagal dicocokkan supaya ditampilkan, bukan hilang diam-diam.

### `app/Services/Geo/GeoAnalyticsService.php` (baru)

File terpisah, bukan menumpang di `AnalyticsReportService` yang sudah 2000+ baris.

Dipakai ulang, bukan ditulis ulang: `ReportPeriodResolver` (termasuk periode Kustom), `Consultation::forUser()`, pola filter `account`/`account_group`, `PendingConfirmation`.

Mengembalikan: `kpi`, `statusBreakdown` (dari `status_categories`, bukan hardcode), `provinces`, `cities`, `accountRanking`, `unmatchedRegions`, `productByRegion`. Agregasi pakai `GROUP BY` di SQL, bukan loop PHP.

### Request / Controller / Route

`GeoAnalyticsRequest` mewarisi bentuk filter `AnalyticsReportRequest` + `province`, `status`, `needs_category`, `level`. Controller tipis. Route `GET /api/v1/geo-analytics`.

## Rancangan Frontend

`public/geo/idn-provinces.geojson` — batas provinsi tersederhanakan (target < 500 KB), di-fetch sekali dan di-cache TanStack Query.

Halaman mengikuti pola halaman berat yang sudah dipakai repo ini: `page.tsx` tipis dengan `next/dynamic` + `ssr: false` + `PageSkeleton` (lihat `report-attendances/page.tsx`). Penting karena `maplibre-gl` besar dan tidak boleh masuk bundle awal.

```
src/app/geo-analytics/
  page.tsx                    dynamic import + skeleton
  geo-analytics-view.tsx      komposisi + state filter
  _components/
    geo-map.tsx               MapLibre choropleth (client-only)
    geo-kpi-cards.tsx
    geo-filters.tsx
    region-ranking.tsx
    region-detail-panel.tsx
src/lib/hooks/useGeoAnalytics.ts
```

Gaya visual mengikuti tema aplikasi yang sudah ada: `rounded-2xl`, **`bg-card` solid (bukan transparan)**, aksen `var(--primary-theme)`, panel filter berbingkai seperti halaman Absensi.

Peta: satu sumber GeoJSON, layer `fill` diwarnai interpolasi berdasarkan `total`, plus layer `line` untuk batas. Wilayah tanpa data diberi kelas warna terpisah supaya berbeda jelas dari wilayah bernilai kecil — menjawab "wilayah mana yang belum pernah dikunjungi".

Registrasi menu: `sidebar.tsx` (`NAV_LINKS`), `bottom-nav.tsx` (`MORE_ITEMS`), `roles.ts` (`ROUTE_ROLES` → `{ prefix: '/geo-analytics', roles: ['admin'] }`).

## Verifikasi yang direncanakan

1. Refactor status tidak menggeser angka — bandingkan sebelum/sesudah untuk Juli 2026 (patokan: 245 lead / 15 survey / 0 deal).
2. KPI cocok dengan SQL manual untuk beberapa kombinasi periode + grup akun.
3. Semua 16 nilai `province` terpetakan atau muncul di `unmatchedRegions`.
4. Angka tidak bocor: total = jumlah semua provinsi di peta + "belum berlokasi" (61).
5. Filter menyatu: NPP2 Juli 2026 harus sama dengan Analitik (patokan: 156).
6. `maplibre-gl` tidak masuk chunk awal.
7. Responsif 1440 / 903 / 390 px tanpa overflow horizontal.
8. Surveyor & manager_surveyor ditolak dari `/geo-analytics`.

## Di luar Fase 1

Marker cluster, heatmap, radius analysis, polygon analysis, territory management, dan AI Insight semuanya bergantung pada koordinat per konsumen yang belum ada.

Prasyarat: kolom `latitude`/`longitude` + cara mengisinya (map picker di form konsultasi). Geocoding data lama **tidak bisa** diandalkan karena `address` kosong 100% — presisi maksimal hanya titik pusat kota, yang membuat semua konsumen satu kota menumpuk di satu titik identik dan membuat radius/polygon tidak bermakna.
