<?php
// ================== Koneksi Database ==================
$mysqli = new mysqli("localhost", "root", "", "kasir_meja");
if ($mysqli->connect_errno) die("Gagal koneksi: " . $mysqli->connect_error);

// ================== Buat Tabel Jika Belum Ada ==================
$mysqli->query("CREATE TABLE IF NOT EXISTS tables_no (
    id INT AUTO_INCREMENT PRIMARY KEY,
    table_number VARCHAR(20),
    status ENUM('free','occupied') DEFAULT 'free'
)");
$mysqli->query("CREATE TABLE IF NOT EXISTS products (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100),
    price INT,
    stock INT DEFAULT 0
)");
$mysqli->query("CREATE TABLE IF NOT EXISTS orders (
    id INT AUTO_INCREMENT PRIMARY KEY,
    table_id INT,
    total INT DEFAULT 0,
    bayar INT DEFAULT 0,
    kembalian INT DEFAULT 0,
    status ENUM('open','paid') DEFAULT 'open',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
)");
$mysqli->query("CREATE TABLE IF NOT EXISTS order_items (
    id INT AUTO_INCREMENT PRIMARY KEY,
    order_id INT,
    product_id INT,
    qty INT,
    price INT,
    subtotal INT
)");

// ================== CRUD Produk ==================
// Tambah Produk
if (isset($_POST['add_product'])) {
    $name = $_POST['name'];
    $price = $_POST['price'];
    $stock = $_POST['stock'];
    $mysqli->query("INSERT INTO products (name, price, stock) VALUES ('$name',$price,$stock)");
    header("Location: index.php?products=1");
    exit;
}

// Hapus Produk
if (isset($_GET['hapus_product'])) {
    $id = $_GET['hapus_product'];
    $mysqli->query("DELETE FROM products WHERE id=$id");
    header("Location: index.php?products=1");
    exit;
}

// Update Produk
if (isset($_POST['update_product'])) {
    $id = $_POST['id'];
    $name = $_POST['name'];
    $price = $_POST['price'];
    $stock = $_POST['stock'];
    $mysqli->query("UPDATE products SET name='$name', price=$price, stock=$stock WHERE id=$id");
    header("Location: index.php?products=1");
    exit;
}

// ================== Ambil Data ==================
$tables = $mysqli->query("SELECT * FROM tables_no ORDER BY id ASC");
$products = $mysqli->query("SELECT * FROM products ORDER BY id ASC");

// ================== Tambah Pesanan ==================
if (isset($_POST['add_order'])) {
    $table_id = $_POST['table_id'];
    $product_name = $_POST['product_name'];
    $qty = $_POST['qty'];
    $prod = $mysqli->query("SELECT * FROM products WHERE name='$product_name'")->fetch_assoc();
    if (!$prod) die("Produk tidak ditemukan!");
    $product_id = $prod['id'];
    $price = $prod['price'];
    $stock = $prod['stock'];
    if ($qty > $stock) die("Stok tidak cukup!");
    $order = $mysqli->query("SELECT * FROM orders WHERE table_id=$table_id AND status='open'")->fetch_assoc();
    if (!$order) {
        $mysqli->query("INSERT INTO orders (table_id) VALUES ($table_id)");
        $order_id = $mysqli->insert_id;
        $mysqli->query("UPDATE tables_no SET status='occupied' WHERE id=$table_id");
    } else $order_id = $order['id'];
    $subtotal = $price * $qty;
    $mysqli->query("INSERT INTO order_items (order_id, product_id, qty, price, subtotal) VALUES ($order_id, $product_id, $qty, $price, $subtotal)");
    $mysqli->query("UPDATE products SET stock = stock - $qty WHERE id = $product_id");
    $mysqli->query("UPDATE orders SET total=(SELECT SUM(subtotal) FROM order_items WHERE order_id=$order_id) WHERE id=$order_id");
    header("Location: index.php?table_id=$table_id");
    exit;
}

// ================== Hapus Item ==================
if (isset($_GET['hapus_item'])) {
    $item_id = $_GET['hapus_item'];
    $item = $mysqli->query("SELECT * FROM order_items WHERE id=$item_id")->fetch_assoc();
    $order_id = $item['order_id'];
    $table_id = $_GET['table_id'];
    $mysqli->query("UPDATE products SET stock = stock + {$item['qty']} WHERE id = {$item['product_id']}");
    $mysqli->query("DELETE FROM order_items WHERE id=$item_id");
    $mysqli->query("UPDATE orders SET total=(SELECT IFNULL(SUM(subtotal),0) FROM order_items WHERE order_id=$order_id) WHERE id=$order_id");
    header("Location: index.php?table_id=$table_id");
    exit;
}

// ================== Proses Pembayaran ==================
if (isset($_POST['pay_order'])) {
    $table_id = $_POST['table_id'];
    $order_id = $_POST['order_id'];
    $bayar = $_POST['bayar'];
    $order = $mysqli->query("SELECT * FROM orders WHERE id=$order_id")->fetch_assoc();
    $total = $order['total'];
    $kembalian = $bayar - $total;
    $mysqli->query("UPDATE orders SET status='paid', bayar=$bayar, kembalian=$kembalian WHERE id=$order_id");
    $mysqli->query("UPDATE tables_no SET status='free' WHERE id=$table_id");
    header("Location: index.php?print_struk=$order_id");
    exit;
}

// ================== Cetak Struk ==================
if (isset($_GET['print_struk'])) {
    $order_id = $_GET['print_struk'];
    $order = $mysqli->query("SELECT o.*, t.table_number FROM orders o JOIN tables_no t ON o.table_id=t.id WHERE o.id=$order_id")->fetch_assoc();
    $items = $mysqli->query("SELECT oi.*, p.name FROM order_items oi JOIN products p ON oi.product_id=p.id WHERE oi.order_id=$order_id"); ?>
    <!DOCTYPE html>
    <html>
    <head>
        <title>Struk</title>
        <style>
            body{font-family:monospace}
            .struk{width:250px}
            .center{text-align:center}
            table{width:100%}
            td{font-size:12px}
        </style>
    </head>
    <body onload="window.print()">
        <div class="struk">
            <div class="center">
                <h3>STRUK PEMBAYARAN</h3>
                <p>Meja: <?= $order['table_number'] ?><br><?= $order['created_at'] ?></p>
            </div>
            <hr>
            <table>
                <?php while($it=$items->fetch_assoc()): ?>
                <tr>
                    <td><?= $it['name'] ?> x<?= $it['qty'] ?></td>
                    <td style="text-align:right">Rp<?= number_format($it['subtotal']) ?></td>
                </tr>
                <?php endwhile; ?>
            </table>
            <hr>
            <p>Total : Rp<?= number_format($order['total']) ?></p>
            <p>Tunai : Rp<?= number_format($order['bayar']) ?></p>
            <p>Kembali: Rp<?= number_format($order['kembalian']) ?></p>
            <hr>
            <div class="center">Terima Kasih 🙏</div>
        </div>
    </body>
    </html>
<?php exit; } ?>

<!DOCTYPE html>
<html lang="id">
<head>
<meta charset="UTF-8">
<title>Kasir Meja</title>
<style>
body{font-family:sans-serif;background:#f4f6f8;margin:20px}
h2{color:#2e7d32}
nav{position:relative}
nav ul{display:flex;gap:10px;list-style:none;padding:0;margin:0}
nav ul li a{padding:8px 15px;background:#2e7d32;color:#fff;text-decoration:none;border-radius:6px;display:inline-block;margin-bottom:5px}
nav ul li a:hover{background:#256328}
.hamburger{display:none;flex-direction:column;cursor:pointer;width:30px;height:25px;justify-content:space-between;position:absolute;right:0;top:0}
.hamburger div{width:100%;height:4px;background-color:red;border-radius:2px}
table{width:100%;border-collapse:collapse;background:#fff;margin-top:10px}
th,td{padding:10px;border:1px solid #ddd;font-size:14px}
th{background:#2e7d32;color:#fff;font-size:14px}
input,button,select{padding:8px;margin:5px 0;border-radius:5px;border:1px solid #ccc;font-size:14px}
button{background:#2e7d32;color:#fff;border:none;cursor:pointer}
button:hover{background:#256328}
.tables a{display:inline-block;padding:15px;margin:10px 5px;background:#fff;border-radius:8px;text-align:center;text-decoration:none;font-weight:bold;min-width:80px}
.tables a.free{border:2px solid green;color:green}
.tables a.occupied{border:2px solid red;color:red}
form{display:flex;flex-wrap:wrap;gap:10px}
form input,form button,form select{flex:1 1 120px;min-width:100px}
@media(max-width:1024px){.hamburger{display:flex}nav ul{flex-direction:column;display:none;width:100%;background:#2e7d32;position:absolute;top:40px;left:0}nav ul.show{display:flex}nav ul li{width:100%;text-align:center;margin:5px 0}table,th,td{font-size:12px}.tables a{padding:12px;font-size:14px;min-width:60px}form input,form button,form select{flex:1 1 100%}}
@media(min-width:1025px){table{font-size:14px}.tables a{min-width:80px;font-size:16px}}
</style>
</head>
<body>
<h2>🍽️ Aplikasi Kasir Meja</h2>
<nav>
    <ul id="nav-links">
        <li><a href="index.php">🏠 Pilih Meja</a></li>
        <li><a href="index.php?riwayat=1">📜 Riwayat Transaksi</a></li>
        <li><a href="index.php?products=1">📦 Data Barang</a></li>
    </ul>
    <div class="hamburger" id="hamburger"><div></div><div></div><div></div></div>
</nav>
<hr>
<script>
const hamburger=document.getElementById('hamburger');
const navLinks=document.getElementById('nav-links');
hamburger.addEventListener('click',()=>{navLinks.classList.toggle('show');});
</script>

<?php
$filter = $_GET['filter'] ?? 'all';
$start = '';
if ($filter=='today') $start=date('Y-m-d 00:00:00');
elseif($filter=='week') $start=date('Y-m-d 00:00:00',strtotime('monday this week'));
elseif($filter=='month') $start=date('Y-m-01 00:00:00');
elseif($filter=='year') $start=date('Y-01-01 00:00:00');
?>

<?php if(isset($_GET['products'])): ?>
    <!-- ================== Data Barang ================== -->
    <h3>📦 Data Barang</h3>
    <form method="post">
        <input type="text" name="name" placeholder="Nama Barang" required>
        <input type="number" name="price" placeholder="Harga" required>
        <input type="number" name="stock" placeholder="Stok" required>
        <button type="submit" name="add_product">Tambah</button>
    </form>
    <form method="get" style="margin-top:10px;">
        <input type="hidden" name="products" value="1">
        <select name="filter" onchange="this.form.submit()">
            <option value="all" <?= $filter=='all'?'selected':''?>>Semua</option>
            <option value="today" <?= $filter=='today'?'selected':''?>>Hari Ini</option>
            <option value="week" <?= $filter=='week'?'selected':''?>>Minggu Ini</option>
            <option value="month" <?= $filter=='month'?'selected':''?>>Bulan Ini</option>
            <option value="year" <?= $filter=='year'?'selected':''?>>Tahun Ini</option>
        </select>
    </form>
    <table>
        <tr>
            <th>ID</th>
            <th>Nama</th>
            <th>Harga</th>
            <th>Stok</th>
            <th>Aksi</th>
        </tr>
        <?php
        $query="SELECT * FROM products";
        if($start) $query="SELECT p.* FROM products p JOIN order_items oi ON p.id=oi.product_id JOIN orders o ON o.id=oi.order_id WHERE o.created_at>='$start' GROUP BY p.id";
        $prod=$mysqli->query($query);
        while($p=$prod->fetch_assoc()):
        ?>
        <tr>
            <td><?= $p['id'] ?></td>
            <td>
                <?php if(isset($_GET['edit_product'])&&$_GET['edit_product']==$p['id']): ?>
                    <form method="post" style="display:flex;gap:5px;">
                        <input type="hidden" name="id" value="<?= $p['id'] ?>">
                        <input type="text" name="name" value="<?= $p['name'] ?>" required>
                <?php else: ?>
                    <?= $p['name'] ?>
                <?php endif; ?>
            </td>
            <td>
                <?php if(isset($_GET['edit_product'])&&$_GET['edit_product']==$p['id']): ?>
                        <input type="number" name="price" value="<?= $p['price'] ?>" required>
                <?php else: ?>
                    Rp<?= number_format($p['price']) ?>
                <?php endif; ?>
            </td>
            <td>
                <?php if(isset($_GET['edit_product'])&&$_GET['edit_product']==$p['id']): ?>
                        <input type="number" name="stock" value="<?= $p['stock'] ?>" required>
                <?php else: ?>
                    <?= $p['stock'] ?>
                <?php endif; ?>
            </td>
            <td>
                <?php if(isset($_GET['edit_product'])&&$_GET['edit_product']==$p['id']): ?>
                        <button type="submit" name="update_product">💾 Simpan</button>
                    </form>
                <?php else: ?>
                    <a href="index.php?products=1&edit_product=<?= $p['id'] ?>">✏️ Edit</a>
                    <a href="index.php?hapus_product=<?= $p['id'] ?>" onclick="return confirm('Hapus?')">🗑️ Hapus</a>
                <?php endif; ?>
            </td>
        </tr>
        <?php endwhile; ?>
    </table>

<?php elseif(isset($_GET['riwayat'])): ?>
    <!-- ================== Riwayat Transaksi ================== -->
    <?php
    $riwayat_query="SELECT * FROM orders WHERE status='paid'";
    if($start) $riwayat_query.=" AND created_at>='$start'";
    $riwayat_query.=" ORDER BY created_at DESC";
    $riwayat=$mysqli->query($riwayat_query);
    ?>
    <h3>📜 Riwayat Transaksi</h3>
    <form method="get" style="margin-bottom:10px;">
        <input type="hidden" name="riwayat" value="1">
        <select name="filter" onchange="this.form.submit()">
            <option value="all" <?= $filter=='all'?'selected':''?>>Semua</option>
            <option value="today" <?= $filter=='today'?'selected':''?>>Hari Ini</option>
            <option value="week" <?= $filter=='week'?'selected':''?>>Minggu Ini</option>
            <option value="month" <?= $filter=='month'?'selected':''?>>Bulan Ini</option>
            <option value="year" <?= $filter=='year'?'selected':''?>>Tahun Ini</option>
        </select>
    </form>
    <table>
        <tr>
            <th>No</th>
            <th>Meja</th>
            <th>Produk</th>
            <th>Total</th>
            <th>Tunai</th>
            <th>Kembalian</th>
            <th>Tanggal</th>
            <th>Aksi</th>
        </tr>
        <?php $no=1; while($r=$riwayat->fetch_assoc()):
            $produk=$mysqli->query("SELECT p.name,oi.qty FROM order_items oi JOIN products p ON oi.product_id=p.id WHERE oi.order_id=".$r['id']);
            $daftar=[];
            while($p=$produk->fetch_assoc()) $daftar[]=$p['name'].' x'.$p['qty'];
        ?>
        <tr>
            <td><?= $no++ ?></td>
            <td><?= $r['table_id'] ?></td>
            <td><?= implode(", ",$daftar) ?></td>
            <td>Rp<?= number_format($r['total']) ?></td>
            <td>Rp<?= number_format($r['bayar']) ?></td>
            <td>Rp<?= number_format($r['kembalian']) ?></td>
            <td><?= $r['created_at'] ?></td>
            <td><a href="index.php?print_struk=<?= $r['id'] ?>">🖨️</a></td>
        </tr>
        <?php endwhile; ?>
    </table>

<?php elseif(isset($_GET['table_id'])): ?>
    <!-- ================== Halaman Meja ================== -->
    <?php $table_id=$_GET['table_id']; 
    $order=$mysqli->query("SELECT * FROM orders WHERE table_id=$table_id AND status='open'")->fetch_assoc();
    $items=[];
    if($order) $items=$mysqli->query("SELECT oi.*, p.name FROM order_items oi JOIN products p ON oi.product_id=p.id WHERE order_id=".$order['id']);
    ?>
    <h3>🪑 Meja <?= $table_id ?></h3>
    <form method="post">
        <input type="hidden" name="table_id" value="<?= $table_id ?>">
        <select name="product_name" required>
            <option value="">Pilih Produk</option>
            <?php while($p=$products->fetch_assoc()): ?>
                <option value="<?= $p['name'] ?>"><?= $p['name'] ?> (Rp<?= number_format($p['price']) ?>, Stok: <?= $p['stock'] ?>)</option>
            <?php endwhile; ?>
        </select>
        <input type="number" name="qty" placeholder="Qty" required>
        <button type="submit" name="add_order">Tambah</button>
    </form>

    <?php if($order): ?>
    <table>
        <tr><th>Produk</th><th>Qty</th><th>Subtotal</th><th>Aksi</th></tr>
        <?php while($it=$items->fetch_assoc()): ?>
        <tr>
            <td><?= $it['name'] ?></td>
            <td><?= $it['qty'] ?></td>
            <td>Rp<?= number_format($it['subtotal']) ?></td>
            <td><a href="index.php?table_id=<?= $table_id ?>&hapus_item=<?= $it['id'] ?>" onclick="return confirm('Hapus?')">🗑️</a></td>
        </tr>
        <?php endwhile; ?>
        <tr>
            <td colspan="2">Total</td>
            <td colspan="2">Rp<?= number_format($order['total']) ?></td>
        </tr>
    </table>
    <form method="post">
        <input type="hidden" name="table_id" value="<?= $table_id ?>">
        <input type="hidden" name="order_id" value="<?= $order['id'] ?>">
        <input type="number" name="bayar" placeholder="Bayar" required>
        <button type="submit" name="pay_order">Bayar</button>
    </form>
    <?php endif; ?>

<?php else: ?>
    <!-- ================== Pilih Meja ================== -->
    <h3>🪑 Pilih Meja</h3>
    <div class="tables">
        <?php while($t=$tables->fetch_assoc()): ?>
            <a class="<?= $t['status'] ?>" href="index.php?table_id=<?= $t['id'] ?>"><?= $t['table_number'] ?></a>
        <?php endwhile; ?>
    </div>
<?php endif; ?>

</body>
</html>
