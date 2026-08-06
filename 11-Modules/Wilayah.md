# Wilayah

Terkait: [[11-Modules/Consultations]], [[05-Database/Data-Model]], [[15-Backend-Internals/Services]].

## Tujuan

Menyediakan data geografis Indonesia untuk form lead dan template import: provinsi, kota/kabupaten, kecamatan.

## Format kanonik

Penamaan mengikuti master Excel tim (`db_wilayah.xlsx`), bukan gaya BPS/emsifa:

| | Format dipakai | Bukan |
|---|---|---|
| Provinsi | `Jakarta`, `Yogyakarta` | `DKI Jakarta`, `Daerah Istimewa Yogyakarta` |
| Kabupaten | `Kab. Bandung` | `Kabupaten Bandung` |
| Kota | `Kota Bandung` | - |

Jumlah: **38 provinsi, 514 kota/kabupaten, 7283 pasangan kota-kecamatan**.

## Sumber data

Dataset dibangkitkan dari master Excel, **bukan** diunduh dari internet.

```
python tools/generate_wilayah.py --file "path/ke/db_wilayah.xlsx"
```

Generator membaca sheet `dv_wilayah (2)` (kolom AA/AB untuk kota-provinsi, J/K untuk kota-kecamatan), merapikan spasi ganda, lalu menulis:

- `config/wilayah.php` - daftar provinsi
- `resources/data/wilayah_kota.php` - `['mapping' => ['Kab. Aceh Barat' => 'Aceh', ...]]`
- `resources/data/wilayah_kecamatan.php` - `['mapping' => [['district','city','province'], ...]]`

`db_wilayah.xlsx` sengaja **tidak** di-commit (9 MB). Jalankan generator ulang setiap master diperbarui.

`app:sync-wilayah` (sumber emsifa) sudah **usang** dan menolak jalan tanpa `--force`. Sumber itu memakai penamaan berbeda dan hanya punya 34 provinsi, sehingga akan memutus kecocokan dengan template import.

## Backend

- Controller: `WilayahController`.
- Akses data: `App\Support\Wilayah` (lazy + memoized, cache-bust lewat mtime file).
- **Normalisasi: `App\Support\WilayahNormalizer`** - satu sumber kebenaran, dipakai bersama form, importer, dan exporter template.
- Generator: `tools/generate_wilayah.py`.
- Backfill data lama: `php artisan consultations:backfill` (lihat [[11-Modules/Consultations]]).

### WilayahNormalizer

- `canonicalProvince()` / `canonicalCity()` - samakan ejaan ke format kanonik; menerima alias lama (`DKI Jakarta`, `Kabupaten X`, `Kab X`) dan singkatan tim (`KBB`, `Jaksel`, `Tangsel`).
- `canonicalDistrict($nama, $kota)` - mengembalikan `['value' => ..., 'matched' => bool]`. Kecamatan boleh diketik manual, jadi nilai selalu ada; `matched=false` berarti tidak ada padanan di dataset.
- `cityKey()` **menyeragamkan** awalan, tidak membuangnya (`kab bandung` vs `kota bandung`). Membuang awalan membuat Kabupaten dan Kota bertabrakan di kunci yang sama.
- `cityPlainKey()` - cadangan untuk nama polos (`Bandung`); bila ada Kabupaten dan Kota bernama sama, varian `Kota` yang dimenangkan.

### Aturan perapian ejaan kecamatan

Dipakai saat import agar ketikan manual disesuaikan ke ejaan dataset. Dua kunci pembanding:

- `looseKey` - huruf kecil, tanpa titik/apostrof, tanda hubung jadi spasi, spasi dirapatkan
- `squashKey` - `looseKey` tanpa spasi sama sekali

Urutan pencocokan (berhenti di yang pertama cocok): persis -> `looseKey` dalam kota terpilih -> `squashKey` dalam kota terpilih -> `looseKey`/`squashKey` global.

Contoh hasil: `kaway xvi` -> `Kaway XVI`, `Blang Pidie` -> `Blangpidie`, `tangan-tangan` -> `Tangan-Tangan`, `SUKA MAKMUR` -> `Sukamakmur`.

Tidak cocok = nilai ketikan dipertahankan (setelah dirapikan spasinya) dan dicatat di log, bukan dibuang.

## API

- `GET /api/v1/wilayah/provinces`
- `GET /api/v1/wilayah/cities?province=...`
- `GET /api/v1/wilayah/districts?city=...`

`WilayahController::matchCity()` memakai kunci ternormalisasi. Sebelumnya memakai `str_contains` sebagai cadangan, sehingga kota `Bandung` ikut menarik kecamatan milik `Bandung Barat`.

## Frontend

- Dipakai di form consultation create/edit (dropdown bertingkat).
- Lokasi tersimpan di consultation: `province`, `city`, `district`, `address`.
- Wilayah yang tidak diisi tersimpan sebagai `Belum konfirmasi` (lihat [[11-Modules/Consultations]]).

## Risiko

- Mengubah format nama memengaruhi duplicate detection (`Consultation::buildLeadProfileKey` memakai `province`) serta filter dan laporan lokasi. Jalankan `consultations:backfill` bersamaan agar data lama ikut selaras.
- Menjalankan `app:sync-wilayah --force` akan menimpa dataset dengan penamaan emsifa dan merusak kecocokan template.
