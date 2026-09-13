Dokumentasi Algoritma Enkripsi Password Login SIMGOS

# 🔐 Algoritma Enkripsi Password Login SIMGOS

 Dokumentasi teknis mengenai algoritma enkripsi dan verifikasi password yang digunakan oleh aplikasi **SIMGOS (Sistem Informasi Manajemen Pelayanan Elektronik)**.

 > \[!WARNING\]\
>  Dokumen ini berisi **kunci rahasia hardcoded** yang digunakan oleh aplikasi.\
>  Jangan publikasikan dokumen ini ke repository atau lingkungan yang dapat diakses publik apabila kunci tersebut masih digunakan di lingkungan produksi.

---

 ## 📁 File Sumber

 | File | Keterangan |
| --- | --- |
| `module/Aplikasi/src/Aplikasi/Password.php` | Kelas enkripsi password |
| `module/Aplikasi/src/Aplikasi/V1/Rpc/Authentication/AuthenticationController.php` | Proses login dan verifikasi password |
| `module/Aplikasi/src/Aplikasi/V1/Rest/Pengguna/PenggunaService.php` | Penyimpanan password user baru |

---

 ## 🔑 Kunci Rahasia

 Kunci yang digunakan oleh algoritma:

```
KDFLDMSTHBWWSGCBH
```

 > **Catatan keamanan:** Jika kunci ini masih aktif di production, sebaiknya dipindahkan ke environment variable atau secret manager dan tidak disimpan langsung di source code.

---

 ## 🗄️ Struktur Database

 Password pengguna disimpan pada tabel berikut:

 | Item | Nilai |
| --- | --- |
| **Schema** | `aplikasi` |
| **Table** | `pengguna` |
| **Kolom Password** | `PASSWORD` |
| **Kolom Status** | `STATUS` |
| **Status Aktif** | `1` |

---

 # 🔐 Algoritma Password

 Sistem login mendukung **3 tipe hash password** dan mencoba memverifikasinya secara berurutan.

 | Prioritas | Tipe | Algoritma | Penggunaan |
| --- | --- | --- | --- |
| 1 | `MD5_WITH_KEY` | MD5 + private key | Password lama |
| 2 | `MD5_ONLY` | MD5 | Password lama |
| 3 | `SHA256_PASS_HASH` | HMAC-SHA256 + bcrypt | Password baru / default |

---

 ## 1\. `MD5_WITH_KEY`

 **Prioritas:** Pertama

 Formula:

```
hash = md5(PRIVATE_KEY + md5(password) + PRIVATE_KEY)
```

 Dengan:

```
PRIVATE_KEY = KDFLDMSTHBWWSGCBH
```

 ### Implementasi PHP

```
$password = "mypassword123";
$privateKey = "KDFLDMSTHBWWSGCBH";

$hash = md5(
    $privateKey .
    md5($password) .
    $privateKey
);
```

---

 ## 2\. `MD5_ONLY`

 **Prioritas:** Kedua

 Formula:

```
hash = md5(password)
```

 ### Implementasi PHP

```
$password = "mypassword123";

$hash = md5($password);
```

---

 ## 3\. `SHA256_PASS_HASH`

 **Prioritas:** Ketiga / Default

 Algoritma ini menggunakan dua tahap hashing:

 ### Tahap 1 — HMAC-SHA256

```
key = sha256(PRIVATE_KEY)

inner_hash = HMAC-SHA256(
    password,
    key
)
```

 ### Tahap 2 — Bcrypt

```
hash = bcrypt(inner_hash)
```

 Sehingga keseluruhan proses dapat ditulis sebagai:

```
inner_hash = hash_hmac(
    "sha256",
    password,
    hash("sha256", PRIVATE_KEY)
)

hash = password_hash(
    inner_hash,
    PASSWORD_BCRYPT
)
```

 ### Implementasi PHP

```
$password = "mypassword123";
$privateKey = "KDFLDMSTHBWWSGCBH";

$inner = hash_hmac(
    "sha256",
    $password,
    hash("sha256", $privateKey, false)
);

$hash = password_hash(
    $inner,
    PASSWORD_BCRYPT
);
```

 Hasil akhirnya memiliki format bcrypt, misalnya:

```
$2y$10$...
```

 > **Catatan:** Hash bcrypt akan berbeda setiap kali password yang sama diproses karena menggunakan **salt acak**.

---

 # 🔄 Proses Login

 Proses autentikasi pada:

```
AuthenticationController::loginAction
```

 mengikuti urutan berikut:

```
┌─────────────────────────────┐
│ 1. Terima LOGIN & PASSWORD  │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│ 2. MD5_WITH_KEY             │
│    Cocok? ────────► BERHASIL│
└──────────────┬──────────────┘
               │ Tidak
               ▼
┌─────────────────────────────┐
│ 3. MD5_ONLY                 │
│    Cocok? ────────► BERHASIL│
└──────────────┬──────────────┘
               │ Tidak
               ▼
┌─────────────────────────────┐
│ 4. SHA256_PASS_HASH         │
│    password_verify()        │
│    Cocok? ────────► BERHASIL│
└──────────────┬──────────────┘
               │ Tidak
               ▼
┌─────────────────────────────┐
│ 5. LOGIN DITOLAK            │
└─────────────────────────────┘
```

 ### Detail Proses

 1. Sistem menerima `LOGIN` dan `PASSWORD`.
2. Password dicoba menggunakan algoritma `MD5_WITH_KEY`.
3. Jika hasil hash cocok dengan nilai `PASSWORD` di database, login berhasil.
4. Jika tidak cocok, sistem mencoba `MD5_ONLY`.
5. Jika masih tidak cocok, sistem menggunakan `SHA256_PASS_HASH`.
6. Untuk bcrypt, password diverifikasi menggunakan `password_verify()`.
7. Jika seluruh metode gagal, login ditolak.

---

 # 🆕 Membuat Password Baru

 Password baru sebaiknya menggunakan **Tipe 3 (`SHA256_PASS_HASH`)**.

 ## Opsi 1 — PHP CLI

 Jalankan perintah berikut:

```
php -r '
$password = "PasswordBaru123";
$privateKey = "KDFLDMSTHBWWSGCBH";

$inner = hash_hmac(
    "sha256",
    $password,
    hash("sha256", $privateKey, false)
);

echo password_hash($inner, PASSWORD_BCRYPT) . PHP_EOL;
'
```

 Contoh output:

```
$2y$10$.......................................................
```

 > Jangan menyalin contoh hash di atas sebagai password production. Gunakan hash yang dihasilkan oleh sistem.

---

 ## Opsi 2 — Script PHP

 Buat file:

```
generate_password.php
```

 Isi dengan:

```
<?php

$password = "PasswordBaru123";
$privateKey = "KDFLDMSTHBWWSGCBH";

$inner = hash_hmac(
    "sha256",
    $password,
    hash("sha256", $privateKey, false)
);

$hashed = password_hash(
    $inner,
    PASSWORD_BCRYPT
);

echo "Password : {$password}" . PHP_EOL;
echo "Hash     : {$hashed}" . PHP_EOL;
```

 Kemudian jalankan:

```
php generate_password.php
```

 Output:

```
Password : PasswordBaru123
Hash     : $2y$10$...
```

---

 # 🗃️ Update Password di Database

 Setelah mendapatkan hash bcrypt, password dapat diperbarui di database:

```
UPDATE aplikasi.pengguna
SET PASSWORD = '$2y$10$...ganti_dengan_hash_hasil...'
WHERE ID = <id_user>
  AND STATUS = 1;
```

 ### Verifikasi hasil update

 Pastikan nilai `PASSWORD` pada user yang dituju sudah berubah:

```
SELECT ID, LOGIN, PASSWORD, STATUS
FROM aplikasi.pengguna
WHERE ID = <id_user>
  AND STATUS = 1;
```

 > \[!CAUTION\]\
>  Lakukan update password hanya pada user yang memang dimaksud. Sebaiknya backup atau gunakan transaksi database sebelum melakukan perubahan pada data production.

---

 # 🧪 Verifikasi Password

 Contoh fungsi PHP untuk memverifikasi password dengan ketiga algoritma:

```
<?php

function verifyPassword(string $passDb, string $passInput): bool
{
    $privateKey = "KDFLDMSTHBWWSGCBH";

    // Tipe 1: MD5_WITH_KEY
    if (
        $passDb === md5(
            $privateKey .
            md5($passInput) .
            $privateKey
        )
    ) {
        return true;
    }

    // Tipe 2: MD5_ONLY
    if ($passDb === md5($passInput)) {
        return true;
    }

    // Tipe 3: SHA256_PASS_HASH
    $inner = hash_hmac(
        "sha256",
        $passInput,
        hash("sha256", $privateKey, false)
    );

    if (password_verify($inner, $passDb)) {
        return true;
    }

    return false;
}
```

---

 # 📊 Ringkasan

```
                    PASSWORD INPUT
                          │
                          ▼
              ┌─────────────────────┐
              │ MD5_WITH_KEY         │
              │ Priority: 1          │
              └──────────┬──────────┘
                         │
                    Tidak cocok
                         │
                         ▼
              ┌─────────────────────┐
              │ MD5_ONLY             │
              │ Priority: 2          │
              └──────────┬──────────┘
                         │
                    Tidak cocok
                         │
                         ▼
              ┌─────────────────────┐
              │ HMAC-SHA256          │
              │        +             │
              │ BCRYPT               │
              │ Priority: 3          │
              └──────────┬──────────┘
                         │
                   password_verify()
                         │
                  ┌──────┴──────┐
                  │             │
                Cocok         Gagal
                  │             │
                  ▼             ▼
              LOGIN OK      LOGIN DITOLAK
```

 | Tipe | Formula | Keterangan |
| --- | --- | --- |
| `MD5_WITH_KEY` | `md5(key + md5(password) + key)` | Legacy |
| `MD5_ONLY` | `md5(password)` | Legacy |
| `SHA256_PASS_HASH` | `bcrypt(HMAC-SHA256(password, SHA256(key)))` | Default |

---

 # 📝 Catatan Penting

 - Password baru yang dibuat melalui API menggunakan **Tipe 3 (`SHA256_PASS_HASH`)**.
- Password lama kemungkinan masih menggunakan **Tipe 1** atau **Tipe 2**.
- Sistem melakukan fallback secara berurutan dari Tipe 1 → Tipe 2 → Tipe 3.
- Tipe 3 menggunakan `password_hash()` dengan `PASSWORD_BCRYPT`.
- `password_verify()` digunakan untuk memverifikasi hash bcrypt.
- Bcrypt menggunakan salt acak sehingga hash dari password yang sama dapat berbeda setiap kali dibuat.
- Hardcoded private key sebaiknya **tidak disimpan langsung di source code**.
- MD5 tidak direkomendasikan untuk penyimpanan password baru. Untuk implementasi baru, gunakan password hashing modern seperti bcrypt atau Argon2id.
- Jika memungkinkan, password legacy berbasis MD5 sebaiknya dimigrasikan secara bertahap ke algoritma hashing yang lebih aman setelah pengguna berhasil login.

 > \[!WARNING\]\
>  **Jangan commit private key, password plaintext, atau hash credential production ke repository Git.**\
>  Untuk deployment, gunakan environment variable atau secret manager.

---

 ## 🔒 Rekomendasi Migrasi

 Untuk sistem legacy, pendekatan yang lebih aman adalah:

```
User Login
    │
    ▼
Verifikasi password legacy
    │
    ├── MD5_WITH_KEY ──┐
    │                  │
    ├── MD5_ONLY ──────┤
    │                  ▼
    │             Password Valid
    │                  │
    │                  ▼
    │          Re-hash dengan
    │          password_hash()
    │                  │
    │                  ▼
    │          Simpan hash baru
    │
    └── BCRYPT ───────► Login OK
```

 Dengan pendekatan tersebut, password lama dapat **dimigrasikan secara otomatis saat pengguna berhasil login**, sehingga sistem tidak perlu mempertahankan hash MD5 selamanya.
