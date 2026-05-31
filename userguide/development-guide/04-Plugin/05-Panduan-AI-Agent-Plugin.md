---
title: Panduan AI Agent (Plugin)
description: Panduan spesifik untuk AI agent saat membuat plugin SLiMS Bulian secara hemat token dan mengutamakan fitur bawaan.
keywords: [slims plugin, ai agent, simbio, rag]
---

# Panduan AI Agent untuk Pembuatan Plugin SLiMS

Dokumen ini khusus untuk AI agent agar membuat plugin berbasis **fitur bawaan SLiMS** (terutama `SLiMS\Plugins` dan pustaka `simbio`) dari codebase `slims/slims9_bulian`.

## Aturan Utama (Wajib)

1. **Utamakan reuse fitur inti SLiMS**, jangan reimplementasi.
2. Prioritas API:
   - `SLiMS\Plugins` (`register`, `registerMenu`, `menu`, `hook`, `registerPages`)
   - helper bawaan SLiMS (jika sudah tersedia)
   - pustaka `simbio` (`simbio_DB`, `simbio_GUI`, `simbio_UTILS`, dll)
3. Hindari menambah dependency baru jika fitur inti sudah cukup.
4. Perubahan harus kecil, fokus, dan kompatibel dengan struktur plugin SLiMS.

## Referensi Sumber Kode (Bulian)

- `lib/Plugins.php` (registrasi hook/menu/pages dan konstanta hook)
- `plugins/label_barcode/index.php` (contoh penggunaan `simbio_GUI` dan `simbio_DB`)
- `plugins/read_counter/index.php` (contoh plugin admin + simbio)
- `plugins/csp-manager/index.php` (contoh require modul simbio)

## CAVEMAN Method (Token-Efficient RAG)

Gunakan format jawaban pendek, satuan informasi kecil, tanpa narasi panjang:

- **C**ontext: modul/hook yang disentuh
- **A**im: tujuan plugin 1 kalimat
- **V**alid API: API inti yang dipakai (`Plugins::*`, simbio)
- **E**xisting reuse: komponen bawaan yang dipakai ulang
- **M**inimal patch: file yang diubah + alasan singkat
- **A**voided complexity: hal yang sengaja tidak ditambah
- **N**ext validation: verifikasi cepat (syntax/build/manual path)

## Template Instruksi untuk AI Agent (RAG Chunk)

```text
KONTEKS:
- Repo: slims/slims9_bulian
- Area: plugins/

TUJUAN:
- Buat/ubah plugin <nama_plugin> untuk <fungsi>.

BATASAN WAJIB:
- Gunakan SLiMS\Plugins lebih dulu.
- Utamakan simbio_* untuk UI/DB/utilitas.
- Jangan tambah library eksternal jika bawaan cukup.
- Patch sekecil mungkin.

OUTPUT:
1) Daftar file berubah (bullet list: path + ringkasan 1 baris)
2) Kode final (hanya potongan relevan)
3) Cara uji singkat (langkah bernomor)
4) Risiko/kompatibilitas (maks 3 poin)
```

## Skeleton Minimal Plugin (Disarankan)

```php
<?php
/**
 * Plugin Name: Contoh Plugin AI
 * Version: 1.0.0
 * Author: Tim
 */
use SLiMS\Plugins;

$plugins = Plugins::getInstance();

// Hook contoh
$plugins->register(Plugins::MEMBERSHIP_INIT, function () {
    // placeholder: validasi field wajib member (mis. email tidak kosong & format valid)
});

// Menu admin contoh
Plugins::menu('membership', 'Contoh AI', __DIR__ . '/index.php');
```

## Saat Perlu UI/DB: Utamakan simbio

Contoh pola yang diprioritaskan:

```php
require SIMBIO . 'simbio_GUI/table/simbio_table.inc.php';
require SIMBIO . 'simbio_GUI/form_maker/simbio_form_table_AJAX.inc.php';
require SIMBIO . 'simbio_DB/datagrid/simbio_dbgrid.inc.php';
```

> Prinsip: jika kebutuhan bisa dipenuhi oleh `SLiMS\Plugins` + `simbio`, gunakan itu dulu sebelum membuat abstraksi baru.
