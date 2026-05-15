# Dokumen Kebutuhan - Platform Marketplace JASAne

## Pendahuluan

JASAne adalah platform marketplace mobile yang menghubungkan pengguna dengan penyedia jasa rumah tangga lokal (mitra). Platform ini memungkinkan pemesanan instan layanan melalui sistem penjadwalan berbasis slot dengan pemrosesan pembayaran terintegrasi, perlindungan escrow, dan komunikasi real-time. Dibangun sebagai aplikasi mobile Flutter (Android & iOS) menggunakan arsitektur 1 Aplikasi, 2 Mode dengan routing berbasis peran, JASAne memanfaatkan Supabase untuk layanan backend dan Xendit untuk pemrosesan pembayaran.

Platform ini menghilangkan alur kerja persetujuan tradisional dengan menerapkan pemesanan instan di mana slot secara otomatis dikunci setelah konfirmasi pembayaran. Hal ini menciptakan pengalaman pengguna yang mulus sambil mempertahankan akuntabilitas melalui sistem penalti komprehensif dan perlindungan escrow 24 jam untuk penyelesaian sengketa.

## Glosarium

- **JASAne_Platform**: Sistem marketplace mobile lengkap termasuk aplikasi mobile dan layanan backend
- **User (Pengguna)**: Pelanggan yang memesan layanan dari mitra
- **Mitra**: Penyedia jasa terverifikasi yang menawarkan layanan rumah tangga
- **Admin**: Administrator platform yang mengelola verifikasi dan sengketa
- **Booking (Pemesanan)**: Janji layanan yang dikonfirmasi dengan pembayaran
- **Slot**: Blok waktu dalam jadwal mitra yang tersedia untuk pemesanan
- **Escrow**: Periode penahanan pembayaran 24 jam untuk perlindungan sengketa
- **Service_Protection_Points (Poin Perlindungan Layanan)**: Sistem pelacakan penalti untuk akuntabilitas mitra (Poin SP)
- **Instant_Booking (Pemesanan Instan)**: Penguncian slot otomatis setelah pembayaran tanpa alur persetujuan
- **Supabase**: Platform backend yang menyediakan database PostgreSQL, Edge Functions, dan layanan Realtime
- **Xendit**: Gateway pembayaran pihak ketiga yang mendukung QRIS, transfer bank, e-wallet, dan pembayaran kartu
- **Edge_Function**: Fungsi serverless yang berjalan di Supabase untuk eksekusi logika bisnis
- **RLS**: Kebijakan Row Level Security di PostgreSQL untuk kontrol akses data
- **FCM**: Firebase Cloud Messaging untuk notifikasi push
- **Clean_Architecture**: Struktur proyek Flutter yang memisahkan lapisan Presentation, Domain, dan Data
- **Riverpod**: Library manajemen state untuk Flutter
- **No_Show**: Ketika pengguna tidak hadir pada waktu terjadwal meskipun pemesanan sudah dikonfirmasi
- **Cancellation_Policy (Kebijakan Pembatalan)**: Aturan pengembalian dana berbasis waktu (>H-2: 100%, H-1: 50%, H-0: 0%)


## Kebutuhan

### Kebutuhan 1: Autentikasi dan Otorisasi Pengguna

**Cerita Pengguna:** Sebagai pengguna, saya ingin melakukan autentikasi menggunakan email atau Google OAuth, sehingga saya dapat mengakses platform dengan aman menggunakan metode login pilihan saya.

#### Kriteria Penerimaan

1. KETIKA pengguna memberikan email dan password yang valid, Sistem_Autentikasi HARUS mengautentikasi pengguna dan membuat sesi
2. KETIKA pengguna memilih login Google OAuth, Sistem_Autentikasi HARUS mengarahkan ke autentikasi Google dan membuat sesi setelah otorisasi berhasil
3. KETIKA autentikasi gagal karena kredensial tidak valid, Sistem_Autentikasi HARUS mengembalikan pesan error yang deskriptif
4. KETIKA pengguna berhasil terautentikasi, Sistem_Autentikasi HARUS mengambil profil pengguna dari tabel users
5. Sistem_Autentikasi HARUS menyimpan token sesi secara aman di penyimpanan perangkat
6. KETIKA pengguna logout, Sistem_Autentikasi HARUS membatalkan token sesi dan menghapus penyimpanan lokal
7. KETIKA token sesi kedaluwarsa, Sistem_Autentikasi HARUS meminta pengguna untuk autentikasi ulang
8. UNTUK SEMUA operasi autentikasi, Sistem_Autentikasi HARUS menerapkan pembatasan laju untuk mencegah serangan brute force

### Kebutuhan 2: Routing Aplikasi Berbasis Peran

**Cerita Pengguna:** Sebagai pengguna platform, saya ingin aplikasi mengarahkan saya ke antarmuka yang sesuai berdasarkan peran saya, sehingga saya melihat fitur yang relevan untuk tipe akun saya.

#### Kriteria Penerimaan

1. KETIKA pengguna dengan peran "user" login, Sistem_Routing HARUS menavigasi ke layar beranda
2. KETIKA pengguna dengan peran "mitra" login, Sistem_Routing HARUS menavigasi ke dashboard mitra
3. KETIKA pengguna dengan peran "admin" login, Sistem_Routing HARUS menavigasi ke dashboard admin
4. KETIKA peran pengguna berubah selama sesi aktif, Sistem_Routing HARUS mengarahkan ke antarmuka yang sesuai dalam 5 detik
5. JIKA pengguna mencoba mengakses rute yang tidak diizinkan untuk perannya, MAKA Sistem_Routing HARUS mengarahkan ke dashboard default mereka
6. Sistem_Routing HARUS menyimpan rute terakhir yang dikunjungi per peran untuk pemulihan sesi

### Kebutuhan 3: Peningkatan Peran Pengguna ke Mitra

**Cerita Pengguna:** Sebagai pengguna, saya ingin meningkatkan akun saya menjadi mitra, sehingga saya dapat menawarkan layanan di platform.

#### Kriteria Penerimaan

1. KETIKA pengguna dengan peran "user" mengirimkan permintaan pendaftaran mitra dengan foto KTP, Sistem_Manajemen_Pengguna HARUS membuat profil mitra dengan is_verified diatur ke false
2. KETIKA profil mitra dibuat, Sistem_Manajemen_Pengguna HARUS memperbarui peran pengguna menjadi "mitra"
3. KETIKA pengguna meningkat ke mitra, Sistem_Manajemen_Pengguna HARUS menginisialisasi Service_Protection_Points ke 100
4. Sistem_Manajemen_Pengguna HARUS menyimpan foto KTP di bucket Supabase Storage privat dengan enkripsi
5. KETIKA profil mitra dibuat, Sistem_Manajemen_Pengguna HARUS mengatur is_available ke false sampai verifikasi selesai
6. JIKA pengguna sudah memiliki profil mitra, MAKA Sistem_Manajemen_Pengguna HARUS mengembalikan error yang mencegah profil duplikat


### Kebutuhan 4: Verifikasi Mitra oleh Admin

**Cerita Pengguna:** Sebagai admin, saya ingin memverifikasi dokumen KTP mitra, sehingga hanya penyedia jasa yang sah yang dapat menawarkan layanan di platform.

#### Kriteria Penerimaan

1. KETIKA admin menyetujui profil mitra, Sistem_Verifikasi HARUS mengatur is_verified ke true dan mencatat timestamp verification_at
2. KETIKA mitra diverifikasi, Sistem_Verifikasi HARUS mencatat ID pengguna admin di field verified_by
3. KETIKA mitra diverifikasi, Sistem_Verifikasi HARUS mengatur is_available ke true untuk mengizinkan penerimaan pemesanan
4. KETIKA admin menolak profil mitra, Sistem_Verifikasi HARUS mengatur is_verified ke false dan mencatat alasan penolakan
5. Sistem_Verifikasi HARUS membuat entri audit log untuk setiap keputusan verifikasi
6. KETIKA mitra diverifikasi, Sistem_Notifikasi HARUS mengirim notifikasi push ke pengguna mitra
7. DIMANA profil mitra memiliki is_verified diatur ke false, Sistem_Pemesanan HARUS mencegah pengguna memesan layanan dari mitra tersebut

### Kebutuhan 5: Manajemen Katalog Layanan Mitra

**Cerita Pengguna:** Sebagai mitra, saya ingin membuat dan mengelola penawaran layanan saya dengan harga, sehingga pengguna dapat memesan layanan saya.

#### Kriteria Penerimaan

1. KETIKA mitra terverifikasi membuat layanan, Sistem_Manajemen_Layanan HARUS menyimpan layanan dengan kategori, nama, deskripsi, harga, dan satuan_harga
2. Sistem_Manajemen_Layanan HARUS memvalidasi bahwa harga lebih besar dari nol
3. KETIKA mitra menentukan duration_minutes, Sistem_Manajemen_Layanan HARUS menggunakan nilai ini untuk perhitungan slot
4. KETIKA mitra memperbarui layanan, Sistem_Manajemen_Layanan HARUS TIDAK mempengaruhi pemesanan yang ada menggunakan harga lama
5. KETIKA mitra menonaktifkan layanan, Sistem_Manajemen_Layanan HARUS mengatur is_active ke false dan menyembunyikannya dari pencarian pengguna
6. Sistem_Manajemen_Layanan HARUS mengizinkan nilai price_unit: "per_job", "per_hour", "per_unit", atau "estimate"
7. DIMANA layanan memiliki price_unit diatur ke "estimate", Sistem_Pemesanan HARUS memerlukan konfirmasi harga manual sebelum pembayaran

### Kebutuhan 6: Konfigurasi Jadwal Ketersediaan Mitra

**Cerita Pengguna:** Sebagai mitra, saya ingin mengkonfigurasi jadwal ketersediaan mingguan saya dengan slot waktu, sehingga pengguna hanya dapat memesan selama jam tersedia saya.

#### Kriteria Penerimaan

1. KETIKA mitra membuat jadwal untuk hari dalam seminggu, Sistem_Manajemen_Jadwal HARUS menyimpan open_time, close_time, dan slot_duration_minutes
2. Sistem_Manajemen_Jadwal HARUS memvalidasi bahwa close_time setelah open_time
3. Sistem_Manajemen_Jadwal HARUS memvalidasi bahwa slot_duration_minutes antara 15 dan 480 menit
4. KETIKA mitra mengatur max_slots_per_day, Sistem_Manajemen_Jadwal HARUS menerapkan batas ini selama pemesanan
5. Sistem_Manajemen_Jadwal HARUS mencegah jadwal duplikat untuk day_of_week yang sama per mitra
6. KETIKA mitra menonaktifkan jadwal, Sistem_Manajemen_Jadwal HARUS mengatur is_active ke false dan mencegah pemesanan baru untuk hari tersebut
7. Sistem_Manajemen_Jadwal HARUS menghitung slot tersedia dengan membagi (close_time - open_time) dengan slot_duration_minutes


### Kebutuhan 7: Manajemen Tanggal Diblokir Mitra

**Cerita Pengguna:** Sebagai mitra, saya ingin memblokir tanggal tertentu ketika saya tidak tersedia, sehingga pengguna tidak dapat memesan saya pada hari libur atau cuti pribadi.

#### Kriteria Penerimaan

1. KETIKA mitra memblokir tanggal, Sistem_Manajemen_Jadwal HARUS menyimpan blocked_date dan alasan opsional
2. Sistem_Manajemen_Jadwal HARUS mencegah tanggal diblokir duplikat untuk mitra yang sama
3. KETIKA pengguna mencoba memesan tanggal yang diblokir, Sistem_Pemesanan HARUS mengecualikan tanggal tersebut dari slot tersedia
4. KETIKA mitra menghapus tanggal yang diblokir, Sistem_Manajemen_Jadwal HARUS segera membuat tanggal tersebut tersedia untuk pemesanan
5. Sistem_Manajemen_Jadwal HARUS mengizinkan pemblokiran tanggal hingga 365 hari ke depan

### Kebutuhan 8: Pembuatan Pemesanan Instan dengan Penguncian Slot

**Cerita Pengguna:** Sebagai pengguna, saya ingin memesan layanan secara instan tanpa menunggu persetujuan, sehingga saya dapat mengamankan slot waktu pilihan saya segera setelah pembayaran.

#### Kriteria Penerimaan

1. KETIKA pengguna memilih layanan dan slot waktu, Sistem_Pemesanan HARUS menghitung subtotal, platform_fee, dan total_amount
2. KETIKA pengguna mengkonfirmasi pemesanan, Sistem_Pemesanan HARUS menghasilkan booking_code unik dalam format "JSN-YYYYMMDD-XXXX"
3. KETIKA pemesanan dibuat, Sistem_Pemesanan HARUS mengatur status ke "terjadwal" segera
4. Sistem_Pemesanan HARUS memvalidasi bahwa slot waktu yang dipilih tersedia sebelum membuat pemesanan
5. Sistem_Pemesanan HARUS memvalidasi bahwa scheduled_at minimal 2 jam di masa depan
6. KETIKA pemesanan dibuat, Sistem_Pemesanan HARUS membuat record booking_items dengan snapshot layanan
7. Sistem_Pemesanan HARUS menghitung platform_fee sebagai 10% dari subtotal secara default
8. KETIKA pemesanan dibuat, Sistem_Pemesanan HARUS mengunci slot waktu untuk mencegah pemesanan ganda
9. JIKA slot yang dipilih sudah dipesan, MAKA Sistem_Pemesanan HARUS mengembalikan error dengan slot alternatif yang tersedia
10. UNTUK SEMUA operasi pembuatan pemesanan, Sistem_Pemesanan HARUS dieksekusi dalam Edge_Function untuk menerapkan aturan bisnis di sisi server

### Kebutuhan 9: Pemrosesan Pembayaran via Xendit

**Cerita Pengguna:** Sebagai pengguna, saya ingin membayar pemesanan menggunakan berbagai metode pembayaran, sehingga saya dapat menyelesaikan transaksi dengan nyaman.

#### Kriteria Penerimaan

1. KETIKA pemesanan dibuat, Sistem_Pembayaran HARUS membuat invoice Xendit dengan jumlah, mata uang, dan waktu kedaluwarsa
2. Sistem_Pembayaran HARUS mendukung metode pembayaran: QRIS, transfer bank, e-wallet, dan kartu
3. KETIKA invoice Xendit dibuat, Sistem_Pembayaran HARUS menyimpan xendit_invoice_id dan invoice_url
4. Sistem_Pembayaran HARUS mengatur kedaluwarsa invoice ke 24 jam dari pembuatan
5. KETIKA Xendit mengirim webhook pembayaran dengan status "PAID", Sistem_Pembayaran HARUS memperbarui status pembayaran ke "paid" dan mencatat timestamp paid_at
6. KETIKA pembayaran dikonfirmasi, Sistem_Pembayaran HARUS memperbarui status pemesanan ke "terjadwal"
7. JIKA pembayaran kedaluwarsa tanpa penyelesaian, MAKA Sistem_Pembayaran HARUS memperbarui status ke "expired" dan melepaskan slot yang dikunci
8. Sistem_Pembayaran HARUS menyimpan payload webhook Xendit lengkap di field xendit_raw untuk keperluan audit
9. Sistem_Pembayaran HARUS memvalidasi tanda tangan webhook untuk mencegah konfirmasi pembayaran palsu
10. UNTUK SEMUA pemrosesan webhook, Sistem_Pembayaran HARUS menerapkan idempotency untuk menangani pengiriman webhook duplikat


### Kebutuhan 10: Sistem Penahanan Pembayaran Escrow

**Cerita Pengguna:** Sebagai pengguna, saya ingin pembayaran saya ditahan dalam escrow selama 24 jam setelah penyelesaian layanan, sehingga saya punya waktu untuk mengajukan sengketa jika layanan tidak memuaskan.

#### Kriteria Penerimaan

1. KETIKA status pemesanan berubah ke "selesai", Sistem_Escrow HARUS membuat record mitra_earnings dengan status "pending"
2. KETIKA record mitra_earnings dibuat, Sistem_Escrow HARUS menghitung net_amount sebagai gross_amount dikurangi komisi
3. Sistem_Escrow HARUS mengatur escrow_release_at ke 24 jam setelah penyelesaian pemesanan
4. KETIKA escrow_release_at tercapai dan is_disputed adalah false, Sistem_Escrow HARUS memperbarui status ke "available"
5. KETIKA pengguna mengajukan sengketa, Sistem_Escrow HARUS mengatur is_disputed ke true dan mencegah pelepasan otomatis
6. KETIKA admin menyelesaikan sengketa, Sistem_Escrow HARUS mencatat dispute_resolved_at dan memperbarui status sesuai
7. Sistem_Escrow HARUS menghitung komisi menggunakan commission_rate yang tersimpan di record pemesanan
8. DIMANA pemesanan dibatalkan sebelum penyelesaian, Sistem_Escrow HARUS TIDAK membuat record mitra_earnings

### Kebutuhan 11: Pembatalan Pengguna dengan Kebijakan Pengembalian Dana Berbasis Waktu

**Cerita Pengguna:** Sebagai pengguna, saya ingin membatalkan pemesanan saya dengan pengembalian dana yang sesuai berdasarkan waktu, sehingga saya dapat mengubah rencana sambil memahami dampak finansialnya.

#### Kriteria Penerimaan

1. KETIKA pengguna membatalkan lebih dari 2 hari sebelum scheduled_at (>H-2), Sistem_Pembatalan HARUS mengatur refund_percentage ke 100
2. KETIKA pengguna membatalkan 1 hari sebelum scheduled_at (H-1), Sistem_Pembatalan HARUS mengatur refund_percentage ke 50
3. KETIKA pengguna membatalkan pada hari scheduled_at (H-0), Sistem_Pembatalan HARUS mengatur refund_percentage ke 0
4. KETIKA pembatalan diproses, Sistem_Pembatalan HARUS memperbarui status pemesanan ke "dibatalkan_pelanggan"
5. KETIKA pembatalan diproses, Sistem_Pembatalan HARUS mencatat cancelled_by sebagai "user", cancel_reason, dan timestamp cancelled_at
6. KETIKA pembatalan diproses, Sistem_Pembatalan HARUS menghitung refund_amount berdasarkan refund_percentage dan total_amount
7. KETIKA refund_amount lebih besar dari nol, Sistem_Pembatalan HARUS memulai transaksi pengembalian dana Xendit
8. KETIKA pembatalan diproses, Sistem_Pembatalan HARUS melepaskan slot waktu yang dikunci untuk pemesanan ulang
9. Sistem_Pembatalan HARUS mengirim notifikasi push ke pengguna dan mitra tentang pembatalan
10. JIKA status pemesanan bukan "terjadwal", MAKA Sistem_Pembatalan HARUS mencegah pembatalan pengguna

### Kebutuhan 12: Pembatalan Mitra dengan Sistem Penalti

**Cerita Pengguna:** Sebagai platform, saya ingin memberikan penalti kepada mitra yang membatalkan pemesanan, sehingga penyedia jasa menjaga keandalan dan pengguna menerima pengembalian dana penuh.

#### Kriteria Penerimaan

1. KETIKA mitra membatalkan pemesanan, Sistem_Pembatalan HARUS mengatur refund_percentage ke 100 tanpa memandang waktu
2. KETIKA mitra membatalkan pemesanan, Sistem_Pembatalan HARUS memperbarui status pemesanan ke "dibatalkan_mitra"
3. KETIKA mitra membatalkan pemesanan, Sistem_Pembatalan HARUS mencatat cancelled_by sebagai "mitra" dan cancel_reason
4. KETIKA mitra membatalkan pemesanan, Sistem_Penalti HARUS membuat record mitra_penalties dengan tipe "cancel"
5. KETIKA penalti pembatalan mitra dibuat, Sistem_Penalti HARUS mengurangi 10 Service_Protection_Points
6. KETIKA penalti pembatalan mitra dibuat, Sistem_Penalti HARUS mengurangi 0.1 dari rating_avg
7. KETIKA Service_Protection_Points turun di bawah 50, Sistem_Penalti HARUS mengatur mitra is_available ke false
8. Sistem_Pembatalan HARUS memproses pengembalian dana penuh ke pengguna via Xendit
9. Sistem_Pembatalan HARUS mengirim notifikasi ke pengguna, mitra, dan admin tentang pembatalan dan penalti


### Kebutuhan 13: Progresi Status Pemesanan

**Cerita Pengguna:** Sebagai pengguna dan mitra, saya ingin melacak status pemesanan melalui siklus hidupnya, sehingga saya mengetahui keadaan terkini dari janji layanan.

#### Kriteria Penerimaan

1. KETIKA pemesanan dibuat dan dibayar, Sistem_Pemesanan HARUS mengatur status ke "terjadwal"
2. KETIKA mitra menekan "Mulai perjalanan", Sistem_Pemesanan HARUS memperbarui status ke "berlangsung" dan mencatat timestamp started_at
3. KETIKA mitra menandai layanan selesai, Sistem_Pemesanan HARUS memperbarui status ke "selesai" dan mencatat timestamp completed_at
4. Sistem_Pemesanan HARUS memvalidasi transisi status mengikuti alur yang diizinkan: terjadwal → berlangsung → selesai
5. JIKA transisi status tidak valid, MAKA Sistem_Pemesanan HARUS mengembalikan error yang menjelaskan transisi yang valid
6. KETIKA status pemesanan berubah, Sistem_Notifikasi HARUS mengirim pembaruan real-time ke pengguna dan mitra via Supabase Realtime
7. Sistem_Pemesanan HARUS mencegah perubahan status pada pemesanan dengan status "dibatalkan_pelanggan", "dibatalkan_mitra", atau "no_show"

### Kebutuhan 14: Penanganan No-Show dengan Bukti GPS

**Cerita Pengguna:** Sebagai mitra, saya ingin melaporkan ketika pengguna tidak hadir pada waktu terjadwal dengan bukti foto bertag GPS, sehingga saya menerima kompensasi untuk biaya perjalanan.

#### Kriteria Penerimaan

1. KETIKA mitra melaporkan no-show, Sistem_No_Show HARUS memerlukan foto dengan metadata GPS
2. Sistem_No_Show HARUS memvalidasi bahwa koordinat GPS foto berada dalam 100 meter dari alamat pemesanan
3. KETIKA laporan no-show yang valid diajukan, Sistem_No_Show HARUS memperbarui status pemesanan ke "no_show"
4. KETIKA no-show dikonfirmasi, Sistem_No_Show HARUS menyimpan no_show_photo_url dan no_show_timestamp
5. KETIKA no-show dikonfirmasi, Sistem_No_Show HARUS menghitung no_show_compensation sebagai 20% dari total_amount
6. KETIKA no-show dikonfirmasi, Sistem_No_Show HARUS membuat record mitra_earnings dengan jumlah kompensasi
7. Sistem_No_Show HARUS mengatur escrow_release_at ke 24 jam setelah konfirmasi no-show untuk jendela sengketa
8. Sistem_No_Show HARUS mengirim notifikasi ke pengguna dan admin tentang laporan no-show
9. JIKA koordinat GPS lebih dari 100 meter dari alamat pemesanan, MAKA Sistem_No_Show HARUS menolak laporan no-show

### Kebutuhan 15: Penyelesaian Layanan dengan Bukti Foto

**Cerita Pengguna:** Sebagai mitra, saya ingin mengunggah bukti foto saat menyelesaikan layanan, sehingga ada dokumentasi pekerjaan yang dilakukan.

#### Kriteria Penerimaan

1. KETIKA mitra menandai pemesanan sebagai selesai, Sistem_Pemesanan HARUS memerlukan unggahan completion_photo_url
2. KETIKA pemesanan ditandai selesai, Sistem_Pemesanan HARUS mengatur completed_at ke timestamp saat ini
3. KETIKA pemesanan ditandai selesai, Sistem_Pemesanan HARUS menghitung auto_complete_at sebagai 24 jam dari completed_at
4. KETIKA pemesanan ditandai selesai, Sistem_Escrow HARUS membuat record mitra_earnings dengan penahanan escrow
5. Sistem_Pemesanan HARUS menyimpan foto penyelesaian di Supabase Storage dengan booking_id di path file
6. KETIKA pemesanan ditandai selesai, Sistem_Notifikasi HARUS meminta pengguna untuk memberikan ulasan


### Kebutuhan 16: Pelepasan Escrow Otomatis

**Cerita Pengguna:** Sebagai mitra, saya ingin pendapatan saya dilepaskan secara otomatis setelah 24 jam jika tidak ada sengketa yang diajukan, sehingga saya menerima pembayaran tanpa intervensi manual.

#### Kriteria Penerimaan

1. KETIKA timestamp escrow_release_at tercapai, Sistem_Escrow HARUS memeriksa apakah is_disputed adalah false
2. DIMANA is_disputed adalah false pada escrow_release_at, Sistem_Escrow HARUS memperbarui status mitra_earnings ke "available"
3. Sistem_Escrow HARUS menjalankan pekerjaan terjadwal setiap jam untuk memproses pelepasan escrow yang tertunda
4. KETIKA status pendapatan berubah ke "available", Sistem_Notifikasi HARUS memberitahu mitra
5. DIMANA is_disputed adalah true, Sistem_Escrow HARUS TIDAK melepaskan dana secara otomatis sampai dispute_resolved_at diatur
6. Sistem_Escrow HARUS membuat entri audit log untuk setiap pelepasan otomatis

### Kebutuhan 17: Sistem Ulasan dan Rating

**Cerita Pengguna:** Sebagai pengguna, saya ingin memberikan rating dan ulasan mitra setelah penyelesaian layanan, sehingga saya dapat berbagi pengalaman dan membantu pengguna lain membuat keputusan yang tepat.

#### Kriteria Penerimaan

1. KETIKA pengguna mengirimkan ulasan, Sistem_Ulasan HARUS memvalidasi bahwa status pemesanan adalah "selesai"
2. Sistem_Ulasan HARUS memvalidasi bahwa rating antara 1 dan 5 bintang
3. KETIKA ulasan dikirimkan, Sistem_Ulasan HARUS menyimpan rating, komentar, dan mengatur is_visible ke true
4. Sistem_Ulasan HARUS mencegah ulasan duplikat untuk pemesanan yang sama
5. KETIKA ulasan dikirimkan, Sistem_Ulasan HARUS menghitung ulang rating_avg mitra dari semua ulasan yang terlihat
6. Sistem_Ulasan HARUS memperbarui rating_avg mitra menggunakan rumus: SUM(rating) / COUNT(reviews)
7. KETIKA admin menyembunyikan ulasan, Sistem_Ulasan HARUS mengatur is_visible ke false dan menghitung ulang rating_avg
8. DIMANA mitra atau admin menambahkan admin_response, Sistem_Ulasan HARUS menyimpan teks respons
9. Sistem_Ulasan HARUS menampilkan ulasan dalam urutan menurun berdasarkan created_at

### Kebutuhan 18: Chat Real-Time Antara Pengguna dan Mitra

**Cerita Pengguna:** Sebagai pengguna dan mitra, saya ingin berkomunikasi secara real-time tentang detail pemesanan, sehingga saya dapat mengklarifikasi kebutuhan dan mengkoordinasikan pengiriman layanan.

#### Kriteria Penerimaan

1. KETIKA pemesanan dibuat, Sistem_Chat HARUS membuat record chat_rooms yang menghubungkan user_id, mitra_id, dan booking_id
2. KETIKA pengguna atau mitra mengirim pesan, Sistem_Chat HARUS menyimpan content, sender_id, dan type di tabel messages
3. Sistem_Chat HARUS mendukung tipe pesan: "text", "image", dan "system"
4. KETIKA pesan dikirim, Sistem_Chat HARUS menyiarkannya via Supabase Realtime ke pihak lain
5. KETIKA pesan diterima, Sistem_Chat HARUS mengatur is_read ke false awalnya
6. KETIKA penerima melihat pesan, Sistem_Chat HARUS memperbarui is_read ke true
7. KETIKA pesan dikirim, Sistem_Chat HARUS memperbarui chat_rooms.last_message_at ke timestamp saat ini
8. DIMANA tipe pesan adalah "image", Sistem_Chat HARUS menyimpan gambar di Supabase Storage dan menyimpan image_url
9. Sistem_Chat HARUS mengurutkan pesan berdasarkan created_at secara ascending
10. Sistem_Chat HARUS mengirim notifikasi push untuk pesan baru ketika aplikasi penerima di background


### Kebutuhan 19: Sistem Notifikasi Push

**Cerita Pengguna:** Sebagai pengguna, saya ingin menerima notifikasi push untuk event pemesanan penting, sehingga saya tetap terinformasi tentang janji layanan saya.

#### Kriteria Penerimaan

1. KETIKA pengguna mendaftar atau login, Sistem_Notifikasi HARUS menyimpan token FCM di users.fcm_token
2. KETIKA event pemesanan terjadi, Sistem_Notifikasi HARUS membuat record notifications dengan type, title, body, dan data
3. Sistem_Notifikasi HARUS mengirim notifikasi push FCM untuk tipe event: booking_new, booking_confirmed, payment_success, booking_started, booking_completed, booking_cancelled, message_received, review_requested
4. KETIKA notifikasi push dikirim, Sistem_Notifikasi HARUS menyertakan booking_id di payload data untuk deep linking
5. KETIKA pengguna mengetuk notifikasi, Aplikasi_Mobile HARUS menavigasi ke layar yang relevan berdasarkan tipe notifikasi
6. KETIKA pengguna melihat notifikasi di dalam aplikasi, Sistem_Notifikasi HARUS mengatur is_read ke true dan mencatat timestamp read_at
7. Sistem_Notifikasi HARUS mengirim notifikasi pengingat 24 jam sebelum scheduled_at untuk pemesanan dengan status "terjadwal"
8. Sistem_Notifikasi HARUS menangani refresh token FCM dan memperbarui users.fcm_token sesuai
9. JIKA token FCM tidak valid atau kedaluwarsa, MAKA Sistem_Notifikasi HARUS mencatat error dan melanjutkan tanpa memblokir operasi

### Kebutuhan 20: Pelacakan Pendapatan Mitra

**Cerita Pengguna:** Sebagai mitra, saya ingin melacak pendapatan saya dengan rincian jelas dari jumlah bruto, komisi, dan jumlah bersih, sehingga saya memahami penghasilan saya.

#### Kriteria Penerimaan

1. KETIKA pemesanan selesai, Sistem_Pendapatan HARUS membuat record mitra_earnings dengan gross_amount sama dengan total_amount pemesanan
2. Sistem_Pendapatan HARUS menghitung komisi sebagai gross_amount dikalikan commission_rate dari pemesanan
3. Sistem_Pendapatan HARUS menghitung net_amount sebagai gross_amount dikurangi komisi
4. Sistem_Pendapatan HARUS mengatur status awal ke "pending" dengan penahanan escrow
5. KETIKA pendapatan dilepaskan dari escrow, Sistem_Pendapatan HARUS memperbarui status ke "available"
6. Sistem_Pendapatan HARUS menyediakan tampilan ringkasan yang menunjukkan total pending, available, dan disbursed per mitra
7. DIMANA status pendapatan adalah "available", Dashboard_Mitra HARUS menampilkannya sebagai siap untuk penarikan

### Kebutuhan 21: Pencairan Pendapatan Mitra

**Cerita Pengguna:** Sebagai mitra, saya ingin menarik pendapatan tersedia saya ke rekening bank, sehingga saya dapat menerima pembayaran untuk layanan yang telah selesai.

#### Kriteria Penerimaan

1. KETIKA mitra meminta pencairan, Sistem_Pencairan HARUS memvalidasi bahwa bank_name, bank_account, dan bank_holder sudah dikonfigurasi
2. Sistem_Pencairan HARUS memvalidasi bahwa total pendapatan tersedia lebih besar dari jumlah penarikan minimum
3. KETIKA pencairan dimulai, Sistem_Pencairan HARUS membuat transaksi pencairan Xendit
4. KETIKA Xendit mengkonfirmasi pencairan, Sistem_Pencairan HARUS memperbarui status mitra_earnings ke "disbursed" dan mencatat timestamp disbursed_at
5. Sistem_Pencairan HARUS menyimpan xendit_disbursement_id untuk pelacakan transaksi
6. Sistem_Pencairan HARUS mendekripsi bank_account menggunakan AES-256 sebelum mengirim ke Xendit
7. JIKA pencairan gagal, MAKA Sistem_Pencairan HARUS mengembalikan status ke "available" dan memberitahu mitra
8. Sistem_Pencairan HARUS menerapkan jumlah penarikan minimum sebesar 50000 IDR


### Kebutuhan 22: Sistem Pelacakan Penalti Mitra

**Cerita Pengguna:** Sebagai administrator platform, saya ingin melacak penalti mitra untuk pelanggaran, sehingga akuntabilitas terjaga dan pelaku buruk teridentifikasi.

#### Kriteria Penerimaan

1. KETIKA event penalti terjadi, Sistem_Penalti HARUS membuat record mitra_penalties dengan type, penalty_points, dan notes
2. Sistem_Penalti HARUS mendukung tipe penalti: "cancel", "no_show", "late", "dispute_lost"
3. KETIKA penalti dibuat, Sistem_Penalti HARUS mengurangi penalty_points dari mitra_profiles.service_protection_points
4. DIMANA tipe penalti adalah "cancel", Sistem_Penalti HARUS mengurangi 10 poin
5. DIMANA tipe penalti adalah "no_show", Sistem_Penalti HARUS mengurangi 15 poin
6. DIMANA tipe penalti adalah "dispute_lost", Sistem_Penalti HARUS mengurangi 20 poin
7. KETIKA Service_Protection_Points turun di bawah 50, Sistem_Penalti HARUS mengatur mitra is_available ke false
8. KETIKA Service_Protection_Points mencapai nol, Sistem_Penalti HARUS mengatur mitra is_active ke false
9. Sistem_Penalti HARUS mengizinkan admin untuk menyesuaikan Service_Protection_Points secara manual dengan justifikasi di notes
10. Sistem_Penalti HARUS membuat entri audit log untuk semua operasi penalti

### Kebutuhan 23: Pencarian Mitra Terdekat dengan GPS

**Cerita Pengguna:** Sebagai pengguna, saya ingin menemukan penyedia jasa di dekat lokasi saya, sehingga saya dapat memesan mitra yang dapat menjangkau saya dengan cepat.

#### Kriteria Penerimaan

1. KETIKA pengguna mencari mitra, Sistem_Pencarian HARUS menggunakan koordinat GPS pengguna
2. Sistem_Pencarian HARUS menghitung jarak menggunakan rumus Haversine dari lokasi pengguna ke latitude/longitude mitra
3. Sistem_Pencarian HARUS memfilter mitra dimana jarak yang dihitung kurang dari atau sama dengan service_radius
4. Sistem_Pencarian HARUS hanya mengembalikan mitra dimana is_verified adalah true dan is_available adalah true
5. Sistem_Pencarian HARUS mengurutkan hasil berdasarkan jarak secara ascending
6. DIMANA filter kategori diterapkan, Sistem_Pencarian HARUS hanya mengembalikan mitra yang menawarkan layanan dalam kategori tersebut
7. Sistem_Pencarian HARUS menampilkan jarak dalam kilometer dengan presisi satu desimal
8. Sistem_Pencarian HARUS menggunakan indeks spasial PostgreSQL untuk kueri lokasi yang efisien

### Kebutuhan 24: Manajemen Kategori Layanan

**Cerita Pengguna:** Sebagai admin, saya ingin mengelola kategori layanan, sehingga layanan terorganisir dan mudah ditemukan.

#### Kriteria Penerimaan

1. KETIKA admin membuat kategori, Sistem_Kategori HARUS memvalidasi bahwa name dan slug unik
2. Sistem_Kategori HARUS menghasilkan slug dari name dengan mengkonversi ke huruf kecil dan mengganti spasi dengan tanda hubung
3. KETIKA kategori dibuat, Sistem_Kategori HARUS mengatur is_active ke true secara default
4. Sistem_Kategori HARUS mengizinkan pengaturan sort_order untuk urutan tampilan
5. KETIKA admin menonaktifkan kategori, Sistem_Kategori HARUS mengatur is_active ke false
6. DIMANA kategori is_active adalah false, Sistem_Pencarian HARUS menyembunyikan layanan dalam kategori tersebut dari pencarian pengguna
7. Sistem_Kategori HARUS menyimpan icon_url untuk ikon kategori yang ditampilkan di aplikasi mobile


### Kebutuhan 25: Manajemen Konfigurasi Sistem

**Cerita Pengguna:** Sebagai admin, saya ingin mengkonfigurasi pengaturan seluruh sistem, sehingga aturan bisnis dapat disesuaikan tanpa perubahan kode.

#### Kriteria Penerimaan

1. KETIKA admin memperbarui pengaturan, Sistem_Pengaturan HARUS memvalidasi value_type cocok dengan format nilai yang diberikan
2. Sistem_Pengaturan HARUS mendukung opsi value_type: "string", "number", "boolean", "json"
3. KETIKA pengaturan diperbarui, Sistem_Pengaturan HARUS mencatat timestamp updated_at dan ID pengguna updated_by
4. DIMANA is_public adalah true, Sistem_Pengaturan HARUS mengizinkan akses sisi klien ke nilai pengaturan
5. DIMANA is_public adalah false, Sistem_Pengaturan HARUS membatasi akses hanya ke Edge Functions sisi server
6. Sistem_Pengaturan HARUS menyediakan pengaturan untuk: commission_rate, no_show_compensation_percentage, minimum_withdrawal_amount, booking_advance_hours, escrow_hold_hours
7. Sistem_Pengaturan HARUS membuat entri audit log untuk semua perubahan pengaturan
8. KETIKA nilai pengaturan diambil, Sistem_Pengaturan HARUS mem-parse-nya sesuai value_type

### Kebutuhan 26: Pencatatan Audit untuk Operasi Sensitif

**Cerita Pengguna:** Sebagai administrator platform, saya ingin log audit komprehensif dari operasi sensitif, sehingga saya dapat melacak perubahan dan menyelidiki masalah.

#### Kriteria Penerimaan

1. KETIKA operasi sensitif terjadi, Sistem_Audit HARUS membuat record audit_logs dengan action, table_name, dan record_id
2. Sistem_Audit HARUS menyimpan old_values dan new_values sebagai JSONB untuk pelacakan perubahan
3. Sistem_Audit HARUS mencatat user_id, ip_address, dan user_agent untuk setiap operasi
4. Sistem_Audit HARUS mencatat aksi: role_change, mitra_verify, booking_cancel, payment_refund, penalty_create, settings_update, dispute_resolve
5. Sistem_Audit HARUS mengatur created_at ke timestamp saat ini untuk setiap entri log
6. Sistem_Audit HARUS menyediakan pemfilteran berdasarkan user_id, action, table_name, dan rentang tanggal
7. Sistem_Audit HARUS menyimpan log audit minimal 2 tahun untuk kepatuhan

### Kebutuhan 27: Sistem Penyelesaian Sengketa

**Cerita Pengguna:** Sebagai admin, saya ingin menyelesaikan sengketa antara pengguna dan mitra, sehingga konflik ditangani secara adil dengan perlindungan escrow.

#### Kriteria Penerimaan

1. KETIKA pengguna mengajukan sengketa, Sistem_Sengketa HARUS mengatur mitra_earnings.is_disputed ke true
2. KETIKA sengketa diajukan, Sistem_Sengketa HARUS mencegah pelepasan escrow otomatis
3. KETIKA admin menyelesaikan sengketa mendukung pengguna, Sistem_Sengketa HARUS memulai pengembalian dana penuh dan mengatur status pendapatan ke "cancelled"
4. KETIKA admin menyelesaikan sengketa mendukung mitra, Sistem_Sengketa HARUS melepaskan escrow dan mengatur status pendapatan ke "available"
5. KETIKA sengketa diselesaikan, Sistem_Sengketa HARUS mencatat timestamp dispute_resolved_at
6. Sistem_Sengketa HARUS membuat record mitra_penalties dengan tipe "dispute_lost" jika diselesaikan melawan mitra
7. Sistem_Sengketa HARUS mengirim notifikasi ke pengguna dan mitra tentang resolusi
8. Sistem_Sengketa HARUS membuat entri audit log detail untuk keputusan penyelesaian sengketa


### Kebutuhan 28: Kebijakan Row Level Security (RLS)

**Cerita Pengguna:** Sebagai arsitek platform, saya ingin kebijakan keamanan tingkat database diterapkan, sehingga pengguna hanya dapat mengakses data mereka sendiri terlepas dari kerentanan sisi klien.

#### Kriteria Penerimaan

1. Sistem_Keamanan HARUS menerapkan kebijakan RLS pada semua tabel yang berisi data pengguna
2. Sistem_Keamanan HARUS mengizinkan pengguna membaca hanya record mereka sendiri di tabel users, bookings, payments, dan notifications
3. Sistem_Keamanan HARUS mengizinkan mitra membaca pemesanan dimana mitra_id cocok dengan profil mereka
4. Sistem_Keamanan HARUS mengizinkan peran admin untuk melewati kebijakan RLS untuk operasi administratif
5. Sistem_Keamanan HARUS mencegah pengguna memodifikasi data pengguna lain melalui operasi UPDATE atau DELETE
6. Sistem_Keamanan HARUS mengizinkan akses baca publik ke tabel categories dan mitra_services untuk penemuan layanan
7. Sistem_Keamanan HARUS membatasi tabel mitra_earnings dan mitra_penalties hanya untuk pemilik mitra dan admin
8. Sistem_Keamanan HARUS menerapkan kebijakan di tingkat PostgreSQL, bukan tingkat aplikasi

### Kebutuhan 29: Implementasi Soft Delete Data

**Cerita Pengguna:** Sebagai administrator platform, saya ingin record yang dihapus di-soft-delete, sehingga data dapat dipulihkan dan jejak audit terjaga.

#### Kriteria Penerimaan

1. KETIKA record dihapus, Sistem_Manajemen_Data HARUS mengatur deleted_at ke timestamp saat ini alih-alih menghapus record
2. Sistem_Manajemen_Data HARUS mengecualikan record dimana deleted_at bukan null dari kueri default
3. Sistem_Manajemen_Data HARUS mengizinkan admin melihat record yang di-soft-delete untuk pemulihan
4. Sistem_Manajemen_Data HARUS menerapkan soft delete untuk tabel: users, mitra_profiles, bookings, mitra_services
5. DIMANA pengguna di-soft-delete, Sistem_Manajemen_Data HARUS juga men-soft-delete mitra_profile terkait mereka
6. Sistem_Manajemen_Data HARUS menyediakan fungsi admin untuk menghapus permanen record yang lebih dari 2 tahun

### Kebutuhan 30: Pembuatan dan Validasi Kode Booking

**Cerita Pengguna:** Sebagai platform, saya ingin kode booking yang unik dan mudah dibaca manusia, sehingga pemesanan dapat dengan mudah direferensikan dalam percakapan dukungan.

#### Kriteria Penerimaan

1. KETIKA pemesanan dibuat, Sistem_Pemesanan HARUS menghasilkan booking_code dalam format "JSN-YYYYMMDD-XXXX"
2. Sistem_Pemesanan HARUS menggunakan tanggal saat ini untuk bagian YYYYMMDD
3. Sistem_Pemesanan HARUS menghasilkan XXXX sebagai string alfanumerik acak 4 karakter
4. Sistem_Pemesanan HARUS memvalidasi keunikan booking_code sebelum menyimpan
5. JIKA terjadi tabrakan, MAKA Sistem_Pemesanan HARUS menghasilkan ulang bagian XXXX hingga 3 kali
6. Sistem_Pemesanan HARUS menggunakan huruf besar dan angka saja (mengecualikan karakter ambigu: 0, O, I, 1)
7. UNTUK SEMUA kode booking, Sistem_Pemesanan HARUS memastikan panjangnya tepat 16 karakter


### Kebutuhan 31: Penegakan Logika Bisnis Edge Function

**Cerita Pengguna:** Sebagai arsitek platform, saya ingin logika bisnis kritis dieksekusi di sisi server, sehingga keamanan dan integritas data tidak dapat dilewati oleh manipulasi klien.

#### Kriteria Penerimaan

1. Arsitektur_Platform HARUS mengimplementasikan logika pembuatan pemesanan di Edge_Function "create-booking"
2. Arsitektur_Platform HARUS mengimplementasikan pemrosesan webhook pembayaran di Edge_Function "xendit-webhook"
3. Arsitektur_Platform HARUS mengimplementasikan konfirmasi pemesanan di Edge_Function "confirm-booking"
4. Arsitektur_Platform HARUS mengimplementasikan pembayaran mitra di Edge_Function "payout-mitra"
5. Arsitektur_Platform HARUS memvalidasi semua parameter input di Edge Functions sebelum pemrosesan
6. Arsitektur_Platform HARUS mengembalikan respons error terstruktur dengan kode status HTTP dan pesan error
7. Arsitektur_Platform HARUS mengimplementasikan pembatasan laju pada Edge Functions untuk mencegah penyalahgunaan
8. Arsitektur_Platform HARUS mencatat semua pemanggilan Edge Function untuk pemantauan dan debugging

### Kebutuhan 32: Pembaruan Status Pemesanan Real-Time

**Cerita Pengguna:** Sebagai pengguna dan mitra, saya ingin melihat perubahan status pemesanan secara real-time, sehingga saya segera mengetahui pembaruan tanpa perlu refresh.

#### Kriteria Penerimaan

1. KETIKA status pemesanan berubah, Sistem_Realtime HARUS menyiarkan pembaruan via Supabase Realtime
2. Aplikasi_Mobile HARUS berlangganan perubahan tabel bookings yang difilter berdasarkan user_id atau mitra_id
3. KETIKA pembaruan realtime diterima, Aplikasi_Mobile HARUS memperbarui UI dalam 2 detik
4. Sistem_Realtime HARUS menggunakan koneksi WebSocket untuk pembaruan latensi rendah
5. JIKA koneksi WebSocket terputus, MAKA Aplikasi_Mobile HARUS secara otomatis terhubung kembali dalam 5 detik
6. Sistem_Realtime HARUS hanya menyiarkan pembaruan ke pengguna yang berwenang berdasarkan kebijakan RLS
7. Aplikasi_Mobile HARUS menangani pembaruan bersamaan menggunakan pembaruan UI optimistik dengan rekonsiliasi server

### Kebutuhan 33: Unggah dan Penyimpanan Gambar

**Cerita Pengguna:** Sebagai pengguna dan mitra, saya ingin mengunggah gambar secara aman, sehingga saya dapat menyediakan dokumentasi visual untuk profil, layanan, dan pemesanan.

#### Kriteria Penerimaan

1. KETIKA gambar diunggah, Sistem_Penyimpanan HARUS memvalidasi tipe file adalah JPEG, PNG, atau WebP
2. Sistem_Penyimpanan HARUS memvalidasi ukuran file kurang dari 5MB
3. KETIKA gambar diunggah, Sistem_Penyimpanan HARUS menghasilkan nama file unik menggunakan UUID
4. Sistem_Penyimpanan HARUS menyimpan gambar di bucket Supabase Storage yang diorganisir berdasarkan tipe: avatars, ktp, services, completion_photos, no_show_photos, chat_images
5. Sistem_Penyimpanan HARUS mengatur akses publik untuk gambar avatar dan layanan
6. Sistem_Penyimpanan HARUS mengatur akses privat untuk KTP, foto penyelesaian, dan foto no-show
7. KETIKA mengakses gambar privat, Sistem_Penyimpanan HARUS memvalidasi otorisasi pengguna melalui kebijakan RLS
8. Sistem_Penyimpanan HARUS menghasilkan signed URL dengan kedaluwarsa 1 jam untuk gambar privat
9. Sistem_Penyimpanan HARUS secara otomatis mengkompresi gambar yang lebih besar dari 1MB untuk mengurangi biaya penyimpanan


### Kebutuhan 34: Perhitungan Metrik Kinerja Mitra

**Cerita Pengguna:** Sebagai platform, saya ingin secara otomatis menghitung metrik kinerja mitra, sehingga pengguna dapat membuat keputusan yang tepat berdasarkan data yang andal.

#### Kriteria Penerimaan

1. KETIKA pemesanan selesai, Sistem_Metrik HARUS menambah mitra_profiles.total_orders sebesar 1
2. Sistem_Metrik HARUS menghitung completion_rate sebagai (completed_bookings / total_bookings) * 100
3. KETIKA ulasan dikirimkan, Sistem_Metrik HARUS menghitung ulang rating_avg sebagai SUM(rating) / COUNT(visible_reviews)
4. Sistem_Metrik HARUS memperbarui metrik secara real-time menggunakan trigger database
5. Sistem_Metrik HARUS membulatkan rating_avg ke 2 tempat desimal
6. Sistem_Metrik HARUS membulatkan completion_rate ke 2 tempat desimal
7. DIMANA pemesanan dibatalkan oleh mitra, Sistem_Metrik HARUS TIDAK menghitungnya ke numerator completion_rate
8. Sistem_Metrik HARUS menghitung years_active berdasarkan mitra_profiles.created_at

### Kebutuhan 35: Perhitungan Ketersediaan Slot

**Cerita Pengguna:** Sebagai pengguna, saya ingin melihat slot waktu yang tersedia untuk pemesanan, sehingga saya dapat memilih waktu janji yang nyaman.

#### Kriteria Penerimaan

1. KETIKA pengguna melihat ketersediaan mitra, Sistem_Slot HARUS menghitung slot tersedia berdasarkan mitra_schedules
2. Sistem_Slot HARUS mengecualikan slot waktu yang sudah dipesan
3. Sistem_Slot HARUS mengecualikan tanggal di mitra_blocked_dates
4. Sistem_Slot HARUS hanya menampilkan slot dimana scheduled_at minimal 2 jam di masa depan
5. Sistem_Slot HARUS menerapkan batas max_slots_per_day dari mitra_schedules
6. Sistem_Slot HARUS menghasilkan slot dengan menambahkan open_time sebesar slot_duration_minutes sampai close_time
7. Sistem_Slot HARUS menampilkan slot dalam zona waktu lokal mitra
8. DIMANA mitra memiliki is_available diatur ke false, Sistem_Slot HARUS mengembalikan array kosong

### Kebutuhan 36: Pemilihan Metode Pembayaran

**Cerita Pengguna:** Sebagai pengguna, saya ingin memilih metode pembayaran pilihan saya, sehingga saya dapat membayar menggunakan opsi yang paling nyaman bagi saya.

#### Kriteria Penerimaan

1. KETIKA pengguna memulai pembayaran, Sistem_Pembayaran HARUS menyajikan metode yang tersedia: QRIS, transfer bank, e-wallet, dan kartu
2. KETIKA pengguna memilih QRIS, Sistem_Pembayaran HARUS menghasilkan kode QR via Xendit
3. KETIKA pengguna memilih transfer bank, Sistem_Pembayaran HARUS menyediakan detail virtual account
4. KETIKA pengguna memilih e-wallet, Sistem_Pembayaran HARUS mendukung OVO, GoPay, Dana, dan LinkAja
5. KETIKA pengguna memilih kartu, Sistem_Pembayaran HARUS mengarahkan ke halaman pembayaran kartu aman Xendit
6. Sistem_Pembayaran HARUS menyimpan payment_method yang dipilih di tabel payments
7. Sistem_Pembayaran HARUS menampilkan instruksi pembayaran spesifik untuk metode yang dipilih
8. Sistem_Pembayaran HARUS memperbarui status pembayaran secara otomatis ketika Xendit mengirim konfirmasi webhook


### Kebutuhan 37: Enkripsi Rekening Bank untuk Mitra

**Cerita Pengguna:** Sebagai mitra, saya ingin informasi rekening bank saya disimpan secara aman, sehingga data keuangan saya terlindungi dari akses tidak sah.

#### Kriteria Penerimaan

1. KETIKA mitra menyimpan informasi rekening bank, Sistem_Keamanan HARUS mengenkripsi bank_account menggunakan enkripsi AES-256
2. Sistem_Keamanan HARUS menyimpan kunci enkripsi di variabel lingkungan, bukan di database
3. KETIKA informasi rekening bank diambil untuk ditampilkan, Sistem_Keamanan HARUS mendekripsi dan menyamarkan semua kecuali 4 digit terakhir
4. KETIKA informasi rekening bank digunakan untuk pencairan, Sistem_Keamanan HARUS mendekripsi nilai lengkap
5. Sistem_Keamanan HARUS memvalidasi format bank_account cocok dengan pola rekening bank Indonesia
6. Sistem_Keamanan HARUS menyimpan bank_name dan bank_holder dalam teks biasa untuk keperluan tampilan
7. Sistem_Keamanan HARUS mencatat semua akses ke data rekening bank terenkripsi di log audit

### Kebutuhan 38: Manajemen Preferensi Notifikasi

**Cerita Pengguna:** Sebagai pengguna, saya ingin mengelola preferensi notifikasi saya, sehingga saya hanya menerima notifikasi yang saya minati.

#### Kriteria Penerimaan

1. KETIKA pengguna memperbarui preferensi notifikasi, Sistem_Notifikasi HARUS menyimpan preferensi di pengaturan pengguna
2. Sistem_Notifikasi HARUS mendukung kategori preferensi: booking_updates, payment_updates, promotional, chat_messages
3. DIMANA pengguna telah menonaktifkan kategori notifikasi, Sistem_Notifikasi HARUS TIDAK mengirim notifikasi tipe tersebut
4. Sistem_Notifikasi HARUS selalu mengirim notifikasi kritis (payment_success, booking_cancelled) tanpa memandang preferensi
5. KETIKA pengguna menonaktifkan semua notifikasi, Sistem_Notifikasi HARUS tetap membuat record notifikasi dalam aplikasi
6. Sistem_Notifikasi HARUS menghormati izin notifikasi tingkat sistem dari OS mobile

### Kebutuhan 39: Pencarian dan Filter untuk Layanan Mitra

**Cerita Pengguna:** Sebagai pengguna, saya ingin mencari dan memfilter layanan mitra, sehingga saya dapat menemukan tepat apa yang saya butuhkan.

#### Kriteria Penerimaan

1. KETIKA pengguna memasukkan kueri pencarian, Sistem_Pencarian HARUS mencocokkan terhadap nama dan deskripsi layanan menggunakan pencarian teks penuh
2. Sistem_Pencarian HARUS mendukung pemfilteran berdasarkan category_id
3. Sistem_Pencarian HARUS mendukung pemfilteran berdasarkan rentang harga (min_price, max_price)
4. Sistem_Pencarian HARUS mendukung pemfilteran berdasarkan rating mitra (ambang batas rating minimum)
5. Sistem_Pencarian HARUS mendukung pengurutan berdasarkan: jarak, rating, harga_rendah_ke_tinggi, harga_tinggi_ke_rendah
6. Sistem_Pencarian HARUS hanya mengembalikan layanan dimana is_active adalah true
7. Sistem_Pencarian HARUS hanya mengembalikan layanan dari mitra terverifikasi
8. Sistem_Pencarian HARUS mengimplementasikan paginasi dengan 20 hasil per halaman


### Kebutuhan 40: Riwayat dan Pemfilteran Pemesanan

**Cerita Pengguna:** Sebagai pengguna dan mitra, saya ingin melihat riwayat pemesanan saya dengan opsi pemfilteran, sehingga saya dapat melacak janji yang lalu dan akan datang.

#### Kriteria Penerimaan

1. KETIKA pengguna melihat riwayat pemesanan, Sistem_Pemesanan HARUS mengembalikan pemesanan diurutkan berdasarkan scheduled_at secara menurun
2. Sistem_Pemesanan HARUS mendukung pemfilteran berdasarkan status: semua, terjadwal, berlangsung, selesai, dibatalkan
3. Sistem_Pemesanan HARUS mendukung pemfilteran berdasarkan rentang tanggal (start_date, end_date)
4. DIMANA pengguna melihat riwayat, Sistem_Pemesanan HARUS mengembalikan pemesanan dimana user_id cocok
5. DIMANA mitra melihat riwayat, Sistem_Pemesanan HARUS mengembalikan pemesanan dimana mitra_id cocok
6. Sistem_Pemesanan HARUS menyertakan data terkait: booking_items, mitra_profile, user_profile dalam respons
7. Sistem_Pemesanan HARUS mengimplementasikan paginasi dengan 10 pemesanan per halaman
8. Sistem_Pemesanan HARUS menampilkan booking_code secara menonjol untuk referensi mudah

### Kebutuhan 41: Penanganan Error dan Umpan Balik Pengguna

**Cerita Pengguna:** Sebagai pengguna, saya ingin pesan error yang jelas ketika sesuatu salah, sehingga saya memahami apa yang terjadi dan cara memperbaikinya.

#### Kriteria Penerimaan

1. KETIKA error terjadi, Sistem_Penanganan_Error HARUS mengembalikan respons error terstruktur dengan kode dan pesan
2. Sistem_Penanganan_Error HARUS menggunakan kode status HTTP: 400 untuk error validasi, 401 untuk error autentikasi, 403 untuk error otorisasi, 404 untuk tidak ditemukan, 500 untuk error server
3. KETIKA error validasi terjadi, Sistem_Penanganan_Error HARUS menyertakan pesan error spesifik per field
4. Aplikasi_Mobile HARUS menampilkan pesan error dalam bahasa yang ramah pengguna, bukan jargon teknis
5. KETIKA error jaringan terjadi, Aplikasi_Mobile HARUS menampilkan opsi coba lagi
6. Sistem_Penanganan_Error HARUS mencatat semua error dengan stack trace untuk debugging
7. DIMANA error dapat dipulihkan, Sistem_Penanganan_Error HARUS menyarankan tindakan korektif dalam pesan error

### Kebutuhan 42: Pembatasan Laju dan Pencegahan Penyalahgunaan

**Cerita Pengguna:** Sebagai administrator platform, saya ingin pembatasan laju pada endpoint API, sehingga sistem terlindungi dari penyalahgunaan dan serangan DDoS.

#### Kriteria Penerimaan

1. Sistem_Pembatasan_Laju HARUS menerapkan batas pada Edge Functions: 100 permintaan per menit per pengguna untuk operasi pemesanan
2. Sistem_Pembatasan_Laju HARUS menerapkan batas pada autentikasi: 5 percobaan login per 15 menit per alamat IP
3. KETIKA batas laju terlampaui, Sistem_Pembatasan_Laju HARUS mengembalikan status HTTP 429 dengan header retry-after
4. Sistem_Pembatasan_Laju HARUS menggunakan Redis atau Supabase untuk pelacakan batas laju terdistribusi
5. Sistem_Pembatasan_Laju HARUS mengecualikan pengguna admin dari batas laju
6. Sistem_Pembatasan_Laju HARUS mencatat pelanggaran batas laju untuk pemantauan keamanan
7. DIMANA aktivitas mencurigakan terdeteksi, Sistem_Pembatasan_Laju HARUS memblokir sementara alamat IP


### Kebutuhan 43: Notifikasi Pengingat Pemesanan

**Cerita Pengguna:** Sebagai pengguna, saya ingin menerima pengingat sebelum janji terjadwal saya, sehingga saya tidak lupa tentang pemesanan saya.

#### Kriteria Penerimaan

1. Sistem_Pengingat HARUS mengirim notifikasi 24 jam sebelum scheduled_at untuk pemesanan dengan status "terjadwal"
2. Sistem_Pengingat HARUS mengirim notifikasi 1 jam sebelum scheduled_at untuk pemesanan dengan status "terjadwal"
3. Sistem_Pengingat HARUS menjalankan pekerjaan terjadwal setiap 15 menit untuk memeriksa pemesanan yang akan datang
4. KETIKA pengingat dikirim, Sistem_Pengingat HARUS menyertakan booking_code, nama mitra, dan waktu terjadwal
5. Sistem_Pengingat HARUS TIDAK mengirim pengingat untuk pemesanan yang dibatalkan atau selesai
6. DIMANA pengguna telah menonaktifkan notifikasi booking_updates, Sistem_Pengingat HARUS TIDAK mengirim pengingat
7. Sistem_Pengingat HARUS menandai pengingat sebagai terkirim untuk mencegah notifikasi duplikat

### Kebutuhan 44: Analitik Dashboard Mitra

**Cerita Pengguna:** Sebagai mitra, saya ingin melihat analitik tentang kinerja saya, sehingga saya dapat melacak pertumbuhan bisnis saya.

#### Kriteria Penerimaan

1. Sistem_Dashboard HARUS menampilkan total pendapatan untuk bulan ini, bulan lalu, dan sepanjang waktu
2. Sistem_Dashboard HARUS menampilkan jumlah pemesanan berdasarkan status: terjadwal, berlangsung, selesai, dibatalkan
3. Sistem_Dashboard HARUS menampilkan rating rata-rata dan total ulasan
4. Sistem_Dashboard HARUS menampilkan tingkat penyelesaian sebagai persentase
5. Sistem_Dashboard HARUS menampilkan Service_Protection_Points dengan indikator visual (hijau >80, kuning 50-80, merah <50)
6. Sistem_Dashboard HARUS menampilkan grafik pendapatan selama 6 bulan terakhir
7. Sistem_Dashboard HARUS menampilkan jumlah escrow tertunda dan jumlah penarikan tersedia
8. Sistem_Dashboard HARUS memperbarui data secara real-time menggunakan langganan Supabase Realtime

### Kebutuhan 45: Ikhtisar Dashboard Admin

**Cerita Pengguna:** Sebagai admin, saya ingin melihat statistik seluruh platform, sehingga saya dapat memantau kesehatan sistem dan kinerja bisnis.

#### Kriteria Penerimaan

1. Dashboard_Admin HARUS menampilkan total pengguna, total mitra, dan total mitra terverifikasi
2. Dashboard_Admin HARUS menampilkan jumlah pemesanan berdasarkan status untuk hari ini, minggu ini, dan bulan ini
3. Dashboard_Admin HARUS menampilkan total pendapatan dan komisi platform untuk bulan ini
4. Dashboard_Admin HARUS menampilkan jumlah verifikasi mitra yang tertunda
5. Dashboard_Admin HARUS menampilkan jumlah sengketa aktif
6. Dashboard_Admin HARUS menampilkan log audit terbaru untuk operasi sensitif
7. Dashboard_Admin HARUS menampilkan metrik kesehatan sistem: koneksi database, waktu respons Edge Function, tingkat error
8. Dashboard_Admin HARUS memperbarui statistik setiap 30 detik


### Kebutuhan 46: Aturan Validasi Pemesanan

**Cerita Pengguna:** Sebagai platform, saya ingin menerapkan aturan validasi pemesanan, sehingga hanya pemesanan yang valid yang dibuat.

#### Kriteria Penerimaan

1. KETIKA pemesanan dibuat, Sistem_Validasi HARUS memverifikasi bahwa scheduled_at minimal 2 jam di masa depan
2. Sistem_Validasi HARUS memverifikasi bahwa mitra yang dipilih terverifikasi (is_verified adalah true)
3. Sistem_Validasi HARUS memverifikasi bahwa mitra yang dipilih tersedia (is_available adalah true)
4. Sistem_Validasi HARUS memverifikasi bahwa semua layanan yang dipilih milik mitra yang ditentukan
5. Sistem_Validasi HARUS memverifikasi bahwa semua layanan yang dipilih aktif (is_active adalah true)
6. Sistem_Validasi HARUS memverifikasi bahwa slot waktu belum dipesan
7. Sistem_Validasi HARUS memverifikasi bahwa tanggal terjadwal tidak ada di mitra_blocked_dates
8. Sistem_Validasi HARUS memverifikasi bahwa kuantitas adalah bilangan bulat positif
9. JIKA validasi apapun gagal, MAKA Sistem_Validasi HARUS mengembalikan pesan error deskriptif dan mencegah pembuatan pemesanan

### Kebutuhan 47: Pencegahan Pemesanan Bersamaan

**Cerita Pengguna:** Sebagai platform, saya ingin mencegah pemesanan ganda pada slot waktu, sehingga mitra tidak menerima janji yang bertentangan.

#### Kriteria Penerimaan

1. KETIKA pemesanan dibuat, Sistem_Konkurensi HARUS menggunakan penguncian tingkat baris database pada slot waktu
2. Sistem_Konkurensi HARUS memvalidasi ketersediaan slot dalam transaksi database
3. JIKA pemesanan lain dibuat untuk slot yang sama selama pemrosesan transaksi, MAKA Sistem_Konkurensi HARUS rollback dan mengembalikan error
4. Sistem_Konkurensi HARUS menggunakan PostgreSQL SELECT FOR UPDATE untuk mengunci record slot
5. Sistem_Konkurensi HARUS melepaskan kunci setelah commit atau rollback transaksi
6. Sistem_Konkurensi HARUS mengimplementasikan mekanisme retry dengan exponential backoff untuk konflik kunci
7. Sistem_Konkurensi HARUS timeout percobaan kunci setelah 5 detik

### Kebutuhan 48: Penanganan Zona Waktu

**Cerita Pengguna:** Sebagai pengguna dan mitra, saya ingin semua waktu ditampilkan dalam zona waktu lokal saya, sehingga saya tidak melewatkan janji karena kebingungan zona waktu.

#### Kriteria Penerimaan

1. Platform HARUS menyimpan semua timestamp dalam UTC di database
2. KETIKA menampilkan waktu ke pengguna, Aplikasi_Mobile HARUS mengkonversi UTC ke zona waktu lokal perangkat
3. KETIKA pengguna memilih waktu pemesanan, Aplikasi_Mobile HARUS mengkonversi waktu lokal ke UTC sebelum mengirim ke server
4. Aplikasi_Mobile HARUS menampilkan singkatan zona waktu (misalnya WIB, WITA, WIT) di samping waktu
5. Platform HARUS menangani transisi daylight saving time dengan benar
6. DIMANA mitra beroperasi di zona waktu berbeda dari pengguna, Aplikasi_Mobile HARUS menampilkan kedua zona waktu
7. Platform HARUS menggunakan format ISO 8601 untuk semua serialisasi timestamp


### Kebutuhan 49: Dukungan Mode Offline

**Cerita Pengguna:** Sebagai pengguna, saya ingin melihat riwayat pemesanan saya saat offline, sehingga saya dapat mengakses informasi penting tanpa koneksi internet.

#### Kriteria Penerimaan

1. KETIKA Aplikasi_Mobile kehilangan koneksi internet, Sistem_Offline HARUS menampilkan data pemesanan yang di-cache
2. Sistem_Offline HARUS meng-cache riwayat pemesanan 30 hari terakhir secara lokal
3. Sistem_Offline HARUS meng-cache profil pengguna dan profil mitra untuk pemesanan terbaru
4. KETIKA Aplikasi_Mobile mendapatkan kembali koneksi internet, Sistem_Offline HARUS menyinkronkan perubahan lokal dengan server
5. Sistem_Offline HARUS menampilkan indikator visual saat beroperasi dalam mode offline
6. DIMANA tindakan memerlukan komunikasi server, Sistem_Offline HARUS mengantrekan tindakan dan mengeksekusi saat online
7. Sistem_Offline HARUS mencegah pembuatan pemesanan dan operasi pembayaran dalam mode offline

### Kebutuhan 50: Deep Linking untuk Notifikasi

**Cerita Pengguna:** Sebagai pengguna, saya ingin mengetuk notifikasi dan langsung dibawa ke layar yang relevan, sehingga saya dapat dengan cepat mengakses detail pemesanan.

#### Kriteria Penerimaan

1. KETIKA pengguna mengetuk notifikasi, Aplikasi_Mobile HARUS mem-parse payload data notifikasi
2. DIMANA tipe notifikasi adalah "booking_new" atau "booking_confirmed", Aplikasi_Mobile HARUS menavigasi ke layar detail pemesanan
3. DIMANA tipe notifikasi adalah "message_received", Aplikasi_Mobile HARUS menavigasi ke layar chat untuk pemesanan tersebut
4. DIMANA tipe notifikasi adalah "payment_success", Aplikasi_Mobile HARUS menavigasi ke layar konfirmasi pembayaran
5. DIMANA tipe notifikasi adalah "review_requested", Aplikasi_Mobile HARUS menavigasi ke layar pengiriman ulasan
6. Aplikasi_Mobile HARUS menangani deep link saat aplikasi tertutup, di background, atau di foreground
7. JIKA layar target memerlukan autentikasi dan pengguna belum login, MAKA Aplikasi_Mobile HARUS menavigasi ke layar login terlebih dahulu

### Kebutuhan 51: Toggle Ketersediaan Mitra

**Cerita Pengguna:** Sebagai mitra, saya ingin dengan cepat mengaktifkan/menonaktifkan ketersediaan saya, sehingga saya dapat mengontrol kapan saya menerima pemesanan baru.

#### Kriteria Penerimaan

1. KETIKA mitra menonaktifkan ketersediaan, Sistem_Mitra HARUS mengatur is_available ke false
2. DIMANA is_available adalah false, Sistem_Pencarian HARUS mengecualikan mitra dari hasil pencarian pengguna
3. DIMANA is_available adalah false, Sistem_Slot HARUS mengembalikan ketersediaan kosong
4. KETIKA mitra mengaktifkan ketersediaan, Sistem_Mitra HARUS mengatur is_available ke true
5. Sistem_Mitra HARUS mengizinkan toggle ketersediaan hanya untuk mitra terverifikasi
6. Sistem_Mitra HARUS menampilkan toggle switch yang menonjol di dashboard mitra
7. KETIKA ketersediaan berubah, Sistem_Mitra HARUS mengirim notifikasi konfirmasi


### Kebutuhan 52: Validasi dan Sanitasi Input

**Cerita Pengguna:** Sebagai platform, saya ingin semua input pengguna divalidasi dan disanitasi, sehingga sistem terlindungi dari serangan injeksi dan korupsi data.

#### Kriteria Penerimaan

1. Sistem_Validasi HARUS memvalidasi format email menggunakan standar RFC 5322
2. Sistem_Validasi HARUS memvalidasi nomor telepon cocok dengan format Indonesia (+62 atau 08)
3. Sistem_Validasi HARUS mensanitasi semua input teks untuk menghapus tag HTML dan konten script
4. Sistem_Validasi HARUS memvalidasi input numerik berada dalam rentang yang dapat diterima
5. Sistem_Validasi HARUS memvalidasi input tanggal adalah tanggal yang valid dan dalam rentang yang dapat diterima
6. Sistem_Validasi HARUS memvalidasi format UUID untuk semua parameter ID
7. Sistem_Validasi HARUS membatasi panjang field teks: nama (100 karakter), deskripsi (500 karakter), catatan (1000 karakter)
8. Sistem_Validasi HARUS menolak input yang mengandung pola injeksi SQL
9. Sistem_Validasi HARUS menggunakan kueri berparameter untuk semua operasi database

### Kebutuhan 53: Manajemen Sesi dan Keamanan

**Cerita Pengguna:** Sebagai pengguna, saya ingin sesi saya aman dan otomatis kedaluwarsa setelah tidak aktif, sehingga akun saya terlindungi.

#### Kriteria Penerimaan

1. Sistem_Sesi HARUS mengatur kedaluwarsa sesi ke 30 hari untuk sesi aktif
2. Sistem_Sesi HARUS memperbarui token sesi pada setiap permintaan terautentikasi
3. KETIKA pengguna logout, Sistem_Sesi HARUS membatalkan token sesi segera
4. Sistem_Sesi HARUS menyimpan token sesi menggunakan cookie HTTP-only yang aman jika berlaku
5. Sistem_Sesi HARUS mengimplementasikan perlindungan CSRF untuk operasi yang mengubah state
6. DIMANA token sesi tidak valid atau kedaluwarsa, Sistem_Sesi HARUS mengembalikan HTTP 401 dan meminta autentikasi ulang
7. Sistem_Sesi HARUS mengizinkan hanya satu sesi aktif per pengguna per perangkat
8. Sistem_Sesi HARUS mencatat semua event pembuatan dan penghentian sesi

### Kebutuhan 54: Pemantauan dan Optimasi Kinerja

**Cerita Pengguna:** Sebagai administrator platform, saya ingin memantau kinerja sistem, sehingga saya dapat mengidentifikasi dan menyelesaikan bottleneck.

#### Kriteria Penerimaan

1. Sistem_Pemantauan HARUS melacak waktu eksekusi Edge Function dan mencatat permintaan lambat (>2 detik)
2. Sistem_Pemantauan HARUS melacak kinerja kueri database dan mencatat kueri lambat (>500ms)
3. Sistem_Pemantauan HARUS memantau waktu respons endpoint API dan tingkat error
4. Sistem_Pemantauan HARUS melacak jumlah koneksi Supabase Realtime dan throughput pesan
5. Sistem_Pemantauan HARUS memperingatkan administrator ketika tingkat error melebihi 5% selama 5 menit
6. Sistem_Pemantauan HARUS melacak tingkat keberhasilan pemrosesan pembayaran dan memperingatkan pada kegagalan
7. Sistem_Pemantauan HARUS menyediakan dashboard kinerja dengan metrik real-time
8. Sistem_Pemantauan HARUS menyimpan log kinerja selama 90 hari


### Kebutuhan 55: Backup dan Pemulihan Data

**Cerita Pengguna:** Sebagai administrator platform, saya ingin backup database otomatis, sehingga data dapat dipulihkan jika terjadi kegagalan sistem.

#### Kriteria Penerimaan

1. Sistem_Backup HARUS membuat backup database penuh setiap hari pada pukul 02:00 UTC
2. Sistem_Backup HARUS menyimpan backup harian selama 30 hari
3. Sistem_Backup HARUS membuat backup mingguan yang disimpan selama 90 hari
4. Sistem_Backup HARUS memverifikasi integritas backup setelah setiap operasi backup
5. Sistem_Backup HARUS menyimpan backup di lokasi yang terpisah secara geografis dari database utama
6. Sistem_Backup HARUS mengenkripsi backup menggunakan enkripsi AES-256
7. Sistem_Backup HARUS memperingatkan administrator jika operasi backup gagal
8. Sistem_Backup HARUS menyediakan prosedur pemulihan terdokumentasi dengan RTO 4 jam

### Kebutuhan 56: Kepatuhan Aksesibilitas

**Cerita Pengguna:** Sebagai pengguna dengan disabilitas, saya ingin aplikasi dapat diakses, sehingga saya dapat menggunakan semua fitur tanpa memandang kemampuan saya.

#### Kriteria Penerimaan

1. Aplikasi_Mobile HARUS menyediakan alternatif teks untuk semua gambar dan ikon
2. Aplikasi_Mobile HARUS mendukung screen reader di iOS (VoiceOver) dan Android (TalkBack)
3. Aplikasi_Mobile HARUS mempertahankan rasio kontras minimum 4.5:1 untuk teks normal
4. Aplikasi_Mobile HARUS mendukung penskalaan teks hingga 200% tanpa kehilangan fungsionalitas
5. Aplikasi_Mobile HARUS menyediakan navigasi keyboard untuk semua elemen interaktif
6. Aplikasi_Mobile HARUS menggunakan struktur HTML/widget semantik untuk navigasi screen reader yang tepat
7. Aplikasi_Mobile HARUS menyediakan indikator fokus yang jelas untuk elemen interaktif
8. Aplikasi_Mobile HARUS menghindari interaksi berbasis waktu yang tidak dapat diperpanjang

### Kebutuhan 57: Persiapan Dukungan Multi-Bahasa

**Cerita Pengguna:** Sebagai arsitek platform, saya ingin sistem dirancang untuk dukungan multi-bahasa di masa depan, sehingga internasionalisasi dapat ditambahkan dengan mudah.

#### Kriteria Penerimaan

1. Aplikasi_Mobile HARUS mengeksternalisasi semua string yang menghadap pengguna ke file lokalisasi
2. Aplikasi_Mobile HARUS menggunakan framework internasionalisasi Flutter (flutter_localizations)
3. Aplikasi_Mobile HARUS memformat tanggal, waktu, dan angka sesuai pengaturan locale
4. Aplikasi_Mobile HARUS memformat nilai mata uang sesuai locale (IDR untuk Indonesia)
5. Database HARUS menggunakan encoding Unicode (UTF-8) untuk semua field teks
6. Aplikasi_Mobile HARUS mendeteksi bahasa perangkat dan menggunakan terjemahan yang sesuai jika tersedia
7. Aplikasi_Mobile HARUS fallback ke Bahasa Indonesia (id_ID) sebagai bahasa default
8. Aplikasi_Mobile HARUS mengizinkan pengguna memilih bahasa secara manual di pengaturan


### Kebutuhan 58: Kebutuhan Pengujian dan Jaminan Kualitas

**Cerita Pengguna:** Sebagai tim pengembangan, saya ingin kebutuhan pengujian komprehensif didefinisikan, sehingga kami dapat memastikan keandalan dan kebenaran sistem.

#### Kriteria Penerimaan

1. Tim_Pengembangan HARUS mengimplementasikan unit test untuk semua Edge Functions dengan cakupan kode minimum 80%
2. Tim_Pengembangan HARUS mengimplementasikan integration test untuk pemrosesan webhook pembayaran dengan lingkungan test Xendit
3. Tim_Pengembangan HARUS mengimplementasikan property-based test untuk pembuatan kode booking untuk memverifikasi keunikan dan format
4. Tim_Pengembangan HARUS mengimplementasikan property-based test untuk perhitungan pengembalian dana pembatalan untuk memverifikasi kebenaran di semua rentang waktu
5. Tim_Pengembangan HARUS mengimplementasikan property-based test untuk perhitungan ketersediaan slot untuk memverifikasi tidak ada skenario pemesanan ganda
6. Tim_Pengembangan HARUS mengimplementasikan property-based test untuk waktu pelepasan escrow untuk memverifikasi penegakan penahanan 24 jam
7. Tim_Pengembangan HARUS mengimplementasikan end-to-end test untuk alur pemesanan lengkap: pembuatan → pembayaran → pembaruan status → penyelesaian
8. Tim_Pengembangan HARUS mengimplementasikan load test untuk memverifikasi sistem menangani 100 permintaan pemesanan bersamaan
9. Tim_Pengembangan HARUS mengimplementasikan security test untuk penegakan kebijakan RLS
10. Tim_Pengembangan HARUS mengimplementasikan accessibility test menggunakan alat otomatis dan pengujian screen reader manual

---

## Metadata Dokumen

**Versi Dokumen:** 1.0  
**Tanggal Dibuat:** 2024  
**Tipe Alur Kerja:** Requirements-First (Kebutuhan Terlebih Dahulu)  
**Tipe Spec:** Fitur  
**Total Kebutuhan:** 58  
**Status:** Draft Awal - Menunggu Review

---

## Langkah Selanjutnya

Setelah persetujuan dokumen kebutuhan ini, alur kerja akan berlanjut ke:

1. **Fase Desain**: Membuat dokumen desain teknis yang menentukan arsitektur, model data, kontrak API, dan pendekatan implementasi
2. **Fase Pembuatan Tugas**: Memecah desain menjadi tugas pengembangan yang dapat ditindaklanjuti dengan kriteria penerimaan
3. **Fase Implementasi**: Mengeksekusi tugas mengikuti prinsip Clean Architecture dengan Flutter dan Supabase

---

## Checklist Review

Sebelum melanjutkan ke fase desain, verifikasi:

- [ ] Semua kebutuhan mengikuti pola EARS dengan benar
- [ ] Semua kebutuhan mematuhi aturan kualitas INCOSE
- [ ] Semua istilah teknis didefinisikan dalam Glosarium
- [ ] Kriteria penerimaan spesifik dan dapat diuji
- [ ] Peluang property-based testing teridentifikasi
- [ ] Kasus tepi dan penanganan error tercakup
- [ ] Kebutuhan keamanan dan perlindungan data lengkap
- [ ] Kebutuhan real-time dan kinerja ditentukan
- [ ] Titik integrasi (Xendit, Supabase, FCM) terdokumentasi
