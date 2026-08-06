# Production Server Spec

Tanggal pencatatan: 2026-08-03

Dokumen ini menyimpan baseline spek server yang direncanakan untuk menjalankan E-Report. Gunakan sebagai referensi deployment, sizing, troubleshooting performa, dan keputusan upgrade.

## Ringkasan kelayakan

Server cocok untuk production kecil sampai menengah/internal dengan beban E-Report normal: Laravel API, Next.js frontend, MySQL/MariaDB, Redis/cache/queue, Web Push PWA, scheduler, dan akses remote Tailscale.

Kapasitas utama yang kuat adalah RAM 15 GiB dan SSD NVMe. Batas utama ada di CPU mobile Intel Core i3-7100U 2 core / 4 thread, sehingga workload berat seperti export besar, build Next.js, import besar, atau banyak container aktif perlu dijadwalkan dan dipantau.

Verdict awal: layak untuk production awal/internal, dengan monitoring dan limit worker yang konservatif.

## Sistem operasi

| Item | Nilai |
| --- | --- |
| Distribusi | Ubuntu 24.04.4 LTS (Noble Numbat) |
| Kernel | Linux 6.8.0-136-generic (x86_64) |
| Arsitektur | 64-bit |

## CPU

| Item | Nilai |
| --- | --- |
| Model | Intel Core i3-7100U @ 2.40 GHz |
| Core / thread | 2 core / 4 thread |
| Socket | 1 |
| Base clock | 2.40 GHz; dapat turun sekitar 400 MHz saat idle |
| Cache | L1 128 KB, L2 512 KB, L3 3 MB |
| Generasi | Kaby Lake generasi ke-7 |
| Fitur | AES-NI, AVX2, FMA, SSE4.2, VT-x |

Catatan: i3-7100U adalah chip mobile 15W dari 2017. Cukup untuk layanan web kecil, tetapi bukan target ideal untuk workload multi-container berat atau concurrency tinggi.

## Virtualisasi dan keamanan CPU

| Item | Status |
| --- | --- |
| NUMA | 1 node |
| Mitigasi eksploitasi arsitektur | Meltdown, Spectre, L1TF, Retbleed, dan MDS sudah dimitigasi lewat microcode + kernel |
| SMT / hyperthreading | Terdeteksi masih memiliki catatan vulnerable; risiko dinilai tidak kritis untuk penggunaan internal |

## Memori dan storage

| Komponen | Kapasitas | Terpakai saat dicatat |
| --- | ---: | ---: |
| RAM | 15 GiB | 3.5 GiB; tersedia sekitar 11 GiB |
| SSD NVMe ADATA SX8200PNP | 238.5 GB | 48 GB atau 22% |
| HDD HGST 5400 RPM | 465.8 GB | 2.5 GB |
| Total storage | sekitar 704 GB | sekitar 50 GB atau 8% |

Rekomendasi:

- Tempatkan sistem, database aktif, app, Redis, dan file cache di SSD NVMe.
- Gunakan HDD untuk backup, arsip export lama, dan snapshot non-runtime.
- Pantau ruang kosong database, file upload, log Laravel, log Nginx, dan backup.

## Jaringan

| Interface | Jenis | Status |
| --- | --- | --- |
| enp2s0 | Ethernet Realtek RTL8111/8168 | UP |
| wlp3s0 | WiFi Realtek RTL8723BE | DOWN |
| tailscale0 | VPN Tailscale | UP |

Rekomendasi:

- Gunakan Ethernet sebagai jalur utama production.
- Gunakan Tailscale untuk akses admin/SSH/maintenance, bukan untuk public traffic utama.
- Public ingress tetap perlu TLS, reverse proxy, firewall, dan rate limiting.

## Layanan yang cocok

Server ini layak menjalankan stack berikut dalam satu host:

- Nginx atau Caddy sebagai reverse proxy TLS.
- PHP-FPM untuk Laravel.
- Next.js server mode untuk frontend.
- MySQL atau MariaDB.
- Redis untuk cache, session, dan queue.
- Laravel queue worker.
- Laravel scheduler.
- Web Push PWA.
- Tailscale untuk akses remote aman.

## Batas aman operasional

Konfigurasi konservatif yang disarankan:

- Jalankan 1 proses Next.js production utama.
- Jalankan PHP-FPM dengan jumlah worker moderat; hindari nilai tinggi yang menghabiskan RAM/CPU.
- Jalankan 1 sampai 2 queue worker terlebih dahulu.
- Jadwalkan export besar, import besar, backup, dan build Next.js di luar jam sibuk.
- Hindari menjalankan banyak container berat yang tidak berhubungan dengan E-Report di host yang sama.

Indikator perlu upgrade:

- CPU sering di atas 70-80% saat jam operasional normal.
- Load average konsisten di atas jumlah core efektif.
- Response API P95 mulai melewati 800-1000 ms untuk endpoint umum.
- Queue attendance/notifikasi/import sering menumpuk.
- MySQL mulai sering masuk slow query untuk dashboard/report.
- Build/deploy mengganggu user aktif.

## Environment production yang direkomendasikan

Backend Laravel:

```env
APP_ENV=production
APP_DEBUG=false
QUEUE_CONNECTION=redis
CACHE_STORE=redis
SESSION_DRIVER=redis
```

Frontend Next.js mode proxy/auth gate:

```env
NEXT_PUBLIC_SAME_ORIGIN_API=true
NEXT_PUBLIC_AUTH_GATE=true
NEXT_PUBLIC_API_URL=
API_PROXY_TARGET=http://127.0.0.1:8000
```

Jika backend Laravel berada di host/domain berbeda, `API_PROXY_TARGET` diisi URL backend production yang sebenarnya, misalnya `https://api.example.com`. Nilai ini server-only dan tidak perlu memakai prefix `NEXT_PUBLIC_`.

## Catatan performa E-Report

Prioritas optimasi untuk server ini:

1. Pastikan query dashboard/report memakai index dan pagination.
2. Pakai Redis untuk session/cache/queue agar request web tidak tersangkut job berat.
3. Isolasi kegagalan Web Push agar absensi tetap berhasil walau push provider gagal.
4. Hindari caching PWA untuk response API privat.
5. Gunakan proxy same-origin + auth gate hanya setelah domain cookie Sanctum lolos uji staging.

## Status verifikasi

Belum ada load test penuh pada server fisik ini. Perlu uji staging dengan data sintetis sebelum production:

- Login dan navigasi role utama.
- Absensi dan notifikasi superadmin.
- Export laporan.
- Queue worker saat push gagal.
- PWA Android install dan refresh offline/online.
- Monitoring CPU, RAM, disk I/O, dan slow query selama uji.
