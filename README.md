**Tutorial 10 Pemrograman Lanjut (Advanced Programming) 2024/2025 Genap**
* Nama    : Muhammad Almerazka Yocendra
* NPM     : 2306241745
* Kelas   : Pemrograman Lanjut - A

## Understanding How It Works
![image](https://github.com/user-attachments/assets/e39b4785-d485-48ec-9855-bc8ecd49d27c)

Setelah 2 detik
![image](https://github.com/user-attachments/assets/7fc140f0-fc04-4a49-ad4a-10928ccb97e0)

Program Rust `async` bekerja dengan cara yang unik dibandingkan eksekusi sinkron biasa. Ketika kode `spawner.spawn(async { ... })` dipanggil, yang terjadi bukanlah eksekusi langsung dari blok `async` tersebut, melainkan hanya pembuatan sebuah **tugas** yang dimasukkan ke dalam antrian `executor`. Proses `spawn` ini sangat cepat, hanya memakan waktu beberapa mikrodetik untuk membungkus `async` block menjadi **Task** dan mengirimkannya ke _channel_. Setelah `spawn` selesai, _main_ `thread` langsung melanjutkan eksekusi ke baris berikutnya tanpa menunggu `async` task tersebut berjalan.
Inilah sebabnya mengapa `println!("hey hey")` yang berada di main thread dapat tercetak lebih dulu. Baris ini dieksekusi secara sinkron dan langsung oleh _main_ `thread`, tidak ada yang menghalangi atau menundanya. Sementara itu, `async` task yang berisi `println!("howdy!")` masih **tertidur** di dalam antrian, menunggu giliran untuk diproses. `Async` di Rust memiliki sifat **lazy evaluation**, artinya `async` _block_ tidak akan berjalan sampai ada `executor` yang secara aktif mem-_poll_ nya.

Eksekusi  `async` _task_ baru dimulai ketika `executor.run()` dipanggil. Pada titik inilah `executor` mulai mengambil _task_ dari antrian dan mem-_poll_ mereka satu per satu. Task pertama yang di-_poll_ akan mencetak `"howdy!"`, kemudian menunggu **TimerFuture** selama 2 detik dalam status **Pending**, lalu setelah _timer_ selesai dan _task_ di-_wake_ kembali, barulah `"done!"` tercetak. Jadi urutan `"hey hey"` → `"howdy!"` → `"done!"` terjadi karena _main_ `thread` menyelesaikan tugasnya lebih dulu sebelum `executor` mulai memproses `async` _tasks_ yang telah di-_spawn_ sebelumnya.

