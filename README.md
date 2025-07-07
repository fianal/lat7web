# lab7web

<h1>Praktikum 7: Relasi Tabel dan Query Builder</h1>

| Data diri| 
|-----------------|
| Nama : Alfian Nur Rizki  | 
| Kelas : TI 23.A6 | 
| NIM : 312310665 | 

## Membuat Tabel Kategori

|  Field                 | Tipe Data  | Ukuran | Keterangan |
|---------------------------|----------|-------|--------|
| id_kategori|    INT   |     11    | PRIMARY KEY, auto_increment |
| nama_kategori |   VARCHAR    |     100    | |
| slug_kategori |   VARCHAR    |    100     | |

<p>Jalankan query berikut</p>

```
CREATE TABLE kategori (
    id_kategori INT(11) AUTO_INCREMENT,
    nama_kategori VARCHAR(100) NOT NULL,
    slug_kategori VARCHAR(100),
    PRIMARY KEY (id_kategori)
  );
```

## Mengubah Tabel Artikel

<p>Tambahkan foreign key `id_kategori` pada tabel `artikel` untuk membuat relasi dengan tabel`kategori`.</p>

<p>Query untuk menambahkan Foreign Key</p>

```
ALTER TABLE artikel
ADD COLUMN id_kategori INT(11),
ADD CONSTRAINT fk_kategori_artikel
FOREIGN KEY (id_kategori) REFERENCES kategori(id_kategori);
```

## Mebuat Model Kategori

<p>Buat file model baru di `app/Models` dengan nama `KategoriModel.php`:</p>

```
<?php
namespace App\Models;
use CodeIgniter\Model;
class KategoriModel extends Model
{
    protected $table = 'kategori';
    protected $primaryKey = 'id_kategori';
    protected $useAutoIncrement = true;
    protected $allowedFields = [nama_kategori', 'slug_kategori'];
}
```

## Memodifikasi Model Artikel

<p>Modifikasi `ArtikelModel.php` untuk mendefinisikan relasi dengan `KategoriModel`:</p>

```
<?php

namespace App\Models;

use CodeIgniter\Model;

class ArtikelModel extends Model
{
    protected $table            = 'artikel';
    protected $primaryKey       = 'id';
    protected $useAutoIncrement = true;
    protected $allowedFields    = ['judul', 'isi', 'status', 'slug', 'gambar', 'id_kategori'];

    public function getArtikelDenganKategori()
    {
        return $this->db->table('artikel')
                        ->select('artikel.*, kategori.nama_kategori')
                        ->join('kategori', 'kategori.id_kategori = artikel.id_kategori')
                        ->get()
                        ->getResultArray();
    }

    public function getArtikelByKategori($slug_kategori)
    {
        return $this->db->table('artikel')
                        ->select('artikel.*, kategori.nama_kategori')
                        ->join('kategori', 'kategori.id_kategori = artikel.id_kategori')
                        ->where('kategori.slug_kategori', $slug_kategori)
                        ->get()
                        ->getResultArray();
    }

}

```

<p>Menambahkan method `getArtikelDenganKategori()` untuk mengambil data artikel beserta nama kategorinya menggunakan join.</p>

## Memodifikasi Controller Artikel

<p>Modifikasi `Artikel.php` untuk menggunakan model baru dan menampilkan data relasi:</p>

```
<?php

namespace App\Controllers;

use App\Models\ArtikelModel;
use App\Models\KategoriModel;

class Artikel extends BaseController
{
    public function index()
    {
        $title = 'Daftar Artikel';
        $model = new ArtikelModel();
        $artikel = $model->getArtikelDenganKategori(); // Use the new method
        return view('artikel/index', compact('artikel', 'title'));
    }

    public function admin_index()
    {
        $title = 'Daftar Artikel (Admin)';
        $model = new ArtikelModel();

        // Get search keyword
        $q = $this->request->getVar('q') ?? '';

        // Get category filter
        $kategori_id = $this->request->getVar('kategori_id') ?? '';

        $data = [
            'title' => $title,
            'q' => $q,
            'kategori_id' => $kategori_id,
        ];

        // Building the query
        $builder = $model->table('artikel')
            ->select('artikel.*, kategori.nama_kategori')
            ->join('kategori', 'kategori.id_kategori = artikel.id_kategori');

        // Apply search filter if keyword is provided
        if ($q != '') {
            $builder->like('artikel.judul', $q);
        }

        // Apply category filter if category_id is provided
        if ($kategori_id != '') {
            $builder->where('artikel.id_kategori', $kategori_id);
        }

        // Apply pagination
        $data['artikel'] = $builder->paginate(10);
        $data['pager'] = $model->pager;

        // Fetch all categories for the filter dropdown
        $kategoriModel = new KategoriModel();
        $data['kategori'] = $kategoriModel->findAll();

        return view('artikel/admin_index', $data);
    }

    public function add()
    {
        // Validation...
        if (
            $this->request->getMethod() == 'post' &&
            $this->validate([
                'judul' => 'required',
                'id_kategori' => 'required|integer' // Ensure id_kategori is required and an integer
            ])
        ) {
            $model = new ArtikelModel();
            $model->insert([
                'judul' => $this->request->getPost('judul'),
                'isi' => $this->request->getPost('isi'),
                'slug' => url_title($this->request->getPost('judul')),
                'id_kategori' => $this->request->getPost('id_kategori')
            ]);
            return redirect()->to('/admin/artikel');
        } else {
            $kategoriModel = new KategoriModel();
            $data['kategori'] = $kategoriModel->findAll(); // Fetch categories for the form
            $data['title'] = "Tambah Artikel";
            return view('artikel/form_add', $data);
        }
    }

    public function edit($id)
    {
        $model = new ArtikelModel();
        if (
            $this->request->getMethod() == 'post' &&
            $this->validate([
                'judul' => 'required',
                'id_kategori' => 'required|integer'
            ])
        ) {
            $model->update($id, [
                'judul' => $this->request->getPost('judul'),
                'isi' => $this->request->getPost('isi'),
                'id_kategori' => $this->request->getPost('id_kategori')
            ]);
            return redirect()->to('/admin/artikel');
        } else {
            $data['artikel'] = $model->find($id);
            $kategoriModel = new KategoriModel();
            $data['kategori'] = $kategoriModel->findAll(); // Fetch categories for the form
            $data['title'] = "Edit Artikel";
            return view('artikel/form_edit', $data);
        }
    }

    public function delete($id)
    {
        $model = new ArtikelModel();
        $model->delete($id);
        return redirect()->to('/admin/artikel');
    }

    public function view($slug)
    {
        $model = new ArtikelModel();
        $data['artikel'] = $model->where('slug', $slug)->first();
        if (empty($data['artikel'])) {
            throw new \CodeIgniter\Exceptions\PageNotFoundException('Cannot find the article.');
        }
        $data['title'] = $data['artikel']['judul'];
        return view('artikel/detail', $data);
    }
}

```

## Memodifikasi View

<p>Buka folder view/artikel sesuaikan masing-masing view.</p>

**index.php**

```
<?= $this->include('template/header'); ?>

<?php if ($artikel): foreach ($artikel as $row): ?>
    <article class="entry">
        <h2>
            <a href="<?= base_url('/artikel/' . $row['slug']); ?>">
                <?= $row['judul']; ?>
            </a>
        </h2>
        <p>Kategori: <?= $row['nama_kategori'] ?></p>
        <img src="<?= base_url('/gambar/' . $row['gambar']); ?>" alt="<?= $row['judul']; ?>">
        <p><?= substr($row['isi'], 0, 200); ?></p>
    </article>
    <hr class="divider" />
<?php endforeach; else: ?>
    <article class="entry">
        <h2>Belum ada data.</h2>
    </article>
<?php endif; ?>

<?= $this->include('template/footer'); ?>

```

**admin_index.php**

```
<?= $this->include('template/admin_header'); ?>

<h2><?= $title; ?></h2>

<div class="row mb-3">
    <div class="col-md-6">
        <form method="get" class="form-inline">
            <input type="text" name="q" value="<?= $q; ?>" placeholder="Cari judul artikel" class="form-control mr-2">

            <select name="kategori_id" class="form-control mr-2">
                <option value="">Semua Kategori</option>
                <?php foreach ($kategori as $k): ?>
                    <option value="<?= $k['id_kategori']; ?>" <?= ($kategori_id == $k['id_kategori']) ? 'selected' : ''; ?>>
                        <?= $k['nama_kategori']; ?>
                    </option>
                <?php endforeach; ?>
            </select>

            <input type="submit" value="Cari" class="btn btn-primary">
        </form>
    </div>
</div>

<table class="table">
    <thead>
        <tr>
            <th>ID</th>
            <th>Judul</th>
            <th>Kategori</th>
            <th>Status</th>
            <th>Aksi</th>
        </tr>
    </thead>
    <tbody>
        <?php if (count($artikel) > 0): ?>
            <?php foreach ($artikel as $row): ?>
                <tr>
                    <td><?= $row->id; ?></td>
                    <td>
                        <b><?= $row->judul; ?></b>
                        <p><small><?= substr($row->isi, 0, 50); ?></small></p>
                    </td>
                    <td><?= $row->nama_kategori; ?></td>
                    <td><?= $row->status; ?></td>
                    <td>
                        <a class="btn btn-sm btn-info" href="<?= base_url('/admin/artikel/edit/' . $row->id); ?>">Ubah</a>
                        <a class="btn btn-sm btn-danger" onclick="return confirm('Yakin menghapus data?');" href="<?= base_url('/admin/artikel/delete/' . $row->id); ?>">Hapus</a>
                    </td>
                </tr>
            <?php endforeach; ?>
        <?php else: ?>
            <tr>
                <td colspan="5">Tidak ada data.</td>
            </tr>
        <?php endif; ?>
    </tbody>
</table>

<?= $pager->only(['q', 'kategori_id'])->links(); ?>

<?= $this->include('template/admin_footer'); ?>

```

**form_add.php**

```
<?= $this->include('template/admin_header'); ?>

<h2><?= $title; ?></h2>

<form action="" method="post">
    <p>
        <label for="judul">Judul</label>
        <input type="text" name="judul" id="judul" required>
    </p>
    
    <p>
        <label for="isi">Isi</label>
        <textarea name="isi" id="isi" cols="50" rows="10"></textarea>
    </p>
    
    <p>
        <label for="id_kategori">Kategori</label>
        <select name="id_kategori" id="id_kategori" required>
            <?php foreach ($kategori as $k): ?>
                <option value="<?= $k['id_kategori']; ?>">
                    <?= $k['nama_kategori']; ?>
                </option>
            <?php endforeach; ?>
        </select>
    </p>
    
    <p>
        <input type="submit" value="Kirim" class="btn btn-large">
    </p>
</form>

<?= $this->include('template/admin_footer'); ?>

```

**form_edit.php**

```
<?= $this->include('template/admin_header'); ?>

<h2><?= $title; ?></h2>

<form action="" method="post">
    <p>
        <label for="judul">Judul</label>
        <input type="text" name="judul" value="<?= $artikel['judul']; ?>" id="judul" required>
    </p>

    <p>
        <label for="isi">Isi</label>
        <textarea name="isi" id="isi" cols="50" rows="10"><?= $artikel['isi']; ?></textarea>
    </p>

    <p>
        <label for="id_kategori">Kategori</label>
        <select name="id_kategori" id="id_kategori" required>
            <?php foreach ($kategori as $k): ?>
                <option value="<?= $k['id_kategori']; ?>" <?= ($artikel['id_kategori'] == $k['id_kategori']) ? 'selected' : ''; ?>>
                    <?= $k['nama_kategori']; ?>
                </option>
            <?php endforeach; ?>
        </select>
    </p>

    <p>
        <input type="submit" value="Kirim" class="btn btn-large">
    </p>
</form>

<?= $this->include('template/admin_footer'); ?>

```

## Testing

<p>Lakukan uji coba untuk memastikan semua fungsi berjalan dengan baik:</p>

• Menampilkan daftar artikel dengan nama kategori.

![image](https://github.com/user-attachments/assets/7a0fb03f-1505-4d59-a631-30e938f81abb)

• Menambah artikel baru dengan memilih kategori.

![image](https://github.com/user-attachments/assets/40d03e5f-5f17-4787-968f-80e3090cfd3c)

• Mengedit artikel dan mengubah kategorinya.

![image](https://github.com/user-attachments/assets/42f4167e-5ee7-4399-ac84-b4d948480438)

• Menghapus artikel.

![image](https://github.com/user-attachments/assets/ffa74d25-9d9c-4cf9-8d78-812ace41f10d)



