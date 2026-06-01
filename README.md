# Dekomposisi_P11_23343082_RendiAigoBrandon
Tugas Pertemuan 11 Mobile Programming Lanjutan tentang dekomposisi sistem integrasi API dan skenario login banking.

# 1. Pemecahan Sistem Integrasi API Menjadi Beberapa Lapisan

Sistem integrasi API pada aplikasi mobile tidak sebaiknya diletakkan langsung di dalam widget. Jika API call dicampur dengan UI, kode akan sulit dibaca, sulit diuji, dan sulit dikembangkan. Karena itu, sistem perlu dipecah menjadi beberapa lapisan agar setiap bagian memiliki tanggung jawab yang jelas.

| No | Lapisan                               | Tanggung Jawab                                                                                                                               | Package Flutter yang Digunakan                           |
| -- | ------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------- |
| 1  | Presentation Layer atau UI Layer      | Menampilkan tampilan aplikasi, menerima input pengguna, menampilkan loading, error, empty state, dan data berhasil dimuat.                   | `flutter/material.dart`                                  |
| 2  | State Management Layer                | Mengatur perubahan state seperti initial, loading, success, error, empty, dan refreshing. Lapisan ini menjadi penghubung antara UI dan data. | `provider`, `riverpod`, `bloc`, `flutter_bloc`           |
| 3  | Repository Layer                      | Menjadi pusat pengambilan data. Repository menentukan apakah data diambil dari API, cache lokal, atau kombinasi keduanya.                    | Dart abstract class, `get_it` untuk dependency injection |
| 4  | Remote Data Source Layer              | Mengambil data dari REST API, mengirim request HTTP, menerima response JSON, dan menangani endpoint API.                                     | `dio`, `retrofit`, `http`                                |
| 5  | Local Data Source Layer               | Menyimpan dan membaca data lokal untuk kebutuhan caching, offline mode, session, dan data sementara.                                         | `hive`, `shared_preferences`, `sqflite`                  |
| 6  | Service Layer atau HTTP Client Layer  | Mengatur konfigurasi HTTP client seperti base URL, timeout, header default, interceptor, logging, retry, dan token refresh.                  | `dio`, `pretty_dio_logger`                               |
| 7  | Security dan Token Storage Layer      | Menyimpan access token, refresh token, PIN, atau data sensitif lain secara aman. Lapisan ini juga mendukung proses auto login dan logout.    | `flutter_secure_storage`, `local_auth`                   |
| 8  | Model atau Entity Layer               | Merepresentasikan struktur data aplikasi. Lapisan ini mengubah JSON response menjadi object Dart yang mudah digunakan.                       | `json_serializable`, `freezed`, `equatable`              |
| 9  | Connectivity dan Error Handling Layer | Mengecek status jaringan, menangani timeout, server error, unauthorized, forbidden, dan fallback ke cache.                                   | `connectivity_plus`, `dio`                               |

## Ringkasan Alur Lapisan

Alur kerja integrasi API berjalan dari UI ke state management, lalu ke repository. Repository akan memilih sumber data yang tepat, yaitu remote data source jika jaringan tersedia atau local data source jika aplikasi perlu mengambil data dari cache. Semua request HTTP dikontrol oleh service layer agar autentikasi, timeout, retry, dan error handling berjalan secara terpusat.

---

# 2. Skenario Login ke Aplikasi Banking

Skenario login banking harus dirancang dengan aman karena aplikasi banking mengelola data sensitif seperti identitas pengguna, saldo, mutasi rekening, dan transaksi. Berikut adalah urutan teknis dari pengguna menekan tombol login sampai dashboard berhasil tampil.

| No | Langkah Teknis                                                                                                              | Komponen Flutter                                     | API Call atau Penyimpanan Lokal                                                                    |
| -- | --------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| 1  | Pengguna membuka halaman login aplikasi banking.                                                                            | `LoginPage`, `TextFormField`, `Form`                 | Belum ada API call                                                                                 |
| 2  | Pengguna memasukkan username atau nomor handphone dan password atau PIN.                                                    | `TextEditingController`, `TextFormField`             | Input masih berada di memory aplikasi                                                              |
| 3  | Pengguna menekan tombol Login.                                                                                              | `ElevatedButton`, `onPressed()`                      | Trigger proses validasi                                                                            |
| 4  | Aplikasi melakukan validasi form di sisi client. Contohnya field tidak boleh kosong dan format nomor handphone harus benar. | `FormState.validate()`                               | Belum ada API call                                                                                 |
| 5  | State berubah menjadi loading agar tombol login tidak bisa ditekan berkali-kali.                                            | `LoginProvider`, `LoginBloc`, atau `StateNotifier`   | State: `Loading`                                                                                   |
| 6  | Aplikasi mengirim request login ke server melalui repository.                                                               | `AuthRepository`                                     | Memanggil `AuthRemoteDataSource.login()`                                                           |
| 7  | Remote data source mengirim HTTP POST ke endpoint login.                                                                    | `DioService`, `Dio`                                  | `POST /auth/login` dengan body berisi username dan password                                        |
| 8  | Service layer menambahkan konfigurasi request seperti base URL, content type, accept header, timeout, dan HTTPS.            | `Dio(BaseOptions)`                                   | Header: `Content-Type: application/json`                                                           |
| 9  | Server memverifikasi kredensial pengguna di database banking.                                                               | Backend server                                       | Jika valid, server mengirim access token, refresh token, data user, dan status MFA jika diperlukan |
| 10 | Jika banking memakai OTP atau MFA, aplikasi menampilkan halaman verifikasi OTP.                                             | `OtpPage`, `PinCodeTextField`                        | `POST /auth/verify-otp`                                                                            |
| 11 | Setelah OTP valid, aplikasi menerima access token dan refresh token.                                                        | `AuthRepository`                                     | Response berisi token dan data user                                                                |
| 12 | Aplikasi menyimpan token secara aman. Token tidak boleh disimpan di `SharedPreferences` karena tidak terenkripsi.           | `TokenStorageService`                                | Simpan ke `flutter_secure_storage`                                                                 |
| 13 | Aplikasi menyimpan data profil dasar yang tidak terlalu sensitif untuk kebutuhan UI awal.                                   | `UserLocalDataSource`                                | Simpan nama user atau metadata ringan ke `Hive` atau secure storage                                |
| 14 | Aplikasi mengambil data awal dashboard seperti saldo rekening, daftar rekening, dan notifikasi penting.                     | `DashboardRepository`                                | `GET /accounts`, `GET /balance`, `GET /notifications`                                              |
| 15 | Setiap request dashboard membawa access token di header Authorization.                                                      | `AuthInterceptor`                                    | Header: `Authorization: Bearer <access_token>`                                                     |
| 16 | Jika access token kedaluwarsa, interceptor menangkap error 401 dan menjalankan refresh token otomatis.                      | `Dio Interceptor`                                    | `POST /auth/refresh`                                                                               |
| 17 | Jika refresh token berhasil, request dashboard yang gagal diulang dengan token baru.                                        | `AuthInterceptor`, `Completer`, `Retry Logic`        | Retry request asli                                                                                 |
| 18 | Jika data dashboard berhasil diterima, state berubah menjadi success.                                                       | `DashboardProvider` atau `DashboardBloc`             | State: `Success`                                                                                   |
| 19 | Aplikasi menyimpan sebagian data dashboard ke cache untuk fallback jika jaringan terganggu.                                 | `DashboardLocalDataSource`                           | Cache ke `Hive` atau `sqflite`                                                                     |
| 20 | Dashboard berhasil ditampilkan kepada pengguna.                                                                             | `DashboardPage`, `ListView`, `Card`, `BalanceWidget` | Data tampil dari response API atau cache yang valid                                                |

## Penanganan Error pada Login Banking

| Skenario Error                | Tindakan Aplikasi                                                      |
| ----------------------------- | ---------------------------------------------------------------------- |
| Username atau password kosong | Tampilkan pesan validasi di form                                       |
| Password salah                | Tampilkan pesan login gagal tanpa membocorkan detail sistem            |
| Tidak ada koneksi internet    | Tampilkan pesan jaringan dan jangan mengirim request                   |
| Server error 500              | Tampilkan pesan server sedang bermasalah dan sediakan tombol coba lagi |
| Timeout                       | Batalkan request dan tampilkan pesan koneksi lambat                    |
| Token expired 401             | Jalankan refresh token otomatis                                        |
| Refresh token expired         | Hapus token dan arahkan pengguna ke halaman login                      |
| OTP salah                     | Tampilkan pesan OTP tidak valid                                        |
| Akses ditolak 403             | Tampilkan pesan akun tidak memiliki izin akses                         |
| Response JSON tidak sesuai    | Log error dan tampilkan pesan umum kepada pengguna                     |

---

# 3. Daftar Sub-Topik Integrasi API yang Perlu Dipelajari

## A. Tingkat Dasar

Sub-topik dasar adalah materi yang harus dikuasai lebih awal sebelum membangun integrasi API yang kompleks.

| No | Sub-Topik                | Alasan Dipelajari                                           |
| -- | ------------------------ | ----------------------------------------------------------- |
| 1  | Konsep REST API          | Memahami cara client dan server bertukar data               |
| 2  | HTTP Method              | Memahami fungsi `GET`, `POST`, `PUT`, `PATCH`, dan `DELETE` |
| 3  | HTTP Status Code         | Membedakan status seperti 200, 400, 401, 403, 404, dan 500  |
| 4  | JSON Parsing             | Mengubah response JSON menjadi object Dart                  |
| 5  | Model Class              | Membuat struktur data yang rapi dan mudah digunakan         |
| 6  | Basic API Call           | Mengirim request sederhana dari Flutter                     |
| 7  | Form Validation          | Memastikan input pengguna valid sebelum dikirim ke API      |
| 8  | Loading dan Error UI     | Memberi feedback saat request berjalan atau gagal           |
| 9  | Package `http` dan `dio` | Mengenal package dasar untuk komunikasi HTTP                |
| 10 | Environment Base URL     | Memisahkan URL development, staging, dan production         |

## B. Tingkat Menengah

Sub-topik menengah dibutuhkan saat aplikasi mulai memiliki banyak endpoint, banyak fitur, dan butuh struktur kode yang rapi.

| No | Sub-Topik                      | Alasan Dipelajari                                                               |
| -- | ------------------------------ | ------------------------------------------------------------------------------- |
| 1  | Repository Pattern             | Memisahkan UI dari sumber data                                                  |
| 2  | Service Layer                  | Mengatur konfigurasi HTTP client secara terpusat                                |
| 3  | Remote Data Source             | Mengelola endpoint API dengan lebih rapi                                        |
| 4  | Local Data Source              | Mengelola cache dan penyimpanan lokal                                           |
| 5  | Token-Based Authentication     | Menggunakan access token dan refresh token                                      |
| 6  | Dio Interceptor                | Menambahkan header token dan menangani error secara otomatis                    |
| 7  | Secure Token Storage           | Menyimpan token dengan aman memakai `flutter_secure_storage`                    |
| 8  | Async State Management         | Mengelola state initial, loading, success, error, empty, dan refreshing         |
| 9  | Caching Strategy               | Memilih strategi seperti cache first, network first, dan stale while revalidate |
| 10 | Pagination dan Infinite Scroll | Memuat data list secara bertahap agar aplikasi ringan                           |

## C. Tingkat Lanjutan

Sub-topik lanjutan dibutuhkan untuk aplikasi production-ready yang memiliki banyak pengguna dan risiko kegagalan tinggi.

| No | Sub-Topik                                | Alasan Dipelajari                                                   |
| -- | ---------------------------------------- | ------------------------------------------------------------------- |
| 1  | Refresh Token Otomatis                   | Menjaga sesi pengguna tanpa login ulang                             |
| 2  | Retry dengan Exponential Backoff         | Menghindari retry berlebihan saat server bermasalah                 |
| 3  | Offline-First Architecture               | Membuat aplikasi tetap dapat digunakan saat jaringan buruk          |
| 4  | Cache Invalidation                       | Menentukan kapan cache dianggap tidak valid                         |
| 5  | Rate Limiting Handling                   | Menangani batas request dari server                                 |
| 6  | Request Queue                            | Mengatur request tertunda saat token refresh atau jaringan terputus |
| 7  | Error Logging dan Crash Reporting        | Membantu debugging di lingkungan produksi                           |
| 8  | API Security Best Practice               | Melindungi data sensitif dari kebocoran                             |
| 9  | Certificate Pinning                      | Mengurangi risiko serangan man-in-the-middle                        |
| 10 | Observability dan Monitoring             | Memantau performa API, latency, dan error rate                      |
| 11 | Clean Architecture                       | Membuat struktur aplikasi lebih modular dan mudah diuji             |
| 12 | Unit Test dan Integration Test untuk API | Memastikan integrasi API berjalan stabil setelah perubahan kode     |

---

# Kesimpulan

Integrasi API pada aplikasi mobile tidak cukup hanya membuat request berhasil dan data tampil di layar. Aplikasi production-ready harus memiliki struktur lapisan yang jelas, autentikasi yang aman, state management yang lengkap, caching, error handling, retry mechanism, dan penyimpanan lokal yang tepat.

Pada skenario login banking, keamanan menjadi prioritas utama. Token harus disimpan secara aman, request harus memakai HTTPS, error harus ditangani dengan jelas, dan refresh token harus berjalan otomatis agar pengalaman pengguna tetap baik.

Dengan memahami pemecahan lapisan, alur login teknis, dan daftar sub-topik yang perlu dipelajari, developer dapat membangun aplikasi mobile yang lebih stabil, aman, dan siap digunakan di lingkungan produksi.
