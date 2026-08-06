# Storyboard

**Format:** 1920x1080
**Audio:** Visual-first tutorial. Voiceover script disiapkan di `SCRIPT.md`; versi render saat ini fokus pada teks layar dan motion.
**VO direction:** Tenang, cepat, jelas. Register tutorial internal.
**Style basis:** `DESIGN.md`

## Asset Audit

| Asset | Type | Assign to Beat | Role |
| --- | --- | --- | --- |
| Mockup UI login | HTML/CSS | Beat 1, 2 | Menunjukkan halaman login E-Report |
| Mockup modal reset | HTML/CSS | Beat 3 | Menunjukkan alur lupa password |
| Mockup bug report | HTML/CSS | Beat 4 | Menunjukkan alur lapor bug |
| Ticket badge | HTML/CSS | Beat 4 | Bukti laporan terkirim |

## BEAT 1 - Pembuka dan konteks (0:00-0:03.5)

**Concept:** Viewer langsung melihat E-Report sebagai alat kerja. Panel login masuk dengan aksen amber, sementara rail kiri menyiapkan tiga topik: login, reset password, dan lapor bug.

**Visual:** Logo teks E-Report besar, subjudul "Panduan 20 detik", dan tiga step pill. Background terang dengan glow amber dan grid operasional tipis.

**Transition:** Push slide ke beat login.

## BEAT 2 - Cara login (0:03.5-0:08.5)

**Concept:** Fokus pada tindakan paling sederhana: isi email, isi password, klik tombol masuk.

**Visual:** Mockup form login membesar. Cursor/highlight bergerak dari field email ke password lalu tombol "Masuk ke E-Report". Sisi kiri menampilkan checklist tiga langkah.

**Transition:** Blur through menuju reset password.

## BEAT 3 - Cara reset password (0:08.5-0:14)

**Concept:** Reset password untuk user umum dimulai dari link "Lupa password?" pada login. Permintaan dikirim ke WhatsApp dukungan untuk diproses admin/tim.

**Visual:** Link lupa password diberi marker, modal bantuan akses akun muncul, tiga field terisi simulasi, tombol "Kirim permintaan" aktif, lalu kartu WhatsApp dukungan muncul.

**Transition:** Staggered amber wipe menuju lapor bug.

## BEAT 4 - Cara laporkan bug (0:14-0:20)

**Concept:** Tunjukkan bahwa laporan bug bisa dibuat cepat dari tombol mengambang, bahkan dari halaman login.

**Visual:** Tombol "Laporkan Bug" muncul di kiri bawah, modal bug report terbuka, deskripsi dan screenshot ditambahkan, tombol kirim ditekan, lalu muncul tiket `BUG-240730-001`.

**Transition:** Final fade lembut ke title akhir.

## Production Architecture

```
e-report-login-reset-bug-20s/
|-- index.html
|-- DESIGN.md
|-- SCRIPT.md
|-- STORYBOARD.md
|-- snapshots/
`-- renders/
```
