-- ============================================================
-- DATABASE SYSTEM POINT OF SALE (POS) KASIR CAFE
-- Engine: MySQL / MariaDB
-- File: schema.sql
-- ============================================================

CREATE DATABASE IF NOT EXISTS pos_cafe 
    CHARACTER SET utf8mb4 
    COLLATE utf8mb4_unicode_ci;

USE pos_cafe;

-- ------------------------------------------------------------
-- 1. PENGGUNA & HAK AKSES (USERS)
-- ------------------------------------------------------------
CREATE TABLE IF NOT EXISTS users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    username VARCHAR(50) NOT NULL UNIQUE,
    password VARCHAR(255) NOT NULL,
    role ENUM('admin', 'cashier', 'barista', 'kitchen') DEFAULT 'cashier',
    phone VARCHAR(20) NULL,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB;

-- ------------------------------------------------------------
-- 2. SHIFT KASIR (CASHIER SHIFTS)
-- ------------------------------------------------------------
CREATE TABLE IF NOT EXISTS cashier_shifts (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    starting_cash DECIMAL(12, 2) NOT NULL DEFAULT 0.00,
    ending_cash DECIMAL(12, 2) NULL,
    expected_cash DECIMAL(12, 2) NULL,
    difference DECIMAL(12, 2) NULL,
    opened_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    closed_at TIMESTAMP NULL,
    notes TEXT NULL,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE RESTRICT
) ENGINE=InnoDB;

-- ------------------------------------------------------------
-- 3. KATEGORI & PRODUK
-- ------------------------------------------------------------
CREATE TABLE IF NOT EXISTS categories (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    description TEXT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB;

CREATE TABLE IF NOT EXISTS products (
    id INT AUTO_INCREMENT PRIMARY KEY,
    category_id INT NOT NULL,
    sku VARCHAR(50) NOT NULL UNIQUE,
    name VARCHAR(150) NOT NULL,
    description TEXT NULL,
    cost_price DECIMAL(12, 2) NOT NULL DEFAULT 0.00, -- Harga Pokok Penjualan (HPP)
    selling_price DECIMAL(12, 2) NOT NULL,
    image VARCHAR(255) NULL,
    is_available BOOLEAN DEFAULT TRUE,
    track_stock BOOLEAN DEFAULT FALSE,
    stock_quantity INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (category_id) REFERENCES categories(id) ON DELETE RESTRICT
) ENGINE=InnoDB;

-- ------------------------------------------------------------
-- 4. MODIFIER / OPSI TAMBAHAN PRODUK (TOPPING / CUSTOMIZATION)
-- ------------------------------------------------------------
CREATE TABLE IF NOT EXISTS product_modifiers (
    id INT AUTO_INCREMENT PRIMARY KEY,
    product_id INT NOT NULL,
    name VARCHAR(100) NOT NULL, -- Contoh: "Extra Espresso Shot", "Oat Milk Swap"
    extra_price DECIMAL(10, 2) DEFAULT 0.00,
    is_available BOOLEAN DEFAULT TRUE,
    FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE
) ENGINE=InnoDB;

-- ------------------------------------------------------------
-- 5. BAHAN BUKU & STOK (INGREDIENTS & INVENTORY)
-- ------------------------------------------------------------
CREATE TABLE IF NOT EXISTS ingredients (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    unit VARCHAR(20) NOT NULL, -- Contoh: 'gram', 'ml', 'pcs'
    stock_quantity DECIMAL(10, 2) DEFAULT 0.00,
    minimum_stock DECIMAL(10, 2) DEFAULT 0.00,
    cost_per_unit DECIMAL(12, 2) DEFAULT 0.00,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB;

-- Resep Produk (Hubungan Produk dengan Bahan Baku)
CREATE TABLE IF NOT EXISTS product_recipes (
    id INT AUTO_INCREMENT PRIMARY KEY,
    product_id INT NOT NULL,
    ingredient_id INT NOT NULL,
    quantity_required DECIMAL(10, 2) NOT NULL,
    FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE,
    FOREIGN KEY (ingredient_id) REFERENCES ingredients(id) ON DELETE RESTRICT
) ENGINE=InnoDB;

-- Mutasi / Riwayat Stok Bahan & Produk
CREATE TABLE IF NOT EXISTS stock_logs (
    id INT AUTO_INCREMENT PRIMARY KEY,
    item_type ENUM('product', 'ingredient') NOT NULL,
    item_id INT NOT NULL,
    type ENUM('in', 'out', 'adjustment', 'waste') NOT NULL,
    quantity DECIMAL(10, 2) NOT NULL,
    notes VARCHAR(255) NULL,
    created_by INT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (created_by) REFERENCES users(id)
) ENGINE=InnoDB;

-- ------------------------------------------------------------
-- 6. MEJA & KELOLA PROMO / DISKON
-- ------------------------------------------------------------
CREATE TABLE IF NOT EXISTS tables (
    id INT AUTO_INCREMENT PRIMARY KEY,
    table_number VARCHAR(20) NOT NULL UNIQUE,
    capacity INT DEFAULT 2,
    status ENUM('available', 'occupied', 'reserved') DEFAULT 'available'
) ENGINE=InnoDB;

CREATE TABLE IF NOT EXISTS payment_methods (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50) NOT NULL, -- Cash, QRIS, Debit BCA, GoPay, dll.
    is_active BOOLEAN DEFAULT TRUE
) ENGINE=InnoDB;

CREATE TABLE IF NOT EXISTS discounts (
    id INT AUTO_INCREMENT PRIMARY KEY,
    code VARCHAR(50) UNIQUE NULL,
    name VARCHAR(100) NOT NULL,
    type ENUM('percentage', 'fixed') NOT NULL DEFAULT 'percentage',
    value DECIMAL(10, 2) NOT NULL, -- Misal 10.00 (%) atau 10000.00 (Rupiah)
    max_discount DECIMAL(12, 2) NULL,
    min_purchase DECIMAL(12, 2) DEFAULT 0.00,
    is_active BOOLEAN DEFAULT TRUE
) ENGINE=InnoDB;

-- ------------------------------------------------------------
-- 7. TRANSAKSI UTAMA & DETAIL PESANAN
-- ------------------------------------------------------------
CREATE TABLE IF NOT EXISTS transactions (
    id INT AUTO_INCREMENT PRIMARY KEY,
    invoice_number VARCHAR(50) NOT NULL UNIQUE,
    shift_id INT NULL,
    user_id INT NOT NULL,
    table_id INT NULL,
    discount_id INT NULL,
    customer_name VARCHAR(100) DEFAULT 'Guest',
    order_type ENUM('dine_in', 'takeaway') DEFAULT 'dine_in',
    subtotal DECIMAL(12, 2) NOT NULL,
    tax_amount DECIMAL(12, 2) DEFAULT 0.00,
    discount_amount DECIMAL(12, 2) DEFAULT 0.00,
    total_amount DECIMAL(12, 2) NOT NULL,
    payment_method_id INT NULL,
    payment_status ENUM('pending', 'paid', 'cancelled', 'refunded') DEFAULT 'pending',
    paid_amount DECIMAL(12, 2) DEFAULT 0.00,
    change_amount DECIMAL(12, 2) DEFAULT 0.00,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (shift_id) REFERENCES cashier_shifts(id) ON DELETE SET NULL,
    FOREIGN KEY (user_id) REFERENCES users(id),
    FOREIGN KEY (table_id) REFERENCES tables(id) SET NULL,
    FOREIGN KEY (discount_id) REFERENCES discounts(id) SET NULL,
    FOREIGN KEY (payment_method_id) REFERENCES payment_methods(id)
) ENGINE=InnoDB;

CREATE TABLE IF NOT EXISTS transaction_items (
    id INT AUTO_INCREMENT PRIMARY KEY,
    transaction_id INT NOT NULL,
    product_id INT NOT NULL,
    quantity INT NOT NULL,
    unit_price DECIMAL(12, 2) NOT NULL, -- Mencatat harga beli saat transaksi terjadi
    cost_price DECIMAL(12, 2) NOT NULL DEFAULT 0.00,
    subtotal DECIMAL(12, 2) NOT NULL,
    notes VARCHAR(255) NULL, -- Misal: "Less ice, extra sweet"
    kitchen_status ENUM('pending', 'preparing', 'ready', 'served') DEFAULT 'pending',
    FOREIGN KEY (transaction_id) REFERENCES transactions(id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES products(id)
) ENGINE=InnoDB;

-- Relasi Modifier yang Dipilih pada Item Transaksi
CREATE TABLE IF NOT EXISTS transaction_item_modifiers (
    id INT AUTO_INCREMENT PRIMARY KEY,
    transaction_item_id INT NOT NULL,
    modifier_id INT NOT NULL,
    price DECIMAL(10, 2) NOT NULL,
    FOREIGN KEY (transaction_item_id) REFERENCES transaction_items(id) ON DELETE CASCADE,
    FOREIGN KEY (modifier_id) REFERENCES product_modifiers(id)
) ENGINE=InnoDB;

-- ------------------------------------------------------------
-- 8. INDEXING UNTUK OPTIMASI QUERY SIKLUS LAPORAN
-- ------------------------------------------------------------
CREATE INDEX idx_transactions_created_at ON transactions(created_at);
CREATE INDEX idx_transactions_payment_status ON transactions(payment_status);
CREATE INDEX idx_products_category ON products(category_id);
CREATE INDEX idx_transaction_items_transaction ON transaction_items(transaction_id);

-- ============================================================
-- DUMMY DATA SEEDER (DATA AWAL UNTUK UJI COBA)
-- ============================================================

-- Data Pengguna (Password default sebelum di-hash)
INSERT INTO users (name, username, password, role) VALUES 
('Administrator', 'admin', '$2y$10$e0MYzXyjpJS7Pd0RVvHwHe11.4e.4/9/XyS2sWp.L/4G9zGgN/7aG', 'admin'),
('Kasir Utama', 'kasir', '$2y$10$e0MYzXyjpJS7Pd0RVvHwHe11.4e.4/9/XyS2sWp.L/4G9zGgN/7aG', 'cashier'),
('Barista Bar', 'barista', '$2y$10$e0MYzXyjpJS7Pd0RVvHwHe11.4e.4/9/XyS2sWp.L/4G9zGgN/7aG', 'barista');

-- Data Kategori
INSERT INTO categories (name, description) VALUES 
('Espresso Based', 'Olahan espresso klasik dan turunan susu'),
('Manual Brew', 'Kopi seduh manual'),
('Non-Coffee', 'Minuman tanpa kandungan kopi'),
('Bakery & Pastry', 'Kudapan pendamping kopi');

-- Data Produk
INSERT INTO products (category_id, sku, name, cost_price, selling_price, track_stock, stock_quantity) VALUES 
(1, 'ESP-001', 'Espresso Single', 4000.00, 18000.00, FALSE, 0),
(1, 'ESP-002', 'Caffe Latte', 8000.00, 28000.00, FALSE, 0),
(1, 'ESP-003', 'Americano', 5000.00, 22000.00, FALSE, 0),
(3, 'NCF-001', 'Matcha Latte', 9000.00, 30000.00, FALSE, 0),
(4, 'PAS-001', 'Butter Croissant', 10000.00, 25000.00, TRUE, 20);

-- Data Modifier Produk
INSERT INTO product_modifiers (product_id, name, extra_price) VALUES 
(2, 'Extra Shot Espresso', 5000.00),
(2, 'Oat Milk Swap', 8000.00),
(4, 'Extra Shot Matcha', 6000.00);

-- Data Metode Pembayaran
INSERT INTO payment_methods (name) VALUES 
('Tunai'), 
('QRIS'), 
('Debit BCA'), 
('Mandiri EDC');

-- Data Meja
INSERT INTO tables (table_number, capacity) VALUES 
('Meja 01', 2), 
('Meja 02', 4), 
('Meja 03', 4), 
('VIP Room', 8);

-- Data Promo / Diskon
INSERT INTO discounts (code, name, type, value, min_purchase) VALUES 
('PROMO10', 'Diskon Grand Opening 10%', 'percentage', 10.00, 50000.00),
('FLAT5K', 'Potongan Rp 5.000', 'fixed', 5000.00, 30000.00);
