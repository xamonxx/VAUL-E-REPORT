# Design System

## Overview

Video ini mengikuti identitas UI login E-Report: permukaan kaca terang, aksen amber, teks slate, dan panel operasional yang padat tetapi mudah dipindai. Layout memakai frame dashboard/tutorial, bukan landing page. Komposisi dibuat landscape 1920x1080 untuk panduan internal.

## Colors

- **Primary Surface**: `#F8FAFC` - latar terang utama.
- **Panel Surface**: `#FFFFFF` - kartu dan mockup form.
- **Ink**: `#0F172A` - teks utama.
- **Muted Ink**: `#64748B` - teks pendukung.
- **Brand Amber**: `#F59E0B` - tombol, highlight, marker langkah.
- **Brand Amber Dark**: `#92400E` - teks label amber kecil agar kontras.
- **Deep Slate**: `#26354C` - panel gelap dan field dark mode.
- **Success**: `#10B981` - status terkirim dan tiket berhasil.
- **Success Dark**: `#047857` - teks/status sukses kecil agar kontras.
- **Danger**: `#EF4444` - bug/error marker.

## Typography

- **Display**: Georgia, 800-900. Judul besar dan angka langkah.
- **Interface**: Arial, 400-700. Label, field, body, dan UI mockup.
- **Mono**: Cascadia Mono, 600. Ticket code dan endpoint-like labels.

## Elevation

Depth dibuat lewat border halus, glass surface, shadow lembut, dan ambient glow amber. Tidak memakai card berlapis di dalam card besar; setiap frame punya panel utama dan elemen pendukung terpisah.

## Components

- **Login Glass Panel**: mockup form dengan email, password, tombol masuk, dan link lupa password.
- **Support Request Modal**: panel reset password berisi nama admin, nama akun, catatan, dan tombol kirim.
- **Bug Report Widget**: tombol mengambang "Laporkan Bug" dan modal ringkas untuk deskripsi serta screenshot.
- **Operational Rail**: daftar langkah di sisi kiri dengan status progres.
- **Ticket Badge**: kode tiket monospace sebagai bukti laporan terkirim.

## Do's and Don'ts

### Do's

- Gunakan `#F59E0B` untuk fokus tindakan utama.
- Pertahankan kontras tinggi pada teks kecil.
- Tampilkan langkah sebagai aksi konkret, bukan klaim marketing.
- Pakai mockup UI yang mirip aplikasi asli.

### Don'ts

- Jangan membuat hero promosi generik.
- Jangan dominasi warna ungu/biru gelap.
- Jangan memakai teks panjang yang sulit dibaca dalam 20 detik.
- Jangan menampilkan password asli atau data sensitif.
