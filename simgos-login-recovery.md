# 🔐 Algoritma Enkripsi Password Login SIMGOS

 Dokumentasi teknis mengenai algoritma enkripsi dan verifikasi password yang digunakan oleh aplikasi **SIMGOS (Sistem Informasi Manajemen Pelayanan Elektronik)**.

 > \[!WARNING\]\
>  Dokumentasi ini telah **menyamarkan private key** yang digunakan oleh aplikasi.\
>  Jangan memasukkan credential, private key, atau secret production secara langsung ke repository.

---

 ## 📁 File Sumber

 | File | Keterangan |
| --- | --- |
| `module/Aplikasi/src/Aplikasi/Password.php` | Kelas enkripsi password |
| `module/Aplikasi/src/Aplikasi/V1/Rpc/Authentication/AuthenticationController.php` | Proses login dan verifikasi password |
| `module/Aplikasi/src/Aplikasi/V1/Rest/Pengguna/PenggunaService.php` | Penyimpanan password user baru |

---

 ## 🔑 Kunci Rahasia

 Private key digunakan dalam proses hashing password.

 Untuk keamanan, nilai sebenarnya **tidak ditampilkan dalam dokumentasi ini**:

```
[REDACTED_PRIVATE_KEY]
```

 Dalam contoh kode di bawah, gunakan placeholder:

```
YOUR_PRIVATE_KEY
```

 > **Catatan keamanan:** Jangan mengganti `YOUR_PRIVATE_KEY` dengan private key production di repository publik.

---

 ## 🗄️ Struktur Database

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
PRIVATE_KEY = YOUR_PRIVATE_KEY
```

 ### Implementasi PHP

```
$password = "mypassword123";
$privateKey = "YOUR_PRIVATE_KEY";

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

 Algoritma ini menggunakan dua tahap hashing.

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

 Secara keseluruhan:

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
$privateKey = "YOUR_PRIVATE_KEY";

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

 Hasil akhirnya memiliki format bcrypt:

```
$2y$10$...
```

 > **Catatan:** Hash bcrypt akan berbeda setiap kali password yang sama diproses karena menggunakan salt acak.

---

 # 🔄 Proses Login

 Proses autentikasi pada:

```
AuthenticationController::loginAction
```

 mengikuti urutan:

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

---

 # 🆕 Membuat Password Baru

 Password baru menggunakan **Tipe 3 (`SHA256_PASS_HASH`)**.

 ## PHP CLI

```
php -r '
$password = "PasswordBaru123";
$privateKey = "YOUR_PRIVATE_KEY";

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
$2y$10$...
```

 > Jangan gunakan hash contoh di atas untuk credential production.

---

 ## Script PHP

```
<?php

$password = "PasswordBaru123";
$privateKey = "YOUR_PRIVATE_KEY";

$inner = hash_hmac(
    "sha256",
    $password,
    hash("sha256", $privateKey, false)
);

$hashed = password_hash(
    $inner,
    PASSWORD_BCRYPT
);

echo "Hash: {$hashed}" . PHP_EOL;
```

 Jalankan:

```
php generate_password.php
```

---

 # 🗃️ Update Password di Database

 Setelah mendapatkan hash bcrypt:

```
UPDATE aplikasi.pengguna
SET PASSWORD = '$2y$10$...'
WHERE ID = <id_user>
  AND STATUS = 1;
```

 > \[!CAUTION\]\
>  Pastikan hash yang digunakan adalah hash hasil generate aktual. Jangan menyimpan password plaintext di database atau repository.

---

 # 🧪 Verifikasi Password

 Contoh fungsi verifikasi:

```
<?php

function verifyPassword(string $passDb, string $passInput): bool
{
    $privateKey = "YOUR_PRIVATE_KEY";

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

 | Tipe | Formula | Keterangan |
| --- | --- | --- |
| `MD5_WITH_KEY` | `md5(key + md5(password) + key)` | Legacy |
| `MD5_ONLY` | `md5(password)` | Legacy |
| `SHA256_PASS_HASH` | `bcrypt(HMAC-SHA256(password, SHA256(key)))` | Default |

 ### Urutan Verifikasi

```
PASSWORD INPUT
      │
      ▼
MD5_WITH_KEY
      │
      ├── Cocok ──► LOGIN OK
      │
      ▼
MD5_ONLY
      │
      ├── Cocok ──► LOGIN OK
      │
      ▼
HMAC-SHA256 + BCRYPT
      │
      ├── Cocok ──► LOGIN OK
      │
      ▼
LOGIN DITOLAK
```

---

 # 🔒 Rekomendasi Keamanan

 - Jangan menyimpan private key secara hardcoded di source code.
- Jangan commit private key ke Git.
- Gunakan **environment variable** atau **secret manager** untuk menyimpan secret.
- Jangan menyimpan password plaintext.
- MD5 hanya dipertahankan untuk kompatibilitas password legacy.
- Password baru sebaiknya menggunakan `password_hash()`.
- Pertimbangkan migrasi bertahap password legacy ke algoritma hashing modern seperti **bcrypt** atau **Argon2id**.
- Jangan menampilkan private key pada dokumentasi, issue, log, screenshot, atau commit Git.

 Contoh penggunaan environment variable:

```
$privateKey = getenv('SIMGOS_PRIVATE_KEY');

if (!$privateKey) {
    throw new RuntimeException('SIMGOS_PRIVATE_KEY is not configured.');
}
```

 Dengan pendekatan tersebut, repository hanya berisi:

```
SIMGOS_PRIVATE_KEY
       │
       └──► environment / secret manager
```

 dan **bukan nilai secret sebenarnya**.
