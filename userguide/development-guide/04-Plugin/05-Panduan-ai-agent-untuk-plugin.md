---
title: Panduan AI Agent untuk Pembuatan Plugin
description: Panduan ringkas agar AI agent menghasilkan plugin SLiMS yang sesuai pola inti dan memanfaatkan pustaka bawaan.
keywords: [ai agent, plugin slims, simbio, lib, pengembangan plugin]
---

# Panduan AI Agent untuk Pembuatan Plugin

Halaman ini ditujukan sebagai acuan khusus untuk AI agent agar proses pembuatan plugin SLiMS lebih konsisten, aman, dan tidak mengulang pustaka yang sebenarnya sudah tersedia di inti SLiMS.

## Tujuan

Saat AI agent diminta membuat plugin, hasil yang diharapkan adalah:

1. Tetap mengikuti format plugin SLiMS (`*.plugin.php`).
2. Memakai hook resmi dari sistem plugin.
3. Mengutamakan fasilitas bawaan SLiMS, terutama yang ada di folder `simbio2` dan `lib` pada root instalasi SLiMS.
4. Menghindari penambahan dependency baru jika kebutuhan sudah bisa ditangani pustaka inti.

## Checklist singkat untuk AI agent

Sebelum menulis kode plugin, AI agent perlu memastikan:

- Lokasi plugin berada di `plugins/` dengan pola nama file yang jelas.
- Header plugin terisi (`Plugin Name`, `Description`, `Version`, `Author`).
- Inisialisasi plugin memakai `SLiMS\Plugins`.
- Hook yang dipilih sesuai konteks (OPAC, area admin, atau proses lain).
- Tidak menyalin ulang fungsi utilitas yang sebenarnya sudah tersedia di `simbio2` atau `lib`.

## Prioritas penggunaan pustaka bawaan

Jika ada kebutuhan umum berikut, AI agent perlu **memprioritaskan** pustaka internal:

- Akses database, helper utilitas, atau komponen pendukung: telusuri lebih dulu subfolder `simbio2` seperti `simbio_DB`, `simbio_UTILS`, `simbio_FILE`, dan `simbio_GUI`.
- Kebutuhan library umum aplikasi: cek folder `lib`, lalu gunakan library yang sudah dipaketkan sebelum menambah dependency baru.

Prinsipnya: **gunakan yang sudah ada terlebih dahulu, baru pertimbangkan library tambahan jika benar-benar tidak tersedia**.

## Contoh arahan prompt untuk AI agent

Gunakan pola arahan yang eksplisit, misalnya:

> Buat plugin SLiMS untuk menambahkan widget statistik di dashboard admin.  
> Wajib gunakan struktur plugin SLiMS, prioritaskan utilitas dari `simbio2` dan `lib`, serta jangan menambahkan dependency eksternal jika sudah ada fungsi bawaan.

## Validasi hasil plugin

Sebelum plugin dinyatakan selesai, AI agent perlu memeriksa:

1. Plugin dapat terdeteksi dan diaktifkan dari menu **Sistem → Plugin**.
2. Fitur plugin berjalan sesuai hook yang dipakai.
3. Kode tetap kompatibel dengan arsitektur inti SLiMS (tanpa modifikasi file core).
4. Tidak ada duplikasi fungsi yang sebenarnya sudah disediakan pustaka bawaan.
