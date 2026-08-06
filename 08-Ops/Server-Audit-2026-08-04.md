# Server Audit & Hardening — 2026-08-04

Tanggal pencatatan: 2026-08-04
Auditor: REMOX (mode read-only dulu, tindakan per-persetujuan)
Server: `serverone` (mini PC home lab, sama dengan [[Production-Server-Spec]])
Akses: SSH via Tailscale `mki@100.76.23.12:2201`

Audit kesehatan, keamanan, dan pembersihan menyeluruh. Fase awal read-only; setiap perubahan diminta persetujuan eksplisit satu per satu (format `ACT-xxx`). Konfigurasi **Nginx** dan **Cloudflare/cloudflared** diperlakukan read-only dan **tidak disentuh** sepanjang sesi.

---

## A. Ringkasan Eksekutif

- **Kesehatan: BAIK.** Disk lega, RAM cukup, swap 0B, 0 service failed, 0 OOM, 0 zombie.
- **Keamanan: SEDANG → membaik.** UFW + fail2ban + root-login off = fondasi bagus. Ditemukan & diperbaiki: bug fail2ban (ban salah port), AI-agent liar (hermes), backup satu-titik-gagal.
- **Risiko terbesar tersisa:** NVMe menunjukkan pola listrik tidak stabil (342 unsafe-shutdown) — akar masalah = daya, bukan drive sekarat.
- **Exposure:** hanya LAN + Tailscale (dikonfirmasi pemilik, tidak ada port-forward publik).
- **Ruang dibebaskan sesi ini:** ~7,7 GB. **Update:** dari 8 upgradable (3 security) → **0 tersisa**.

---

## B. Inventaris Server

| Item | Nilai |
| --- | --- |
| OS / Kernel | Ubuntu 24.04.4 LTS / 6.8.0-136-generic (bare-metal, virt=none) |
| CPU | Intel Core i3-7100U — 2 core / 4 thread @ 2.40GHz |
| RAM / Swap | 15 GiB / 4 GiB (swap 0B terpakai) |
| Disk sistem | NVMe ADATA SX8200PNP 238G — `/` ext4 (19–22%), `/boot/efi` |
| Disk data | HGST 465G — `/data` ext4 (1%, ~432G kosong) |
| Network | enp2s0 `192.168.1.100`, tailscale0 `100.76.23.12`, 4 docker bridge |
| Runtime | Docker 29.7.1 (4→3 container), Cloudflare tunnel (host), Tailscale |
| Layanan host (localhost) | MariaDB `3306`, Redis `6379`, PHP `8080`, Postfix `25` |
| Panel | aaPanel (`37964`/`888`) + CasaOS (`81`) — keduanya dipakai |
| Keamanan | UFW aktif, fail2ban aktif, AIDE (dailyaidecheck), root-login off |

---

## C. Tindakan Dieksekusi (semua atas persetujuan)

| ID | Tindakan | Hasil | Rollback |
| --- | --- | --- | --- |
| ACT-001 | Stop container `hermes` (AI-agent eksperimen, expose 0.0.0.0:8642/9119) | ✅ Exited(0) | `docker start hermes` (saat itu) |
| BATCH-A | Vacuum journald 2.2G→481M; pasang 3 security update (tzdata, tzdata-legacy, distro-info-data); apt clean | ✅ −1.7G | — |
| ACT-006 | Hapus permanen hermes (container + image 4.59GB + network + def CasaOS + data `/DATA/AppData/hermes`) | ✅ −6G | irreversible |
| ACT-007 | Salin `/www/backup` → disk sda `/data` (redundansi backup) | ✅ 41 file | — |
| ACT-007b | Pasang cron auto-sync harian 02:00 (snapshot bertanggal + retensi 30h) | ✅ aktif | hapus cron + `/usr/local/bin/backup-sync-sda.sh` |
| ACT-008 | Hapus sisa host-side hermes `/home/mki/.hermes` (2.2G, berisi token) | ✅ bersih total | irreversible |
| ACT-005 | Upgrade Docker 29.6.2→29.7.1 + Node 22.23.1→22.23.2 (maintenance window) | ✅ 0 upgradable | downgrade paket |
| ACT-009 | Fix fail2ban: jail sshd `port = 2201` (sebelumnya ban salah port 22) | ✅ terbukti via uji ban-dummy | `jail.local.bak-2026-08-04` |
| ACT-013 | Set `PASS_MIN_DAYS = 1` (login.defs + `mki`) atas item aaPanel-scan | ✅ ternyata sudah patuh (no-op) | `login.defs.bak-2026-08-04` |

---

## D. Temuan Kesehatan

| Komponen | Status | Bukti |
| --- | --- | --- |
| Disk `/` | BAIK | 19% terpakai sesudah cleanup, inode 7% |
| RAM/Swap | BAIK | 5.3/15Gi, swap 0B, 0 OOM |
| CPU/Load | INFO | load ~1.14 = proses `aide` (dailyaidecheck) 100% 1-core, wajar |
| NVMe SMART | **PERHATIAN** | PASSED, tapi Media/Data Integrity Errors **3.428** + Unsafe Shutdown **342** (lihat bagian F) |
| sda SMART | BAIK | HGST PASSED |
| Proses | BAIK | 0 zombie, 0 service failed |

---

## E. Temuan Keamanan

| Area | Temuan | Risiko | Status |
| --- | --- | --- | --- |
| fail2ban | Jail sshd ban `port = ssh` (22) padahal SSH di **2201** → ban tak efektif | TINGGI | **Diperbaiki (ACT-009)** |
| Container hermes | AI-agent (OpenAI key + Playwright) expose 0.0.0.0, terus probe host tailscale:8443 | TINGGI | **Dihapus (ACT-001/006/008)** |
| Backup | DB dump aaPanel hanya di NVMe yang sama (satu titik gagal) | TINGGI | **Diperbaiki (ACT-007/007b)** |
| Update | 3 security update tertunda | SEDANG | **Dipasang (BATCH-A)** |
| SSH | root-login off, pubkey on, **password on**, port 2201 | SEDANG | Terbuka — hardening opsional |
| UFW | aktif; rule `192.168.1.0/24 → semua port` + 887/2201 Anywhere | SEDANG | Terbuka — LAN-only turunkan bobot |
| Panel ganda | aaPanel + CasaOS jalan bersama | SEDANG (INFO) | Dipertahankan (kehendak pemilik) |
| UID0/sudo | hanya `root` & `mki` | BAIK | — |

---

## F. Analisis NVMe (ADATA SX8200PNP 256GB, fw R0906I)

### Data SMART
| Metrik | Nilai | Arti |
| --- | --- | --- |
| Critical Warning | 0x00 | tak ada flag bahaya |
| Available Spare | 100% (batas 10%) | nol blok cadangan terpakai |
| Percentage Used | 19% | keausan tulis ~19% → ~81% sisa |
| Data Units Written | 25.5 TB | total tulis seumur hidup |
| Power On Hours | 12.750 jam (~1,5 th) | umur nyala |
| Power Cycles | 5.137 | |
| Unsafe Shutdowns | 342 (~6,7%) | mati mendadak/listrik putus |
| Media/Data Integrity Errors | 3.428 | angka mencolok |
| Error Information Log Entries | 0 | kunci penafsiran |

### Penafsiran (terukur)
Angka 3.428 **kontradiktif** dengan indikator koroboratif: jika error uncorrectable nyata, Available Spare harusnya turun (ini 100%), Error-Log Entries > 0 (ini 0), Critical Warning menyala (ini 0x00). Ketiganya bersih. Pola "Media Errors tinggi + Error-Log 0 + Spare 100% + Critical 0" adalah **quirk firmware terdokumentasi** pada keluarga ADATA SX8200 (controller SMI): counter melonjak keliru, sering seiring event daya — **bukan** kerusakan media fisik. Namun tidak diabaikan karena berkorelasi dengan 342 unsafe-shutdown, dan mati mendadak **bisa** merusak data in-flight nyata (mis. MariaDB).

### Umur / sisa hidup
- Endurance rated SX8200 Pro 256GB ≈ **160 TBW**; terpakai 25.5 TB (~16%), sinkron dengan "Percentage Used 19%".
- Laju tulis ~48 GB/hari → proyeksi **~6–7 tahun lagi** sebelum batas tulis.
- **Keausan BUKAN faktor pembatas.** Akar risiko = **ketidakstabilan daya**, bukan drive sekarat.

### Dampak & risiko
| Risiko | Level | Sebab |
| --- | --- | --- |
| Korupsi data saat listrik putus | TINGGI | 342 unsafe-shutdown; MariaDB (`sql_api_ereport_interiorcustom_id`, `gudangmki`) & tulis container rentan |
| Gagal drive karena aus | RENDAH | Spare 100%, Used 19%, ~6th sisa |
| Bit-rot/uncorrectable menumpuk | RENDAH | Error-Log 0, Spare 100% |
| Counter 3.428 | INFORMASI | indikasi kuat artefak firmware |

---

## G. Skema Backup (baru, hasil ACT-007/007b)

Backup DB aaPanel (`/www/backup`, NVMe) kini **disalin otomatis ke disk sda sehat** — data + backup tidak lagi di satu drive.

```
/data/db-backups/www-backup/
├── 2026-08-04_0359/        <- snapshot (format tanggal_jam, mudah dibaca)
│   └── database/mysql/crontab_backup/<db>/db_<db>_<tgl>_<jam>_mysql_data.sql.gz
└── latest -> 2026-08-04_0359/   <- selalu tunjuk snapshot terbaru
```

- **Skrip:** `/usr/local/bin/backup-sync-sda.sh` (rsync + hardlink dedup + retensi 30 hari).
- **Cron root:** `0 2 * * *` (jalan sesudah dump aaPanel 01:30). 2 entri cron aaPanel existing tetap utuh.
- **Log:** `/var/log/backup-sync-sda.log`.

---

## G2. Review aaPanel Security Scan (skor 85)

Scan aaPanel melaporkan 9 risiko. Setelah verifikasi read-only, **praktis tak ada yang nyata** untuk server ini:

| # | Item | Vonis | Alasan |
| --- | --- | --- | --- |
| 1 | User FTP access security | Abaikan | FTP (pure-ftpd) tidak jalan |
| 2 | Min interval ganti password SSH | **Sudah patuh** | `PASS_MIN_DAYS=1` di login.defs, `mki`, `root` (ACT-013 konfirmasi, no-op) |
| 3 | SSL transmit info | Abaikan | terkait FTP (mati); jangan sentuh panel/nginx |
| 4 | Panel login alarm | Abaikan | upsell (support@aapanel.com) |
| 5 | SSH idle timeout | **False-positive** | `ClientAliveInterval 300` sudah efektif (config di drop-in, bukan file utama) |
| 6 | SUID/SGID check | Jangan one-click | 582 file normal; auto-strip bisa rusak `sudo`/`ping`; audit manual bila perlu |
| 7 | SSH login whitelist | Abaikan | upsell; `AllowUsers` manual berisiko lockout |
| 8 | Semua website SSL | Abaikan | upsell + **area Nginx/Cloudflare dilindungi**; SSL sudah di Cloudflare tunnel |
| 9 | Website anti cross-site | Abaikan di aaPanel | **area Nginx**; app E-REPORT set CSP sendiri → hindari bentrok header |

**Peringatan:** JANGAN pakai "Perbaikan satu klik" aaPanel — membundel perubahan sshd + website/nginx + hapus suid, berisiko putus akses dan menyentuh area dilindungi (Nginx/Cloudflare). Untuk 3 item upsell (4/7/8) cukup tekan "Mengabaikan"; sisanya false-positive/N/A. Skor 85 aman untuk homelab.

---

## H. Area Dilindungi (dinyatakan eksplisit)

- Konfigurasi **Nginx** tidak diubah (hanya `nginx -t` / status read-only).
- Konfigurasi **Cloudflare/cloudflared** tidak diubah.
- **MariaDB, Redis, aaPanel, CasaOS** tidak dihentikan/diubah (di luar hermes).
- 2 cron **aaPanel** tetap utuh.
- **Credential tidak ditampilkan** di laporan (env hermes, serial NVMe disensor).
- Tidak ada service dihentikan / paket dihapus / data dihapus **tanpa persetujuan eksplisit**.

---

## I. Rekomendasi Terbuka (belum jadi tindakan)

1. **UPS (prioritas tertinggi, ROI terbaik).** Akar 342 unsafe-shutdown = daya. UPS line-interactive 600–900VA + **NUT** untuk auto graceful-shutdown. Langsung setop sumber korupsi data & kemungkinan lonjakan counter NVMe.
2. **Monitoring SMART.** Pasang `smartd` — alert bila **Available Spare < 100%**, **Critical Warning ≠ 0**, atau **Percentage Used** naik cepat. Pantau **delta**, bukan angka Media-Errors absolut. (Kandidat ACT-010.)
3. **Backup off-site.** `rclone` sudah terpasang tapi 0 remote — tambah target cloud/NAS untuk pertahanan bila NVMe total hilang. (Kandidat ACT-012, butuh kredensial dari pemilik.)
4. **Hardening SSH** pubkey-only — **pastikan key `mki` terpasang & teruji dulu** sebelum matikan password-auth (jangan putus akses).
5. **Ganti NVMe: belum perlu.** Trigger hanya bila Available Spare turun, Critical Warning menyala, atau Percentage Used mendekati 80%.
6. **Catatan fail2ban:** jika jail SSH diedit lewat UI aaPanel Fail2ban Manager, `port = 2201` bisa tertimpa — set ulang bila itu terjadi.

---

Terkait: [[Production-Server-Spec]] · [[Runbook]] · [[Deployment-Notes]] · [[deploy-storage-link]]
