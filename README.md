**Tutorial 10 Pemrograman Lanjut (Advanced Programming) 2024/2025 Genap**
* Nama    : Muhammad Almerazka Yocendra
* NPM     : 2306241745
* Kelas   : Pemrograman Lanjut - A

## 🍊 Understanding How It Works
![image](https://github.com/user-attachments/assets/e39b4785-d485-48ec-9855-bc8ecd49d27c)

#### Setelah 2 detik
![image](https://github.com/user-attachments/assets/7fc140f0-fc04-4a49-ad4a-10928ccb97e0)

Program Rust `async` bekerja dengan cara yang unik dibandingkan eksekusi sinkron biasa. Ketika kode `spawner.spawn(async { ... })` dipanggil, yang terjadi bukanlah eksekusi langsung dari blok `async` tersebut, melainkan hanya pembuatan sebuah **tugas** yang dimasukkan ke dalam antrian `executor`. Proses `spawn` ini sangat cepat, hanya memakan waktu beberapa mikrodetik untuk membungkus `async` block menjadi **Task** dan mengirimkannya ke _channel_. Setelah `spawn` selesai, _main_ `thread` langsung melanjutkan eksekusi ke baris berikutnya tanpa menunggu `async` task tersebut berjalan.

Inilah sebabnya mengapa `println!("hey hey")` yang berada di main thread dapat tercetak lebih dulu. Baris ini dieksekusi secara sinkron dan langsung oleh _main_ `thread`, tidak ada yang menghalangi atau menundanya. Sementara itu, `async` task yang berisi `println!("howdy!")` masih **tertidur** di dalam antrian, menunggu giliran untuk diproses. `Async` di Rust memiliki sifat **lazy evaluation**, artinya `async` _block_ tidak akan berjalan sampai ada `executor` yang secara aktif mem-_poll_ nya.

Eksekusi  `async` _task_ baru dimulai ketika `executor.run()` dipanggil. Pada titik inilah `executor` mulai mengambil _task_ dari antrian dan mem-_poll_ mereka satu per satu. Task pertama yang di-_poll_ akan mencetak `"howdy!"`, kemudian menunggu **TimerFuture** selama 2 detik dalam status **Pending**, lalu setelah _timer_ selesai dan _task_ di-_wake_ kembali, barulah `"done!"` tercetak. Jadi urutan `"hey hey"` → `"howdy!"` → `"done!"` terjadi karena _main_ `thread` menyelesaikan tugasnya lebih dulu sebelum `executor` mulai memproses `async` _tasks_ yang telah di-_spawn_ sebelumnya.

## 🍋 Multiple Spawn and removing drop

### 🥑 Multiple Spawn
![image](https://github.com/user-attachments/assets/8733c072-fdfa-4172-9e4b-8b0c01c2c33f)

Berdasarkan gambar diatas yang menunjukkan _multiple spawn_, terlihat pola _cooperative multitasking_ yang jelas, semua pesan **"howdy"** tercetak berurutan sesuai _spawn_, kemudian semua _task_ memulai `TimerFuture` dan mengembalikan `Poll::Pending` hampir bersamaan. Hal ini terjadi karena _executor_ mem-_poll_ setiap _task_ satu per satu dengan cepat dalam satu _cycle_, sehingga semua _task_ dapat mencetak **"howdy"** tanpa interupsi sebelum ada yang sempat menunggu _timer_.

Setelah mencapai `await TimerFuture`, semua _task_ menunggu _concurrent_ selama 2 detik dengan _timer_ yang berjalan di background _thread_ terpisah. Meskipun _timer_ dimulai hampir bersamaan, mereka dapat selesai dalam urutan berbeda karena faktor _timing_ sistem operasi dan _thread scheduling_ yang tidak deterministik. Ketika _timer_ selesai, _background thread_ memanggil `waker.wake()` yang mengirim _task_ kembali ke `executor` _queue_. Fenomena urutan **"done"** yang tidak sesuai dengan urutan kode (seperti `done2!` muncul sebelum `done!`) mendemonstrasikan _concurrent execution_.  Perbedaan waktu penyelesaian hanya beberapa milidetik tetapi cukup mengubah urutan output, mendemonstrasikan _illusion of parallelism_ dari _async programming_ di mana multiple task "berjalan bersamaan" meskipun dijalankan dalam _single-threaded_ `executor` melalui _cooperative scheduling_.

### 🫐 Removing Drop
![image](https://github.com/user-attachments/assets/2e6bcdd8-b80a-4d3d-81d0-7785d9331faf)

Berdasarkan gambar kita dapat melihat perbedaan perilaku yang signifikan antara menggunakan `drop(spawner)` dan tidak menggunakannya. Ketika `drop(spawner)` tidak dipanggil, program akan hang setelah mencetak semua pesan `"done"` karena _executor_ masih menunggu _task_ baru yang mungkin datang. Hal ini terjadi karena metode `run()` pada `executor` menggunakan `loop while let Ok(task) = self.ready_queue.recv()` yang akan terus memblokir dan menunggu _task_ baru selama _channel_ masih terbuka. Selama `spawner` masih ada, _channel_ tetap terbuka dan `executor` akan terus menunggu tanpa batas waktu, menyebabkan program tidak pernah berhenti secara natural. Peran `drop(spawner)` sangat krusial sebagai sinyal _shutdown_ untuk sistem `async`. Tanpa drop ini, `executor` tidak memiliki cara untuk mengetahui bahwa tidak akan ada _task_ baru yang datang, sehingga ia menunggu tanpa batas waktu. **Dropping** _spawner_ menutup sisi _sender_ dari _channel_, yang memungkinkan `executor` mendeteksi kapan semua pekerjaan telah selesai dan keluar.

## 🍎 Conclusion
- `Effect of Spawning` menciptakan task baru dalam sistem `async` tanpa memblokir eksekusi _main thread_
- `Spawner` berperan sebagai _interface_ untuk membuat dan mendistribusikan _task_ ke sistem `executor`.
- `Executor` adalah _runtime_ _engine_ yang menjalankan task-task secara _actual_.
- `Drop spawner` berfungsi sebagai _shutdown signal_ untuk seluruh sistem _async_.
-  Korelasi keseluruhan membentuk sistem _async_ yang lengkap : `spawner` sebagai _producer_, `executor` sebagai _consumer_, `channel` sebagai _communication medium_, dan `drop` sebagai _termination signal_
