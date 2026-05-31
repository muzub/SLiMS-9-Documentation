---
title: Panduan AI Agent (Plugin)
description: Panduan lengkap AI agent untuk plugin SLiMS Bulian: laporan, CRUD, filter, tab UI, dan penanganan submit form.
keywords: [slims plugin, ai agent, simbio, rag, laporan, crud, filter, tab, submit]
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

---

## ⚠️ Masalah Umum: Submit Form Redirect ke Halaman Default Admin

**Ini adalah masalah paling sering terjadi ketika AI agent membuat plugin dengan form.**

### Penyebab

Ketika form dalam plugin menggunakan `action=""` kosong, `action="/"`, atau tidak menyertakan parameter plugin yang tepat, SLiMS akan mengarahkan kembali ke halaman default admin setelah submit — **bukan** kembali ke halaman plugin.

### Solusi Wajib: Gunakan `pluginUrl()`

Selalu gunakan helper `pluginUrl()` sebagai nilai `action` pada setiap `<form>` di dalam file plugin:

```php
// ❌ SALAH - akan redirect ke halaman default admin setelah submit
// (form tanpa atribut action, atau action kosong/"/" berperilaku sama)
<form method="post" action="">
<form method="post" action="/">
<form method="post">   <!-- tanpa action pun tetap salah -->

// ✅ BENAR - form kembali ke halaman plugin yang sama setelah submit
<form method="post" action="<?= pluginUrl() ?>">

// ✅ BENAR - form dengan parameter tambahan (misal action=save)
<form method="post" action="<?= pluginUrl(['action' => 'save']) ?>">

// ✅ BENAR - reset URL ke state awal plugin (tanpa parameter ekstra)
<form method="get" action="<?= pluginUrl(reset: true) ?>">
```

`pluginUrl()` menghasilkan URL lengkap seperti:
```
/admin/plugin_container.php?mod=system&id=a86efea58....
```

URL ini memastikan SLiMS tetap berada di dalam konteks plugin yang sedang aktif.

### Pola Lengkap: Proses POST Tetap di Halaman Plugin

```php
<?php
// Tangani submit form SEBELUM output HTML apapun
if ($_SERVER['REQUEST_METHOD'] === 'POST' && isset($_POST['action'])) {
    $action = $_POST['action'];

    if ($action === 'save') {
        // proses simpan data...
        $nama = trim(strip_tags($_POST['nama'] ?? ''));

        // setelah selesai, redirect balik ke plugin dengan pesan sukses
        flash('pesan_sukses', 'Data berhasil disimpan!');
        redirect()->simbioAJAX(pluginUrl());
        exit;
    }

    if ($action === 'delete') {
        // proses hapus data...
        flash('pesan_sukses', 'Data berhasil dihapus!');
        redirect()->simbioAJAX(pluginUrl());
        exit;
    }
}

// Tampilkan pesan flash jika ada
if (flash()->has('pesan_sukses')) {
    toastr(flash()->get('pesan_sukses'))->success('Berhasil');
}

// Render form dengan action yang benar
?>
<form method="post" action="<?= pluginUrl() ?>">
    <input type="hidden" name="action" value="save">
    <input type="text" name="nama" value="">
    <button type="submit">Simpan</button>
</form>
```

---

## Contoh 1: Plugin Laporan (Report)

Plugin laporan menampilkan data berdasarkan filter (misal rentang tanggal), lalu merender tabel hasil.

### Struktur File

```
plugins/
└── laporan_peminjaman/
    ├── laporan_peminjaman.plugin.php   ← registrasi plugin
    └── index.php                       ← halaman laporan + filter
```

### File: `laporan_peminjaman.plugin.php`

```php
<?php
/**
 * Plugin Name: Laporan Peminjaman
 * Plugin URI: -
 * Description: Laporan rekapitulasi data peminjaman berdasarkan rentang tanggal
 * Version: 1.0.0
 * Author: Tim Perpustakaan
 * Author URI: -
 */
use SLiMS\Plugins;

$plugins = Plugins::getInstance();

// Daftarkan menu di modul reporting
Plugins::menu('reporting', 'Laporan Peminjaman', __DIR__ . '/index.php');
```

### File: `index.php`

```php
<?php
// Ambil parameter filter dari form (GET karena ini filter/pencarian)
$tgl_awal  = isset($_GET['tgl_awal'])  ? trim($_GET['tgl_awal'])  : date('Y-m-01');
$tgl_akhir = isset($_GET['tgl_akhir']) ? trim($_GET['tgl_akhir']) : date('Y-m-d');

// Validasi format tanggal
if (!strtotime($tgl_awal))  $tgl_awal  = date('Y-m-01');
if (!strtotime($tgl_akhir)) $tgl_akhir = date('Y-m-d');

// Ambil data dari database SLiMS
// Gunakan $dbs yang sudah tersedia secara global di area admin SLiMS
$sql = "SELECT
            l.loan_id,
            m.member_name,
            b.title,
            l.loan_date,
            l.due_date,
            l.return_date
        FROM loan l
        LEFT JOIN member m ON l.member_id = m.member_id
        LEFT JOIN biblio b ON l.biblio_id = b.biblio_id
        WHERE DATE(l.loan_date) BETWEEN ? AND ?
        ORDER BY l.loan_date DESC";

$result = $dbs->query($sql, [$tgl_awal, $tgl_akhir]); // parameter ke-2 adalah array bind value — mencegah SQL injection
$rows   = $result->fetchAll();
?>

<div class="menuBox">
    <div class="menuBoxInner reporting_icon">
        <div class="per_title">
            <h2>Laporan Peminjaman</h2>
        </div>
    </div>
</div>

<!-- Form Filter — action WAJIB pluginUrl(reset: true) agar kembali ke plugin ini -->
<fieldset>
    <legend>Filter Laporan</legend>
    <form method="get" action="<?= pluginUrl(reset: true) ?>">
        <table>
            <tr>
                <td>Tanggal Awal</td>
                <td>
                    <input type="date" name="tgl_awal"
                           value="<?= htmlspecialchars($tgl_awal) ?>">
                </td>
            </tr>
            <tr>
                <td>Tanggal Akhir</td>
                <td>
                    <input type="date" name="tgl_akhir"
                           value="<?= htmlspecialchars($tgl_akhir) ?>">
                </td>
            </tr>
            <tr>
                <td colspan="2">
                    <button type="submit" class="operationButton">Tampilkan</button>
                </td>
            </tr>
        </table>
    </form>
</fieldset>

<!-- Tabel Hasil -->
<fieldset>
    <legend>Hasil (<?= count($rows) ?> data)</legend>
    <table class="s-table" width="100%">
        <thead>
            <tr>
                <th>No</th>
                <th>Nama Anggota</th>
                <th>Judul</th>
                <th>Tanggal Pinjam</th>
                <th>Batas Kembali</th>
                <th>Tanggal Kembali</th>
                <th>Status</th>
            </tr>
        </thead>
        <tbody>
        <?php if (empty($rows)): ?>
            <tr><td colspan="7" align="center">Tidak ada data</td></tr>
        <?php else: ?>
            <?php foreach ($rows as $i => $row): ?>
            <tr>
                <td><?= $i + 1 ?></td>
                <td><?= htmlspecialchars($row['member_name']) ?></td>
                <td><?= htmlspecialchars($row['title']) ?></td>
                <td><?= $row['loan_date'] ?></td>
                <td><?= $row['due_date'] ?></td>
                <td><?= $row['return_date'] ?? '-' ?></td>
                <td><?= $row['return_date'] ? 'Sudah Kembali' : '<span style="color:red">Belum Kembali</span>' ?></td>
            </tr>
            <?php endforeach; ?>
        <?php endif; ?>
        </tbody>
    </table>
</fieldset>
```

---

## Contoh 2: Plugin CRUD Data dengan Tab

Plugin CRUD mengelola data kustom (tambah, edit, hapus) dengan tampilan tab untuk memisahkan antara daftar data, form tambah, dan form edit.

### Struktur File

```
plugins/
└── data_kustom/
    ├── data_kustom.plugin.php   ← registrasi plugin
    └── index.php                ← CRUD + tab UI
```

### File: `data_kustom.plugin.php`

```php
<?php
/**
 * Plugin Name: Data Kustom
 * Plugin URI: -
 * Description: Pengelolaan data kustom dengan tab UI dan CRUD lengkap
 * Version: 1.0.0
 * Author: Tim Perpustakaan
 * Author URI: -
 */
use SLiMS\Plugins;

$plugins = Plugins::getInstance();

// Daftarkan menu di modul master_file
Plugins::menu('master_file', 'Data Kustom', __DIR__ . '/index.php');
```

### File: `index.php`

```php
<?php
// =========================================================
// PROSES POST — tangani SEBELUM output HTML apapun
// Gunakan pluginUrl() sebagai action agar tetap di halaman plugin
// =========================================================
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $action = $_POST['action'] ?? '';

    if ($action === 'save_add') {
        $nama     = trim(strip_tags($_POST['nama'] ?? ''));
        $deskripsi = trim(strip_tags($_POST['deskripsi'] ?? ''));

        if ($nama !== '') {
            $dbs->query(
                "INSERT INTO custom_data (nama, deskripsi, created_at) VALUES (?, ?, NOW())",
                [$nama, $deskripsi]
            );
            flash('msg_sukses', "Data \"$nama\" berhasil ditambahkan.");
        } else {
            flash('msg_error', 'Nama tidak boleh kosong.');
        }

        // ✅ Redirect balik ke plugin — BUKAN ke halaman default admin
        redirect()->simbioAJAX(pluginUrl());
        exit;
    }

    if ($action === 'save_edit') {
        $id        = (int)($_POST['id'] ?? 0);
        $nama      = trim(strip_tags($_POST['nama'] ?? ''));
        $deskripsi = trim(strip_tags($_POST['deskripsi'] ?? ''));

        if ($id > 0 && $nama !== '') {
            $dbs->query(
                "UPDATE custom_data SET nama = ?, deskripsi = ?, updated_at = NOW() WHERE id = ?",
                [$nama, $deskripsi, $id]
            );
            flash('msg_sukses', "Data berhasil diperbarui.");
        } else {
            flash('msg_error', 'Data tidak valid.');
        }

        redirect()->simbioAJAX(pluginUrl());
        exit;
    }

    if ($action === 'delete') {
        $id = (int)($_POST['id'] ?? 0);
        if ($id > 0) {
            $dbs->query("DELETE FROM custom_data WHERE id = ?", [$id]);
            flash('msg_sukses', 'Data berhasil dihapus.');
        }

        redirect()->simbioAJAX(pluginUrl());
        exit;
    }
}

// =========================================================
// Tentukan tab aktif dari query string
// =========================================================
$tab_aktif = $_GET['tab'] ?? 'list';
$edit_id   = (int)($_GET['edit_id'] ?? 0);

// Ambil data untuk tab edit jika ada
$data_edit = null;
if ($tab_aktif === 'edit' && $edit_id > 0) {
    $res = $dbs->query("SELECT * FROM custom_data WHERE id = ? LIMIT 1", [$edit_id]);
    $data_edit = $res->fetch();
    if (!$data_edit) {
        $tab_aktif = 'list'; // data tidak ditemukan, kembali ke list
    }
}

// Ambil semua data untuk tab list
$semua_data = [];
if ($tab_aktif === 'list') {
    $res        = $dbs->query("SELECT * FROM custom_data ORDER BY id DESC");
    $semua_data = $res->fetchAll();
}

// Tampilkan pesan flash
if (flash()->has('msg_sukses')) {
    toastr(flash()->get('msg_sukses'))->success('Berhasil');
}
if (flash()->has('msg_error')) {
    toastr(flash()->get('msg_error'))->error('Gagal');
}
?>

<div class="menuBox">
    <div class="menuBoxInner master_file_icon">
        <div class="per_title">
            <h2>Data Kustom</h2>
        </div>
    </div>
</div>

<!-- =========================================================
     Tab Navigation
     URL tab menggunakan pluginUrl() agar tetap di halaman plugin
     ========================================================= -->
<div id="content-menu">
    <div id="content-menu-inside">
        <a href="<?= pluginUrl(['tab' => 'list']) ?>"
           class="<?= $tab_aktif === 'list' ? 'active' : '' ?>">
            Daftar Data
        </a>
        <a href="<?= pluginUrl(['tab' => 'add']) ?>"
           class="<?= $tab_aktif === 'add' ? 'active' : '' ?>">
            Tambah Data
        </a>
        <?php if ($tab_aktif === 'edit' && $data_edit): ?>
        <a href="<?= pluginUrl(['tab' => 'edit', 'edit_id' => $edit_id]) ?>"
           class="active">
            Edit Data
        </a>
        <?php endif; ?>
    </div>
</div>

<!-- =========================================================
     Tab: Daftar Data
     ========================================================= -->
<?php if ($tab_aktif === 'list'): ?>
<fieldset>
    <legend>Daftar Data</legend>
    <table class="s-table" width="100%">
        <thead>
            <tr>
                <th>ID</th>
                <th>Nama</th>
                <th>Deskripsi</th>
                <th>Dibuat</th>
                <th>Aksi</th>
            </tr>
        </thead>
        <tbody>
        <?php if (empty($semua_data)): ?>
            <tr><td colspan="5" align="center">Belum ada data</td></tr>
        <?php else: ?>
            <?php foreach ($semua_data as $row): ?>
            <tr>
                <td><?= $row['id'] ?></td>
                <td><?= htmlspecialchars($row['nama']) ?></td>
                <td><?= htmlspecialchars($row['deskripsi']) ?></td>
                <td><?= $row['created_at'] ?></td>
                <td>
                    <!-- Tombol Edit: buka tab edit dengan edit_id -->
                    <a href="<?= pluginUrl(['tab' => 'edit', 'edit_id' => $row['id']]) ?>"
                       class="operationButton">Edit</a>

                    <!-- Tombol Hapus: form POST dengan action delete -->
                    <!-- action WAJIB pluginUrl() agar tidak redirect ke halaman default admin -->
                    <form method="post" action="<?= pluginUrl() ?>"
                          style="display:inline"
                          onsubmit="return confirm('Yakin hapus data ini?')">
                        <input type="hidden" name="action" value="delete">
                        <input type="hidden" name="id" value="<?= $row['id'] ?>">
                        <button type="submit" class="operationButton deleteButton">Hapus</button>
                    </form>
                </td>
            </tr>
            <?php endforeach; ?>
        <?php endif; ?>
        </tbody>
    </table>
</fieldset>

<!-- =========================================================
     Tab: Tambah Data
     ========================================================= -->
<?php elseif ($tab_aktif === 'add'): ?>
<fieldset>
    <legend>Tambah Data Baru</legend>
    <!-- action WAJIB pluginUrl() agar form POST kembali ke plugin ini -->
    <form method="post" action="<?= pluginUrl() ?>">
        <input type="hidden" name="action" value="save_add">
        <table>
            <tr>
                <td width="150">Nama <span style="color:red">*</span></td>
                <td>
                    <input type="text" name="nama" maxlength="100"
                           style="width:300px" required>
                </td>
            </tr>
            <tr>
                <td>Deskripsi</td>
                <td>
                    <textarea name="deskripsi" rows="4"
                              style="width:300px"></textarea>
                </td>
            </tr>
            <tr>
                <td colspan="2">
                    <button type="submit" class="operationButton">Simpan</button>
                    <a href="<?= pluginUrl(['tab' => 'list']) ?>"
                       class="operationButton">Batal</a>
                </td>
            </tr>
        </table>
    </form>
</fieldset>

<!-- =========================================================
     Tab: Edit Data
     ========================================================= -->
<?php elseif ($tab_aktif === 'edit' && $data_edit): ?>
<fieldset>
    <legend>Edit Data (ID: <?= $data_edit['id'] ?>)</legend>
    <!-- action WAJIB pluginUrl() agar form POST kembali ke plugin ini -->
    <form method="post" action="<?= pluginUrl() ?>">
        <input type="hidden" name="action" value="save_edit">
        <input type="hidden" name="id" value="<?= $data_edit['id'] ?>">
        <table>
            <tr>
                <td width="150">Nama <span style="color:red">*</span></td>
                <td>
                    <input type="text" name="nama" maxlength="100"
                           value="<?= htmlspecialchars($data_edit['nama']) ?>"
                           style="width:300px" required>
                </td>
            </tr>
            <tr>
                <td>Deskripsi</td>
                <td>
                    <textarea name="deskripsi" rows="4"
                              style="width:300px"><?= htmlspecialchars($data_edit['deskripsi']) ?></textarea>
                </td>
            </tr>
            <tr>
                <td colspan="2">
                    <button type="submit" class="operationButton">Update</button>
                    <a href="<?= pluginUrl(['tab' => 'list']) ?>"
                       class="operationButton">Batal</a>
                </td>
            </tr>
        </table>
    </form>
</fieldset>
<?php endif; ?>
```

---

## Contoh 3: Plugin Filter Advance (Pencarian Lanjutan)

Plugin filter advance memberikan form pencarian dengan banyak parameter, berguna untuk modul yang memerlukan filtering data kompleks (misal pencarian bibliografi berdasarkan beberapa kriteria sekaligus).

### Struktur File

```
plugins/
└── filter_advance_biblio/
    ├── filter_advance_biblio.plugin.php   ← registrasi plugin
    └── index.php                          ← form filter + hasil
```

### File: `filter_advance_biblio.plugin.php`

```php
<?php
/**
 * Plugin Name: Filter Advance Bibliografi
 * Plugin URI: -
 * Description: Pencarian lanjutan bibliografi dengan multi-kriteria
 * Version: 1.0.0
 * Author: Tim Perpustakaan
 * Author URI: -
 */
use SLiMS\Plugins;

$plugins = Plugins::getInstance();

// Daftarkan menu di modul bibliography
Plugins::menu('bibliography', 'Filter Advance', __DIR__ . '/index.php');
```

### File: `index.php`

```php
<?php
// =========================================================
// Ambil parameter filter dari form GET
// =========================================================
$keyword    = trim(strip_tags($_GET['keyword']    ?? ''));
$tahun_dari = trim($_GET['tahun_dari'] ?? '');
$tahun_ke   = trim($_GET['tahun_ke']   ?? '');
$bahasa     = trim($_GET['bahasa']     ?? '');
$gmd        = (int)($_GET['gmd']       ?? 0);
$sudah_cari = isset($_GET['cari']);  // apakah form sudah disubmit

// Ambil opsi untuk dropdown (dari database SLiMS)
$bahasa_options = $dbs->query("SELECT language_id, language_name FROM mst_language ORDER BY language_name")->fetchAll();
$gmd_options    = $dbs->query("SELECT gmd_id, gmd_name FROM mst_gmd ORDER BY gmd_name")->fetchAll();

// =========================================================
// Bangun query dinamis berdasarkan filter yang diisi
// =========================================================
$rows        = [];
$total_hasil = 0;

if ($sudah_cari) {
    $kondisi = ['1=1'];
    $params  = [];

    if ($keyword !== '') {
        $kondisi[] = "(b.title LIKE ? OR b.author LIKE ? OR b.isbn_issn LIKE ?)";
        $params[]  = "%$keyword%";
        $params[]  = "%$keyword%";
        $params[]  = "%$keyword%";
    }

    if ($tahun_dari !== '') {
        $kondisi[] = "b.publish_year >= ?";
        $params[]  = $tahun_dari;
    }

    if ($tahun_ke !== '') {
        $kondisi[] = "b.publish_year <= ?";
        $params[]  = $tahun_ke;
    }

    if ($bahasa !== '') {
        $kondisi[] = "b.language_id = ?";
        $params[]  = $bahasa;
    }

    if ($gmd > 0) {
        $kondisi[] = "b.gmd_id = ?";
        $params[]  = $gmd;
    }

    $where = implode(' AND ', $kondisi);
    $sql   = "SELECT b.biblio_id, b.title, b.author, b.publish_year,
                     l.language_name, g.gmd_name
              FROM biblio b
              LEFT JOIN mst_language l ON b.language_id = l.language_id
              LEFT JOIN mst_gmd g ON b.gmd_id = g.gmd_id
              WHERE $where
              ORDER BY b.title
              LIMIT 200";

    $result      = $dbs->query($sql, $params);
    $rows        = $result->fetchAll();
    $total_hasil = count($rows);
}
?>

<div class="menuBox">
    <div class="menuBoxInner bibliography_icon">
        <div class="per_title">
            <h2>Filter Advance Bibliografi</h2>
        </div>
    </div>
</div>

<!-- Form Filter Advance
     Gunakan method GET + pluginUrl(reset: true) agar filter bisa di-bookmark
     dan tetap berada di halaman plugin setelah submit -->
<fieldset>
    <legend>Parameter Pencarian</legend>
    <form method="get" action="<?= pluginUrl(reset: true) ?>">
        <!-- Penanda bahwa form sudah disubmit -->
        <input type="hidden" name="cari" value="1">

        <table>
            <tr>
                <td width="150">Kata Kunci</td>
                <td>
                    <input type="text" name="keyword"
                           value="<?= htmlspecialchars($keyword) ?>"
                           placeholder="Judul / Pengarang / ISBN"
                           style="width:300px">
                </td>
            </tr>
            <tr>
                <td>Tahun Terbit</td>
                <td>
                    <input type="number" name="tahun_dari"
                           value="<?= htmlspecialchars($tahun_dari) ?>"
                           placeholder="Dari" style="width:80px" min="1900" max="2100">
                    &nbsp;s/d&nbsp;
                    <input type="number" name="tahun_ke"
                           value="<?= htmlspecialchars($tahun_ke) ?>"
                           placeholder="Ke" style="width:80px" min="1900" max="2100">
                </td>
            </tr>
            <tr>
                <td>Bahasa</td>
                <td>
                    <select name="bahasa" style="width:200px">
                        <option value="">-- Semua Bahasa --</option>
                        <?php foreach ($bahasa_options as $b): ?>
                        <option value="<?= htmlspecialchars($b['language_id']) ?>"
                            <?= $bahasa === $b['language_id'] ? 'selected' : '' ?>>
                            <?= htmlspecialchars($b['language_name']) ?>
                        </option>
                        <?php endforeach; ?>
                    </select>
                </td>
            </tr>
            <tr>
                <td>Jenis Media (GMD)</td>
                <td>
                    <select name="gmd" style="width:200px">
                        <option value="0">-- Semua GMD --</option>
                        <?php foreach ($gmd_options as $g): ?>
                        <option value="<?= (int)$g['gmd_id'] ?>"
                            <?= $gmd === (int)$g['gmd_id'] ? 'selected' : '' ?>>
                            <?= htmlspecialchars($g['gmd_name']) ?>
                        </option>
                        <?php endforeach; ?>
                    </select>
                </td>
            </tr>
            <tr>
                <td colspan="2">
                    <button type="submit" class="operationButton">Cari</button>
                    <a href="<?= pluginUrl(reset: true) ?>"
                       class="operationButton">Reset</a>
                </td>
            </tr>
        </table>
    </form>
</fieldset>

<!-- Hasil Pencarian (hanya tampil jika form sudah disubmit) -->
<?php if ($sudah_cari): ?>
<fieldset>
    <legend>Hasil Pencarian (<?= $total_hasil ?> data ditemukan)</legend>
    <?php if (empty($rows)): ?>
        <p>Tidak ada data yang sesuai dengan kriteria pencarian.</p>
    <?php else: ?>
    <table class="s-table" width="100%">
        <thead>
            <tr>
                <th>No</th>
                <th>Judul</th>
                <th>Pengarang</th>
                <th>Tahun</th>
                <th>Bahasa</th>
                <th>GMD</th>
            </tr>
        </thead>
        <tbody>
        <?php foreach ($rows as $i => $row): ?>
            <tr>
                <td><?= $i + 1 ?></td>
                <td><?= htmlspecialchars($row['title']) ?></td>
                <td><?= htmlspecialchars($row['author']) ?></td>
                <td><?= $row['publish_year'] ?></td>
                <td><?= htmlspecialchars($row['language_name'] ?? '-') ?></td>
                <td><?= htmlspecialchars($row['gmd_name'] ?? '-') ?></td>
            </tr>
        <?php endforeach; ?>
        </tbody>
    </table>
    <?php endif; ?>
</fieldset>
<?php endif; ?>
```

---

## Ringkasan Pola Tab dan Submit yang Benar

| Kebutuhan | Pola yang Digunakan |
|---|---|
| Form simpan/tambah/edit (POST) | `action="<?= pluginUrl() ?>"` |
| Form filter/pencarian (GET) | `action="<?= pluginUrl(reset: true) ?>"` |
| Redirect setelah POST | `redirect()->simbioAJAX(pluginUrl())` lalu `exit` |
| Link pindah tab | `href="<?= pluginUrl(['tab' => 'nama_tab']) ?>"` |
| Link edit dengan ID | `href="<?= pluginUrl(['tab' => 'edit', 'edit_id' => $id]) ?>"` |
| Reset ke tampilan awal plugin | `pluginUrl(reset: true)` |

---

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
- Setiap <form> WAJIB menggunakan action="<?= pluginUrl() ?>" agar tidak redirect ke halaman default admin.
- Setelah proses POST selesai, WAJIB redirect()->simbioAJAX(pluginUrl()) lalu exit.

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
