# 📚 Perpustakaan API

REST API **Laravel 12** (PHP 8.3) dengan autentikasi token **Laravel Sanctum**, CRUD data buku, dan dokumentasi interaktif **OpenAPI 3 (Swagger UI)** lewat **L5-Swagger**.

![Laravel](https://img.shields.io/badge/Laravel-12-FF2D20?logo=laravel&logoColor=white)
![PHP](https://img.shields.io/badge/PHP-8.3-777BB4?logo=php&logoColor=white)
![Sanctum](https://img.shields.io/badge/Auth-Sanctum-orange)
![OpenAPI](https://img.shields.io/badge/OpenAPI-3.0-6BA539?logo=openapiinitiative&logoColor=white)
![Tests](https://img.shields.io/badge/tests-PHPUnit-366488)

---

## Daftar Isi

- [Fitur](#fitur)
- [Tech Stack](#tech-stack)
- [Daftar Endpoint](#daftar-endpoint)
- [Struktur Proyek](#struktur-proyek)
- [Instalasi](#instalasi)
- [Konfigurasi](#konfigurasi)
- [Menjalankan Aplikasi](#menjalankan-aplikasi)
- [Dokumentasi API (Swagger)](#dokumentasi-api-swagger)
- [Contoh Penggunaan](#contoh-penggunaan)
- [Menjalankan Test](#menjalankan-test)
- [Troubleshooting](#troubleshooting)

---

## Fitur

- Registrasi, login, profil (`/me`), dan logout dengan token **Sanctum** (Bearer token).
- CRUD data buku: paginasi, pencarian berdasarkan judul/penulis, dan validasi lewat **Form Request**.
- Format JSON seragam memakai **API Resource**.
- Semua error di `/api/*` selalu dibalas **JSON** (termasuk `401 Unauthenticated`), walaupun klien lupa mengirim header `Accept: application/json`.
- Rate limiting pada endpoint login (`throttle:6,1`, maksimal 6 percobaan per menit).
- Dokumentasi **OpenAPI 3** memakai PHP Attributes `#[OA\...]`, tampil di Swagger UI dan bisa diimpor ke Postman.
- Test otomatis **PHPUnit** (9 test fitur) memakai SQLite in-memory.

## Tech Stack

| Komponen | Keterangan |
|---|---|
| Laravel 12 + PHP 8.3 | Framework dan bahasa |
| Laravel Sanctum | Token API (dipasang oleh `php artisan install:api`) |
| `darkaonline/l5-swagger` | Swagger UI + generator OpenAPI (berbasis `zircote/swagger-php`) |
| SQLite / MySQL | SQLite adalah default Laravel 12, bisa diganti MySQL |
| PHPUnit | Test otomatis |

## Daftar Endpoint

| Method | Endpoint | Fungsi | Login? |
|---|---|---|---|
| `POST` | `/api/register` | Daftar akun, dapat token | Tidak |
| `POST` | `/api/login` | Login, dapat token | Tidak |
| `GET` | `/api/me` | Profil user yang login | Ya |
| `POST` | `/api/logout` | Hapus token aktif | Ya |
| `GET` | `/api/books` | Daftar buku (paginasi + pencarian) | Ya |
| `POST` | `/api/books` | Tambah buku | Ya |
| `GET` | `/api/books/{book}` | Detail buku | Ya |
| `PUT/PATCH` | `/api/books/{book}` | Ubah buku | Ya |
| `DELETE` | `/api/books/{book}` | Hapus buku | Ya |

Parameter query untuk `GET /api/books`:

| Parameter | Keterangan |
|---|---|
| `search` | Cari berdasarkan judul atau penulis |
| `per_page` | Jumlah data per halaman (1-100, default 10) |
| `page` | Nomor halaman (default 1) |

## Struktur Proyek

File yang ditulis atau diubah di atas proyek Laravel baru:

```
app/
  Http/Controllers/Controller.php          # info global OpenAPI
  Http/Controllers/Api/AuthController.php
  Http/Controllers/Api/BookController.php
  Http/Requests/Auth/RegisterRequest.php
  Http/Requests/Auth/LoginRequest.php
  Http/Requests/StoreBookRequest.php
  Http/Requests/UpdateBookRequest.php
  Http/Resources/BookResource.php
  Http/Resources/UserResource.php
  Models/Book.php
  Models/User.php
bootstrap/app.php                          # error API selalu JSON
config/l5-swagger.php                      # hasil vendor:publish
config/sanctum.php                         # masa berlaku token (opsional)
database/migrations/2025_11_07_000000_create_books_table.php
database/factories/BookFactory.php
database/seeders/DatabaseSeeder.php
routes/api.php
tests/Feature/AuthApiTest.php
tests/Feature/BookApiTest.php
```

---

## Instalasi

**Prasyarat:** PHP 8.3 (ekstensi `pdo_sqlite` atau `pdo_mysql`, `mbstring`, `openssl`, `fileinfo`), Composer 2, dan terminal. Di Windows, Laragon atau XAMPP dengan PHP 8.3 sudah cukup.

### Menjalankan repositori ini

```bash
git clone https://github.com/faisallzz/perpustakaan-api.git
cd perpustakaan-api

composer install
cp .env.example .env
php artisan key:generate
```

Atur `.env` (lihat [Konfigurasi](#konfigurasi)), lalu migrasi dan isi data contoh:

```bash
php artisan migrate:fresh --seed
```

> ⚠️ `migrate:fresh` menghapus semua tabel lalu membuatnya ulang. Jangan dijalankan di server produksi.

Seeder membuat 1 akun demo dan 20 buku contoh:

| Email | Password |
|---|---|
| `admin@example.com` | `password` |

---

## Konfigurasi

### `.env`

Laravel 12 memakai SQLite secara default. Contoh konfigurasi:

```env
APP_URL=http://localhost:8000

# Default Laravel 12: SQLite. Untuk MySQL ganti jadi:
# DB_CONNECTION=mysql
# DB_HOST=127.0.0.1
# DB_PORT=3306
# DB_DATABASE=perpustakaan_api
# DB_USERNAME=root
# DB_PASSWORD=

L5_SWAGGER_CONST_HOST=http://localhost:8000
L5_SWAGGER_GENERATE_ALWAYS=true
```

- `L5_SWAGGER_CONST_HOST` menjadi URL server di Swagger UI.
- `L5_SWAGGER_GENERATE_ALWAYS=true` membuat dokumentasi diperbarui otomatis setiap halaman dibuka (cocok untuk development).

Setelah mengubah `.env`, jalankan:

```bash
php artisan config:clear
```

### `bootstrap/app.php` (error API selalu JSON)

Konfigurasi ini memastikan request ke `/api/*` yang tidak terautentikasi dibalas **401 JSON**, bukan di-redirect ke route `login` (yang tidak ada di proyek API-only dan memicu error **500** `Route [login] not defined`).

```php
<?php

use Illuminate\Auth\AuthenticationException;
use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Exceptions;
use Illuminate\Foundation\Configuration\Middleware;
use Illuminate\Http\Request;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        api: __DIR__.'/../routes/api.php', // ditambahkan oleh install:api
        commands: __DIR__.'/../routes/console.php',
        health: '/up',
    )
    ->withMiddleware(function (Middleware $middleware): void {
        // Proyek API-only: jangan redirect ke route 'login'
        $middleware->redirectGuestsTo(fn (Request $request) => null);
    })
    ->withExceptions(function (Exceptions $exceptions): void {
        // Semua error di /api/* selalu dibalas JSON,
        // walaupun klien lupa mengirim header Accept: application/json.
        $exceptions->shouldRenderJsonWhen(function (Request $request, Throwable $e) {
            return $request->is('api/*') || $request->expectsJson();
        });

        // Respons 401 yang konsisten
        $exceptions->render(function (AuthenticationException $e, Request $request) {
            if ($request->is('api/*') || $request->expectsJson()) {
                return response()->json(['message' => 'Unauthenticated.'], 401);
            }
        });
    })->create();
```

### `routes/api.php`

Semua route di file ini otomatis berawalan `/api`.

```php
<?php

use App\Http\Controllers\Api\AuthController;
use App\Http\Controllers\Api\BookController;
use Illuminate\Support\Facades\Route;

// Endpoint publik
Route::post('/register', [AuthController::class, 'register']);
Route::post('/login', [AuthController::class, 'login'])->middleware('throttle:6,1');

// Endpoint yang wajib login (Bearer token Sanctum)
Route::middleware('auth:sanctum')->group(function () {
    Route::get('/me', [AuthController::class, 'me']);
    Route::post('/logout', [AuthController::class, 'logout']);
    Route::apiResource('books', BookController::class);
});
```

Cek route (harus ada 9 route, PUT dan PATCH tampil sebagai satu baris `PUT|PATCH`):

```bash
php artisan route:list --path=api
```

### Skema database: tabel `books`

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | bigint | Primary key |
| `title` | string | Wajib |
| `author` | string | Wajib |
| `publisher` | string | Opsional |
| `isbn` | string(20) | Wajib, **unik** |
| `published_year` | unsigned smallint | Wajib, 1000 sampai tahun berjalan |
| `stock` | unsigned int | Default 0, minimal 0 |
| `description` | text | Opsional |
| `created_at`, `updated_at` | timestamp | |

---

## Menjalankan Aplikasi

```bash
php artisan serve
```

Server berjalan di `http://localhost:8000`.

Jika memakai MySQL di Laragon, pastikan MySQL sudah hidup (**Start All**) dan database sudah dibuat sebelum menjalankan migrasi.

---

## Dokumentasi API (Swagger)

Generate dokumentasi (tidak perlu diulang jika `L5_SWAGGER_GENERATE_ALWAYS=true`):

```bash
php artisan l5-swagger:generate
```

| URL | Isi |
|---|---|
| `http://localhost:8000/api/documentation` | Swagger UI interaktif |
| `http://localhost:8000/docs` | JSON mentah OpenAPI (bisa diimpor ke Postman) |

Hasil generate: 2 tag (**Auth**, **Books**), 10 operasi (9 route ditambah PATCH yang didokumentasikan terpisah), dan 10 schema.

### Cara memakai Swagger UI

1. Buka **Auth → POST /api/login**, klik **Try it out**, isi `admin@example.com` / `password`, lalu **Execute**.
2. Salin nilai `token` dari respons, misalnya `1|Xz9aBcD...`.
3. Klik tombol **Authorize** (ikon gembok), tempel token **tanpa** kata `Bearer`, klik **Authorize**, lalu **Close**.
4. Coba **GET /api/me** dan **GET /api/books** (20 buku dari seeder dalam 2 halaman).
5. Coba **POST /api/books**, lalu kirim lagi dengan ISBN yang sama untuk melihat respons `422`.
6. Coba **PUT**, **PATCH**, **GET**, dan **DELETE** pada `/api/books/{book}`.
7. Coba **POST /api/logout**, lalu ulangi **GET /api/books**. Hasilnya harus **401** `{"message": "Unauthenticated."}` karena token sudah dihapus.

---

## Contoh Penggunaan

### Respons login

```json
{
  "message": "Login berhasil",
  "token": "1|Xz9aBcDeFgHiJkLmNoPq...",
  "token_type": "Bearer",
  "user": {
    "id": 1,
    "name": "Admin Perpustakaan",
    "email": "admin@example.com",
    "created_at": "2026-11-07T13:00:00.000000Z"
  }
}
```

### Login gagal (422)

```json
{
  "message": "Email atau password salah.",
  "errors": { "email": ["Email atau password salah."] }
}
```

### Respons `GET /api/books?per_page=2`

```json
{
  "data": [
    {
      "id": 20,
      "title": "Laskar Pelangi",
      "author": "Andrea Hirata",
      "publisher": "Bentang Pustaka",
      "isbn": "9789793062792",
      "published_year": 2005,
      "stock": 10,
      "description": "...",
      "created_at": "2026-11-07T13:00:00.000000Z",
      "updated_at": "2026-11-07T13:00:00.000000Z"
    }
  ],
  "links": {
    "first": "http://localhost:8000/api/books?per_page=2&page=1",
    "last": "http://localhost:8000/api/books?per_page=2&page=10",
    "prev": null,
    "next": "http://localhost:8000/api/books?per_page=2&page=2"
  },
  "meta": { "current_page": 1, "from": 1, "last_page": 10, "per_page": 2, "to": 2, "total": 20 }
}
```

### Uji lewat curl

Jalankan di terminal bash (Linux, macOS, Git Bash, WSL, atau terminal Laragon). **Bukan** CMD/PowerShell biasa. Server (`php artisan serve`) harus sudah berjalan.

```bash
BASE=http://localhost:8000/api
H_JSON=(-H "Accept: application/json" -H "Content-Type: application/json")

# 1. Login dan simpan token
TOKEN=$(curl -s -X POST $BASE/login "${H_JSON[@]}" \
  -d '{"email":"admin@example.com","password":"password"}' \
  | php -r 'echo json_decode(stream_get_contents(STDIN))->token;')
echo "Token: $TOKEN"
AUTH=(-H "Authorization: Bearer $TOKEN")

# 2. Profil
curl -s $BASE/me "${H_JSON[@]}" "${AUTH[@]}"

# 3. Daftar + pencarian
curl -s "$BASE/books?search=a&per_page=5" "${H_JSON[@]}" "${AUTH[@]}"

# 4. Tambah buku
curl -s -X POST $BASE/books "${H_JSON[@]}" "${AUTH[@]}" \
  -d '{"title":"Laskar Pelangi","author":"Andrea Hirata","publisher":"Bentang Pustaka","isbn":"9789793062792","published_year":2005,"stock":10}'

# 5. Detail, ubah sebagian, hapus (ganti 21 dengan ID dari langkah 4)
curl -s $BASE/books/21 "${H_JSON[@]}" "${AUTH[@]}"
curl -s -X PATCH $BASE/books/21 "${H_JSON[@]}" "${AUTH[@]}" -d '{"stock":25}'
curl -s -X DELETE $BASE/books/21 "${H_JSON[@]}" "${AUTH[@]}"

# 6. Logout, lalu akses lagi harus 401
curl -s -X POST $BASE/logout "${H_JSON[@]}" "${AUTH[@]}"
curl -s -o /dev/null -w "%{http_code}\n" $BASE/books "${H_JSON[@]}" "${AUTH[@]}"
```

> ⚠️ Skrip ini memakai database yang aktif di `.env` (bukan database test), jadi data buku akan benar-benar ditambah, diubah, dan dihapus. Gunakan database development, bukan data penting. Langkah terakhir harus mencetak `401`.

---

## Menjalankan Test

```bash
php artisan test
```

Ada dua file test:

| File | Jumlah test | Cakupan |
|---|---|---|
| `tests/Feature/AuthApiTest.php` | 4 | Register, login, password salah, `/me`, logout |
| `tests/Feature/BookApiTest.php` | 5 | Akses tanpa login, list + paginasi + search, create + validasi, show + update PUT/PATCH + 404, delete |

Hasil yang diharapkan: **11 passed** (termasuk `ExampleTest` bawaan Laravel).

### Database untuk test

Test memakai **SQLite in-memory** dari `phpunit.xml` (bawaan Laravel 12), sehingga database development (misalnya MySQL) tidak tersentuh. Pastikan dua baris ini **tidak dikomentari**:

```xml
<env name="DB_CONNECTION" value="sqlite"/>
<env name="DB_DATABASE" value=":memory:"/>
```

Syarat tambahan:

- Ekstensi `pdo_sqlite` dan `sqlite3` aktif. Cek dengan `php -m` (di Windows: `php -m | findstr sqlite`).
- Jalankan `php artisan config:clear` bila ada cache config lama, karena config yang ter-cache dapat mengabaikan `phpunit.xml`.

### Catatan: `forgetGuards()` pada test logout

Di dalam satu test, Laravel menyimpan user yang sudah terautentikasi. Karena itu `AuthApiTest` memanggil `$this->app['auth']->forgetGuards();` setelah logout, supaya request berikutnya benar-benar memeriksa token lagi dan menghasilkan `401`.

---

## Troubleshooting

| Gejala | Penyebab | Solusi |
|---|---|---|
| `500` + `Route [login] not defined` saat token tidak valid/sudah logout | Middleware `Authenticate` mencoba redirect ke route `login`, yang tidak ada di proyek API | Pasang `redirectGuestsTo(fn () => null)` dan handler `AuthenticationException` di `bootstrap/app.php`, lalu `php artisan optimize:clear` |
| `Required @OA\Info() not found` | Atribut `#[OA\Info]` tidak terbaca | Pastikan ada di `app/Http/Controllers/Controller.php` dan `paths.annotations` berisi `base_path('app')`, lalu `composer dump-autoload` |
| `Undefined constant L5_SWAGGER_CONST_HOST` | Konstanta belum ada di config | Cek bagian `constants` di `config/l5-swagger.php` dan `.env`, lalu `php artisan config:clear` |
| `Call to undefined method createToken()` | Model `User` belum memakai `HasApiTokens` | Tambahkan `use HasApiTokens` di `app/Models/User.php` |
| `403 This action is unauthorized` | `authorize()` di Form Request masih `return false` | Ganti menjadi `return true;` di semua Form Request |
| `no such table: personal_access_tokens` | Migration Sanctum belum dijalankan | `php artisan migrate` (atau `migrate:fresh --seed` di development) |
| Swagger UI selalu `401` | Token belum di-Authorize, atau endpoint tidak punya `security` | Klik **Authorize** dan tempel token tanpa kata `Bearer`. Pastikan atribut punya `security: [['sanctum' => []]]` |
| Respons HTML/redirect, bukan JSON | Header `Accept` tidak dikirim | Kirim `Accept: application/json` dan pasang `shouldRenderJsonWhen` di `bootstrap/app.php` |
| `429 Too Many Attempts` saat login | Lebih dari 6 percobaan login per menit | Tunggu 1 menit, atau ubah angka di `throttle:6,1` |
| Perubahan atribut tidak muncul di Swagger | Dokumentasi lama masih tersimpan | `php artisan l5-swagger:generate` atau set `L5_SWAGGER_GENERATE_ALWAYS=true` |
| Schema hilang atau `$ref` tidak ditemukan | Nama di `ref` salah ketik, atau kelas schema tidak berada di `app/` | Samakan `ref: '#/components/schemas/Nama'` dengan `schema: 'Nama'` |
| Tampilan Swagger UI kosong / aset 404 | Aset belum dipublish atau `APP_URL` salah | Samakan `APP_URL` dengan URL server, lalu `vendor:publish` ulang |

---