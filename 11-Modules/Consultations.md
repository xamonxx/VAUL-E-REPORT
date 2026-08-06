# Consultations

Terkait: [[11-Modules/Survey]], [[11-Modules/Notifications]], [[11-Modules/Accounts]], [[06-API/API-Catalog]], [[05-Database/Data-Model]].

## Tujuan

Consultation adalah entity lead utama. Modul ini menangani pencatatan prospek, pipeline status, kebutuhan produk, lokasi, follow-up, reminder, import massal, dan export leads.

## Backend

- Controller: `ConsultationController`, `ConsultationNoteController`, `ReminderController`.
- Request: `ConsultationRequest`, `ConsultationNoteRequest`, `ReminderRequest`.
- Model: `Consultation`, `ConsultationNote`, `Reminder`, `ConsultationImport`, `ConsultationStatusHistory`, `NeedsCategory`, `StatusCategory`.
- Service: `ConsultationImportService`.
- Job: `ProcessConsultationImportJob`.
- Policy: `ConsultationPolicy`, `ConsultationNotePolicy`, `ReminderPolicy`.
- Support: `PendingConfirmation` (label placeholder), `PhoneNumber` (E.164), `WilayahNormalizer` (ejaan wilayah).
- Command: `consultations:backfill`.

## API

- `GET /consultations`
- `GET /consultations/id-preview`
- `GET /consultations/import/template`
- `POST /consultations/import`
- `POST /consultations`
- `GET /consultations/{consultation}`
- `PUT/PATCH /consultations/{consultation}`
- `DELETE /consultations/{consultation}`
- `PATCH /consultations/{consultation}/status`
- `POST /consultations/{consultation}/notes`
- `DELETE /consultations/{consultation}/notes/{note}`
- `POST /consultations/{consultation}/reminders`
- `DELETE /consultations/{consultation}/reminders/{reminder}`

## Frontend

- Route:
  - `/consultations`
  - `/consultations/create`
  - `/consultations/[id]`
  - `/consultations/[id]/edit`
- View: `consultations-view.tsx`.
- Components: `panduan-pengisian.tsx`, `needs-category-option.tsx`, `consultation-form-actions.tsx`.
- Hook: `useConsultations.ts`.
- Types: `Consultation`, `ConsultationFilters`.

## ID lead

Format: `AA.YYMM.NNNN`.

- `AA`: account id zero padded 2 digit.
- `YYMM`: tahun dan bulan.
- `NNNN`: urutan per account per bulan.
- Generator memakai transaksi dan `consultation_sequences`.

## Nilai default saat field dikosongkan

| Field | Nilai bila kosong |
|---|---|
| `province` / `city` / `district` | `Belum konfirmasi` |
| kategori kebutuhan | kategori bernama `Tidak konfirmasi` |

Keduanya diterapkan di `ConsultationRequest::prepareForValidation()` dan juga di `ConsultationImportService`, sehingga lead dari form maupun import konsisten.

`App\Support\PendingConfirmation` menyimpan label wilayah (`REGION_LABEL`) sekaligus label lama (`LEGACY_LABEL = 'Belum ada konfirmasi'`). `matches()` menerima keduanya agar data yang belum di-backfill tetap dikenali sebagai placeholder dan tidak divalidasi terhadap dataset wilayah.

Kategori kebutuhan **tidak wajib dipilih**. Form create/edit tidak lagi memblokir submit; backend yang mengisi `Tidak konfirmasi`.

## Nomor telepon

Disimpan **E.164** (`+6283134774955`) - satu bentuk baku tanpa spasi atau strip.

- Helper: `App\Support\PhoneNumber` (memakai `giggsey/libphonenumber-for-php`, mesin yang sama dengan `libphonenumber-js` di frontend).
- Nomor tanpa `+` dianggap nomor Indonesia; nomor asing wajib membawa kode negara.
- Panjang divalidasi per aturan negara, bukan rentang digit tetap - nomor lokal Singapura yang hanya 8 digit tetap sah.
- `normalizeLeadPhone()` memakai kunci E.164, sehingga `0831...` dan `+62 831...` terbaca sebagai nomor yang sama saat deteksi duplikat.
- Tampilan dan export memakai `PhoneNumber::format()`.

Catatan importer: kolom telepon disanitasi `sanitizePhoneCell()`, bukan `stripCsvInjection()` biasa. Sanitasi umum membuang `+` di awal, yang membuat `+60123456789` berubah jadi nomor Indonesia.

## Template import leads

**15 kolom** (sebelumnya 14; `Kecamatan` disisipkan di posisi J):

```
A NO · B ID KONSUMEN · C TGL. AWAL KONSUL · D NAMA AKUN · E Nama Konsumen
F WA Konsumen · G Domisili · H Provinsi · I Kota Kab. · J Kecamatan
K Jenis Kebutuhan · L Detail Kebutuhan · M Catatan · N kategori · O Tanggal Update
```

Kolom oranye (`NO`, `ID KONSUMEN`, `Domisili`) berisi rumus otomatis - jangan diisi manual.

Dropdown bertingkat memakai `OFFSET/MATCH/COUNTIF` atas kolom lookup di sheet `Opsi`, bukan ratusan named range:

- Kota: `OFFSET(Opsi!$I$2, MATCH(H5,CityProvinceCol,0)-1, 0, COUNTIF(CityProvinceCol,H5), 1)`
- Kecamatan: `OFFSET(Opsi!$Q$2, MATCH(I5,DistrictCityCol,0)-1, 0, COUNTIF(DistrictCityCol,I5), 1)`

Syaratnya baris lookup harus terurut per induk supaya tiap kelompok menempati blok berurutan.

**Kecamatan sengaja tidak mengikat**: validasinya memakai `showErrorMessage=0`, jadi dropdown tetap muncul tapi user boleh mengetik kecamatan yang tidak ada di daftar. Ejaannya dirapikan saat import (lihat [[11-Modules/Wilayah]]). Provinsi dan Kota tetap mengikat karena daftarnya lengkap.

Dukungan non-mengikat ini butuh elemen `<Strict>0</Strict>` di SpreadsheetML, yang diterjemahkan `SpreadsheetXmlToXlsxConverter` menjadi atribut `showErrorMessage`.

Importer menerima lebar **14 maupun 15 kolom**. `hasDistrictColumn()` menentukannya dari posisi kolom Jenis Kebutuhan (indeks 10 pada tata letak baru, 9 pada lama), dengan jumlah kolom sebagai penentu terakhir.

Import hanya menerima **CSV** (`mimes:csv,txt`), jadi template xlsx harus di-Save As CSV lebih dulu.

## Backfill data lama

```
php artisan consultations:backfill            # pratinjau, tidak menulis
php artisan consultations:backfill --apply    # tulis perubahan
```

Menyelaraskan: ejaan wilayah, label placeholder, `district` yang masih NULL, nomor telepon ke E.164, dan mengosongkan `address` yang terlanjur berisi `Dalam Kota`/`Luar Kota` (akibat salah pemetaan kolom Domisili di importer lama).

Aman diulang - jalan kedua menghasilkan nol perubahan. Nilai yang tidak dikenali dataset dipertahankan dan dilaporkan, tidak dibuang.

## Duplicate detection

Model `Consultation` punya normalisasi:

- `normalizeLeadText`
- `normalizeLeadPhone`
- `buildLeadPhoneKey`
- `buildLeadProfileKey`
- `findDuplicateLead`
- `findDuplicatePhone`

Duplicate bisa cocok lewat phone per account atau profile key.

## Cache/query

- Query list harus menjaga filter server-side dan pagination.
- Perubahan consultation menginvalidate dashboard, analytics, consultation list/detail, dan notification terkait.
- Catatan aktivitas dikirim realtime melalui event `ConsultationNoteCreated`. Daftar penerima tunggal dipakai bersama Reverb dan Web Push: super admin serta admin akun konsultasi yang sama, kecuali pengirim.
- Perubahan pesan dikirim lewat `ConsultationNotesChanged` agar edit, hapus pilihan, dan bersihkan percakapan langsung memperbarui detail pengguna lain.
- Pada percakapan, admin hanya dapat mengedit dan menghapus pesan miliknya. Super admin dapat menghapus pesan siapa pun dan membersihkan seluruh percakapan, tetapi tetap hanya dapat mengedit pesan miliknya sendiri.
- Akses detail tetap ditegakkan oleh `ConsultationPolicy`; frontend gating bukan pengaman data.

## Risiko perubahan

- Ubah field consultation berarti update request backend, migration, type frontend, form create/edit, detail view, export, dan import.
- Ubah status pipeline harus cek sinkronisasi dengan survey state.
- Menyisipkan kolom baru di template menggeser indeks di `ConsultationImportService::parseDetailedTemplateRow()`. Perbarui juga `hasDistrictColumn()` bila pola deteksinya ikut berubah.
- Kategori kebutuhan dan status pipeline diambil dari master data tanpa daftar putih. Kategori uji coba yang pernah dibuat di Master Data akan ikut tampil di form dan template - tinjau isi `needs_categories` sebelum rilis.
- Mengubah nama baris `needs_categories` lewat migrasi wajib membersihkan `MasterDataController::CACHE_NEEDS`; tanpa itu form dan template menyajikan nama lama sampai cache 6 jam kedaluwarsa.
