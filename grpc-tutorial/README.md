# Module 08: High Level Networking

**Nama:** Andi Hakim Himawan  
**NPM:** 2406495792  
**Mata Kuliah:** Advanced Programming, Tutorial Module 08

---

## Cara Menjalankan

Pastikan sudah menginstall Rust dan `protoc` (Protobuf compiler).

**Terminal 1, jalankan Server:**
```bash
cargo run --bin grpc-server
```

**Terminal 2, jalankan Client:**
```bash
cargo run --bin grpc-client
```

> Untuk mencoba **Chat Service**, setelah client berjalan ketik pesan apa saja di terminal lalu tekan `Enter`. Server akan langsung membalas secara real-time.

---

## Reflection

### 1. Apa perbedaan utama antara Unary, Server Streaming, dan Bi-Directional Streaming? Kapan masing-masing paling cocok digunakan?

Ketiganya adalah pola komunikasi yang berbeda dalam gRPC, dan pemilihan pola yang tepat sangat bergantung pada kebutuhan aplikasinya.

**Unary** adalah pola paling sederhana, yaitu client mengirim satu request, server membalas dengan satu response, lalu selesai. Ini mirip seperti kita mengirim pertanyaan lewat pesan dan menunggu jawaban. Pola ini paling cocok untuk operasi yang memang bersifat satu-satu, seperti submit pembayaran, login, atau mengambil satu data spesifik dari database. Dalam tutorial ini, `PaymentService` menggunakan unary karena setiap pengajuan pembayaran menghasilkan tepat satu hasil: berhasil atau gagal.

**Server Streaming** cocok digunakan ketika client hanya perlu mengirim satu permintaan, tapi server punya banyak data untuk dikirimkan secara bertahap. Bayangkan kamu minta laporan transaksi selama setahun, daripada server menunggu semua data selesai dikumpulkan baru dikirim sekaligus (yang lambat dan boros memori), server bisa langsung mengirim data per bagian begitu tersedia. Pola ini ideal untuk riwayat transaksi, feed berita, pembaruan harga saham, atau pengiriman file besar. Dalam tutorial ini, `TransactionService` mengirim 30 data transaksi secara bertahap dengan jeda tiap 10 data untuk mensimulasikan perilaku ini.

**Bi-Directional Streaming** adalah pola paling fleksibel, di mana baik client maupun server bisa mengirim pesan kapan saja tanpa perlu menunggu giliran. Koneksi tetap terbuka selama percakapan berlangsung. Ini sangat cocok untuk aplikasi chat, kolaborasi real-time seperti Google Docs, atau sistem monitoring yang butuh interaksi dua arah secara terus-menerus. `ChatService` dalam tutorial ini mengimplementasikan pola ini, di mana user bisa mengetik pesan kapan saja dan server langsung merespons.

---

### 2. Apa saja pertimbangan keamanan yang perlu diperhatikan saat mengimplementasikan gRPC service di Rust?

Implementasi gRPC dalam tutorial ini berjalan di atas koneksi HTTP biasa tanpa enkripsi, yang tidak aman untuk lingkungan produksi. Ada beberapa hal yang perlu diperhatikan:

**Enkripsi data (TLS):** Semua data yang dikirim antara client dan server harus dienkripsi menggunakan TLS. Tonic mendukung konfigurasi `ServerTlsConfig` dan `ClientTlsConfig` untuk keperluan ini. Tanpa TLS, siapa pun yang ada di jaringan yang sama bisa membaca data yang dikirimkan.

**Autentikasi:** Perlu dipastikan bahwa hanya client yang berwenang yang bisa memanggil service. Ini bisa dilakukan menggunakan JWT token atau API key yang dikirim melalui metadata request, lalu divalidasi lewat interceptor di server sebelum handler dipanggil.

**Otorisasi:** Setelah autentikasi berhasil, server tetap perlu memverifikasi apakah user tersebut punya hak akses untuk operasi yang diminta. Misalnya, user hanya boleh melihat transaksinya sendiri, bukan milik user lain.

**Validasi input:** Semua field dalam Protobuf message yang masuk harus divalidasi, misalnya memastikan `amount` tidak negatif dan `user_id` tidak kosong, untuk mencegah data tidak valid masuk ke sistem.

---

### 3. Apa saja tantangan yang mungkin muncul saat menangani bi-directional streaming di Rust gRPC?

Bi-directional streaming adalah fitur yang powerful, tapi kompleksitasnya cukup tinggi dibanding unary.

Tantangan terbesar adalah **manajemen koneksi asinkron**. Karena client dan server bisa mengirim pesan kapan saja secara bersamaan, kita harus menggunakan `tokio::spawn` untuk menjalankan task secara paralel. Jika tidak dikelola dengan benar, task bisa saling menunggu (deadlock) atau ada resource yang tidak dibersihkan saat koneksi putus.

**Ukuran buffer channel** juga krusial. Dalam tutorial ini, `mpsc::channel(10)` artinya maksimal 10 pesan bisa antri di buffer. Jika server menghasilkan pesan lebih cepat dari kemampuan client menerima, buffer penuh dan pengiriman akan tertunda. Ukuran buffer harus disesuaikan dengan kebutuhan traffic aktual.

**Penanganan error dan pemutusan koneksi** juga tidak sepele. Ketika salah satu sisi terputus tiba-tiba, sisi lainnya harus mendeteksi ini dan berhenti dengan bersih tanpa menyebabkan panic atau resource leak. Penggunaan `unwrap_or_else(|_| None)` dalam loop pesan membantu menangani ini secara graceful.

---

### 4. Apa kelebihan dan kekurangan menggunakan `tokio_stream::wrappers::ReceiverStream` untuk streaming response?

`ReceiverStream` adalah jembatan antara channel async Tokio dengan interface Stream yang dibutuhkan oleh Tonic/gRPC.

**Kelebihannya** adalah ia membuat kode menjadi jauh lebih bersih dan mudah dipahami. Kita cukup membuat `mpsc::channel`, spawn task yang mengisi channel tersebut, lalu bungkus receiver-nya dengan `ReceiverStream` dan kembalikan sebagai response. Logika bisnis (pengisian data) terpisah bersih dari logika pengiriman gRPC. Selain itu, channel Tokio secara bawaan sudah mendukung backpressure, sehingga jika consumer lambat, producer akan ikut melambat secara otomatis.

**Kekurangannya** adalah ia hanya mendukung satu consumer (single consumer channel). Jika kita butuh broadcast ke banyak client sekaligus, perlu mekanisme tambahan seperti `tokio::sync::broadcast`. Selain itu, jika task yang mengisi channel mengalami panic, receiver akan tertutup dan stream berakhir tiba-tiba tanpa pesan error yang informatif ke client.

---

### 5. Bagaimana kode Rust gRPC bisa distruktur agar lebih modular dan mudah dirawat?

Dalam tutorial ini, semua logika server ditulis dalam satu file `grpc_server.rs` yang cukup panjang. Untuk proyek yang lebih besar, ada beberapa pendekatan yang lebih baik.

Pisahkan setiap service ke modul tersendiri, misalnya `src/services/payment.rs`, `src/services/transaction.rs`, dan `src/services/chat.rs`. Masing-masing file hanya berisi satu tanggung jawab sehingga lebih mudah dibaca dan ditest secara independen.

Pindahkan deklarasi modul proto (`pub mod services { tonic::include_proto!("services"); }`) ke `lib.rs` agar bisa diimport oleh server maupun client dari satu sumber, menghindari duplikasi kode.

Gunakan trait untuk memisahkan logika bisnis dari handler gRPC. Misalnya, buat trait `PaymentProcessor` yang mendefinisikan operasi pembayaran, lalu inject implementasinya ke dalam struct `MyPaymentService`. Ini membuat unit testing jauh lebih mudah karena kita bisa menggunakan mock implementation saat testing.

---

### 6. Langkah tambahan apa yang diperlukan untuk menangani logika pembayaran yang lebih kompleks di `MyPaymentService`?

Implementasi saat ini hanya mengembalikan `success: true` secara langsung tanpa logika apa pun, yang hanya cocok untuk keperluan demo.

Untuk implementasi nyata, setidaknya dibutuhkan koneksi ke database untuk menyimpan record transaksi dan mengecek saldo user, validasi apakah saldo mencukupi sebelum memproses pembayaran, integrasi dengan payment gateway eksternal seperti Midtrans atau Stripe melalui HTTP client async (`reqwest`), mekanisme idempotency key untuk mencegah pembayaran ganda jika client mengirim ulang request yang sama, serta penanganan error yang informatif menggunakan `Status::failed_precondition` untuk saldo kurang atau `Status::not_found` untuk user yang tidak ditemukan, bukan sekadar boolean.

---

### 7. Apa dampak adopsi gRPC terhadap arsitektur sistem terdistribusi?

gRPC mendorong pendekatan **contract-first development**, yaitu semua tim harus menyepakati definisi service di file `.proto` terlebih dahulu sebelum mulai coding. Ini terdengar birokratis, tapi justru mencegah miskomunikasi antara tim yang mengembangkan service yang berbeda.

Salah satu dampak terbesar adalah **dukungan multilingual**. Satu file `.proto` bisa menghasilkan client dan server code untuk Java, Rust, Go, Python, dan banyak bahasa lain secara otomatis. Ini sangat memudahkan tim yang bekerja dengan stack teknologi yang berbeda-beda untuk tetap bisa berkomunikasi satu sama lain tanpa perlu menulis serialization code secara manual.

Namun gRPC juga membawa kompleksitas baru: file `.proto` harus dikelola sebagai shared artifact (biasanya di repository tersendiri), ada pipeline code generation yang harus diintegrasikan ke CI/CD, dan perubahan schema harus dikelola dengan hati-hati agar tidak mematahkan client yang sudah ada.

---

### 8. Apa kelebihan dan kekurangan HTTP/2 dibanding HTTP/1.1 untuk REST API?

HTTP/2 adalah fondasi dari gRPC dan ia membawa sejumlah peningkatan signifikan dibanding HTTP/1.1.

**Kelebihannya:** HTTP/2 mendukung **multiplexing**, yaitu beberapa request bisa dikirim sekaligus melalui satu koneksi TCP tanpa harus menunggu response sebelumnya selesai. Di HTTP/1.1, setiap request harus menunggu antrian (head-of-line blocking). Selain itu, HTTP/2 menggunakan **header compression (HPACK)** yang mengurangi ukuran header secara signifikan, serta **binary framing** yang lebih efisien untuk diparse oleh mesin.

**Kekurangannya:** Format binary membuat HTTP/2 jauh lebih sulit untuk di-debug secara manual. Di HTTP/1.1, kita bisa langsung membaca request dan response menggunakan `curl` atau browser DevTools. Di HTTP/2 apalagi gRPC, kita butuh tool khusus seperti `grpcurl`. Selain itu, HTTP/2 tetap berjalan di atas TCP sehingga masih rentan terhadap TCP head-of-line blocking di level transport, dan masalah ini baru benar-benar selesai di HTTP/3 yang menggunakan QUIC.

Untuk WebSocket + HTTP/1.1 sebagai perbandingan: WebSocket memang mendukung komunikasi dua arah, tapi ia tidak punya standar schema, tidak ada code generation otomatis, dan tidak ada built-in load balancing seperti yang dimiliki gRPC.

---

### 9. Bagaimana model request-response REST berbeda dengan kemampuan bidirectional streaming gRPC untuk komunikasi real-time?

REST dirancang di atas HTTP yang bersifat stateless, di mana setiap request berdiri sendiri dan koneksi biasanya ditutup setelah response diterima. Untuk mendapatkan data terbaru, client harus terus-menerus mengirim request baru (polling) yang boros bandwidth dan menambah latensi. Alternatifnya adalah Server-Sent Events (SSE), tapi itu hanya satu arah dari server ke client saja.

gRPC bi-directional streaming mempertahankan satu koneksi yang tetap terbuka, dan kedua sisi bisa mengirim pesan kapan saja tanpa overhead pembukaan koneksi baru. Ini membuat latensi menjadi sangat rendah karena pesan bisa terkirim dalam hitungan milidetik setelah event terjadi di server, tanpa harus menunggu polling interval.

Perbedaan ini terasa sangat nyata di aplikasi seperti chat, live dashboard, atau game multiplayer, di mana setiap milidetik keterlambatan terasa. Untuk kasus-kasus tersebut, gRPC streaming jauh lebih efisien dan responsif dibanding pendekatan REST apapun.

---

### 10. Apa implikasi pendekatan berbasis schema (Protocol Buffers) di gRPC dibanding JSON yang lebih fleksibel di REST?

Protocol Buffers mengharuskan kita mendefinisikan setiap message dan field secara eksplisit di file `.proto` sebelum bisa digunakan. Ini terasa lebih "kaku" dibanding JSON, tapi kekakuan ini justru menjadi kekuatan.

Karena schema-nya ketat dan di-generate otomatis, **perubahan yang merusak kompatibilitas (breaking changes) langsung terdeteksi saat kompilasi**, bukan saat runtime di production. Jika sebuah field dihapus atau tipenya diubah, semua client yang masih menggunakan field itu akan gagal build sehingga masalah terdeteksi lebih awal.

Dari sisi performa, **binary encoding Protobuf secara konsisten jauh lebih kecil** dari JSON yang setara dan jauh lebih cepat untuk di-serialize maupun di-deserialize. Ini signifikan di sistem dengan traffic tinggi.

Di sisi lain, JSON jauh lebih **ramah manusia dan fleksibel**. Kita bisa menambahkan field baru tanpa koordinasi karena client lama cukup mengabaikan field yang tidak dikenalinya. Ini membuat JSON lebih cocok untuk public API yang konsumennya beragam dan tidak bisa dikontrol, sedangkan Protobuf lebih cocok untuk komunikasi internal antar service di mana semua pihak bisa dikoordinasi.