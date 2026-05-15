# Rencana Implementasi: Platform Marketplace JASAne

## Ikhtisar

Rencana implementasi ini memecah platform Marketplace JASAne menjadi tugas-tugas diskrit dan berurutan untuk membangun aplikasi mobile Flutter dengan backend Supabase. Platform ini mengimplementasikan pemesanan instan, routing berbasis peran, pemrosesan pembayaran via Xendit, chat real-time, dan sistem penalti komprehensif.

Implementasi mengikuti prinsip Clean Architecture dengan organisasi berbasis fitur, menggunakan Riverpod 2.x untuk manajemen state. Tugas-tugas diorganisir untuk membangun infrastruktur dasar terlebih dahulu, kemudian melapisi fitur secara bertahap dengan pengujian terintegrasi sepanjang proses.

## Tugas-Tugas

### 1. Setup Proyek dan Infrastruktur Inti

- [ ] 1.1 Inisialisasi proyek Flutter dengan struktur Clean Architecture
  - Buat proyek Flutter 3.x dengan dukungan Android dan iOS
  - Siapkan struktur folder: `lib/core/`, `lib/features/`
  - Buat subdirektori: `constants/`, `errors/`, `network/`, `utils/`, `theme/`
  - Konfigurasi `pubspec.yaml` dengan dependensi: `flutter_riverpod: ^2.4.0`, `supabase_flutter: ^2.0.0`, `dio: ^5.3.0`, `firebase_messaging: ^14.7.0`, `geolocator: ^10.1.0`, `image_picker: ^1.0.4`, `intl: ^0.18.1`, `freezed_annotation: ^2.4.1`, `json_annotation: ^4.8.1`
  - Tambahkan dev dependencies: `build_runner: ^2.4.6`, `freezed: ^2.4.5`, `json_serializable: ^6.7.1`
  - _Kebutuhan: Semua fitur bergantung pada fondasi ini_

- [ ] 1.2 Konfigurasi klien Supabase dan variabel lingkungan
  - Buat `lib/core/network/supabase_client.dart` dengan klien Supabase singleton
  - Siapkan file `.env` dengan `SUPABASE_URL`, `SUPABASE_ANON_KEY`, `XENDIT_API_KEY`
  - Buat `lib/core/constants/api_endpoints.dart` dengan endpoint Edge Function
  - Inisialisasi Supabase di `main.dart` sebelum `runApp()`
  - _Kebutuhan: 1.1, 1.2, 8.10, 9.1_

- [ ] 1.3 Siapkan penanganan error dan kelas failure
  - Buat `lib/core/errors/failures.dart` dengan hierarki sealed class: `ServerFailure`, `NetworkFailure`, `AuthFailure`, `ValidationFailure`
  - Buat `lib/core/errors/exceptions.dart` dengan exception kustom
  - Implementasikan tipe `Either` menggunakan paket `dartz` untuk penanganan error fungsional
  - _Kebutuhan: 1.3, 8.9_

- [ ] 1.4 Buat utilitas inti dan formatter
  - Implementasikan `lib/core/utils/date_formatter.dart` dengan format locale Indonesia
  - Implementasikan `lib/core/utils/currency_formatter.dart` untuk format IDR
  - Implementasikan `lib/core/utils/validators.dart` dengan validasi email, telepon, dan kode booking
  - Buat `lib/core/utils/booking_code_generator.dart` untuk format "JSN-YYYYMMDD-XXXX"
  - _Kebutuhan: 8.2, 20.1_

- [ ] 1.5 Konfigurasi tema aplikasi dan sistem desain
  - Buat `lib/core/theme/app_colors.dart` dengan palet warna brand
  - Buat `lib/core/theme/app_theme.dart` dengan konfigurasi tema terang dan gelap
  - Definisikan text styles, button styles, dan tema dekorasi input
  - _Kebutuhan: Konsistensi UI di semua fitur_

- [ ] 1.6 Siapkan Firebase Cloud Messaging
  - Tambahkan file konfigurasi Firebase (`google-services.json` untuk Android, `GoogleService-Info.plist` untuk iOS)
  - Buat `lib/core/services/fcm_service.dart` dengan inisialisasi FCM dan manajemen token
  - Implementasikan handler pesan foreground, background, dan terminated state
  - Konfigurasi notification channels untuk Android
  - _Kebutuhan: 19.1, 19.2, 19.8_

### 2. Skema Database dan Setup Supabase

- [ ] 2.1 Buat skema database Supabase
  - Eksekusi migrasi SQL untuk 15 tabel: `users`, `mitra_profiles`, `mitra_penalties`, `categories`, `mitra_services`, `mitra_schedules`, `mitra_blocked_dates`, `bookings`, `booking_items`, `payments`, `mitra_earnings`, `reviews`, `chat_rooms`, `messages`, `notifications`
  - Siapkan foreign key constraints dan indeks
  - Buat indeks unik `booking_code` pada tabel `bookings`
  - Buat indeks spasial pada `mitra_profiles(latitude, longitude)` untuk kueri lokasi
  - _Kebutuhan: 8.1, 23.8_

- [ ] 2.2 Konfigurasi kebijakan Row Level Security (RLS)
  - Aktifkan RLS pada semua tabel
  - Buat kebijakan untuk `users`: pengguna dapat membaca/memperbarui profil mereka sendiri
  - Buat kebijakan untuk `mitra_profiles`: baca publik untuk mitra terverifikasi, pemilik dapat memperbarui
  - Buat kebijakan untuk `bookings`: pengguna melihat pemesanan mereka, mitra melihat pemesanan yang ditugaskan
  - Buat kebijakan untuk `messages`: peserta dapat membaca/menulis pesan di chat room mereka
  - Buat kebijakan untuk `mitra_earnings`: mitra hanya dapat melihat pendapatan mereka sendiri
  - _Kebutuhan: Prinsip desain security-first_

- [ ] 2.3 Siapkan bucket Supabase Storage
  - Buat bucket `avatars` dengan akses baca publik
  - Buat bucket `ktp-documents` dengan akses privat (admin dan pemilik saja)
  - Buat bucket `booking-photos` dengan akses baca terautentikasi
  - Buat bucket `chat-images` dengan akses hanya peserta
  - Konfigurasi kebijakan storage untuk setiap bucket
  - _Kebutuhan: 3.4, 14.1, 15.1, 18.8_

- [ ] 2.4 Buat fungsi dan trigger database
  - Buat fungsi `calculate_mitra_rating()` untuk menghitung ulang `rating_avg` pada insert/update review
  - Buat trigger pada tabel `reviews` untuk memanggil fungsi perhitungan rating
  - Buat fungsi `update_service_protection_points()` untuk menyesuaikan poin SP
  - Buat trigger pada `mitra_penalties` untuk memperbarui `service_protection_points` dan `is_available`
  - Buat fungsi `generate_booking_code()` untuk pembuatan kode unik
  - _Kebutuhan: 17.5, 17.6, 12.5, 12.7_

### 3. Fitur Autentikasi

- [ ] 3.1 Buat lapisan domain autentikasi
  - Definisikan entitas `lib/features/auth/domain/entities/user.dart` dengan enum `UserRole`
  - Buat interface `lib/features/auth/domain/repositories/auth_repository.dart`
  - Implementasikan use case: `LoginWithEmail`, `LoginWithGoogle`, `Logout`, `GetCurrentUser`
  - _Kebutuhan: 1.1, 1.2_

- [ ] 3.2 Implementasikan lapisan data autentikasi
  - Buat `lib/features/auth/data/models/user_model.dart` dengan serialisasi JSON
  - Implementasikan `lib/features/auth/data/datasources/auth_remote_datasource.dart` menggunakan Supabase Auth
  - Implementasikan `lib/features/auth/data/repositories/auth_repository_impl.dart`
  - Tangani alur Google OAuth dengan `supabase.auth.signInWithOAuth()`
  - _Kebutuhan: 1.1, 1.2, 1.5_

- [ ] 3.3 Bangun lapisan presentasi autentikasi
  - Buat `lib/features/auth/presentation/providers/auth_provider.dart` dengan `StateNotifierProvider`
  - Implementasikan `lib/features/auth/presentation/pages/login_page.dart` dengan tombol login email dan Google
  - Implementasikan `lib/features/auth/presentation/pages/register_page.dart` dengan form registrasi email
  - Buat `lib/features/auth/presentation/widgets/login_form.dart` dengan validasi
  - Tangani perubahan state autentikasi dan navigasi
  - _Kebutuhan: 1.1, 1.2, 1.3, 1.4_

- [ ] 3.4 Implementasikan sistem routing berbasis peran
  - Buat `lib/core/router/app_router.dart` dengan pembuatan rute berbasis peran
  - Implementasikan metode `_userRoutes()`, `_mitraRoutes()`, dan `_adminRoutes()`
  - Buat `lib/core/router/route_guards.dart` untuk mencegah akses tidak sah
  - Dengarkan perubahan profil pengguna via Supabase Realtime untuk transisi peran
  - _Kebutuhan: 2.1, 2.2, 2.3, 2.4, 2.5_


### 4. Profil Pengguna dan Pendaftaran Mitra

- [ ] 4.1 Buat lapisan domain dan data profil pengguna
  - Definisikan entitas `lib/features/profile/domain/entities/user_profile.dart`
  - Buat interface dan implementasi repository untuk operasi profil
  - Implementasikan use case: `GetUserProfile`, `UpdateUserProfile`, `UpgradeToMitra`
  - _Kebutuhan: 3.1, 3.2, 3.3_

- [ ] 4.2 Implementasikan alur pendaftaran mitra
  - Buat `lib/features/profile/presentation/pages/mitra_registration_page.dart`
  - Implementasikan unggah foto KTP dengan paket `image_picker`
  - Unggah KTP ke bucket Supabase Storage `ktp-documents`
  - Buat record `mitra_profiles` dengan `is_verified: false` dan `service_protection_points: 100`
  - Perbarui peran pengguna ke "mitra" di tabel `users`
  - _Kebutuhan: 3.1, 3.2, 3.3, 3.4, 3.5, 3.6_

- [ ] 4.3 Bangun UI profil pengguna
  - Buat `lib/features/profile/presentation/pages/profile_page.dart` dengan tampilan avatar, nama, telepon
  - Implementasikan fungsionalitas edit profil
  - Tambahkan tombol "Jadi Mitra" untuk pengguna dengan peran "user"
  - Tampilkan status verifikasi mitra untuk pengguna mitra
  - _Kebutuhan: 3.1, 3.6_

### 5. Sistem Verifikasi Admin

- [ ] 5.1 Buat lapisan domain verifikasi admin
  - Definisikan entitas `lib/features/admin/domain/entities/verification_request.dart`
  - Buat interface repository untuk operasi verifikasi
  - Implementasikan use case: `GetPendingVerifications`, `ApproveMitra`, `RejectMitra`
  - _Kebutuhan: 4.1, 4.2, 4.3, 4.4_

- [ ] 5.2 Implementasikan lapisan data verifikasi admin
  - Buat model data dan remote datasource untuk verifikasi
  - Implementasikan repository dengan kueri Supabase memfilter `is_verified: false`
  - Tangani persetujuan: perbarui `is_verified`, `verification_at`, `verified_by`, `is_available`
  - Tangani penolakan: atur `is_verified: false`, catat alasan penolakan
  - _Kebutuhan: 4.1, 4.2, 4.3, 4.4_

- [ ] 5.3 Bangun UI verifikasi admin
  - Buat `lib/features/admin/presentation/pages/verification_dashboard_page.dart`
  - Tampilkan daftar mitra tertunda dengan gambar KTP
  - Implementasikan aksi setuju/tolak dengan dialog konfirmasi
  - Tampilkan riwayat verifikasi dengan log audit
  - _Kebutuhan: 4.1, 4.2, 4.3, 4.4, 4.5, 4.6_

- [ ] 5.4 Implementasikan notifikasi verifikasi
  - Kirim notifikasi FCM ke mitra saat disetujui dengan tipe `mitra_verified`
  - Kirim notifikasi FCM ke mitra saat ditolak dengan tipe `mitra_rejected`
  - Buat record notifikasi di tabel `notifications`
  - _Kebutuhan: 4.6_

### 6. Manajemen Katalog Layanan

- [ ] 6.1 Buat lapisan domain dan data kategori layanan
  - Definisikan entitas `lib/features/services/domain/entities/category.dart`
  - Implementasikan repository untuk operasi CRUD kategori
  - Buat use case: `GetCategories`, `CreateCategory`, `UpdateCategory`
  - _Kebutuhan: 24.1, 24.2, 24.3, 24.4, 24.5, 24.6, 24.7_

- [ ] 6.2 Implementasikan lapisan domain dan data layanan mitra
  - Definisikan entitas `lib/features/services/domain/entities/mitra_service.dart` dengan enum `PriceUnit`
  - Buat interface repository untuk manajemen layanan
  - Implementasikan use case: `CreateService`, `UpdateService`, `DeactivateService`, `GetMitraServices`
  - _Kebutuhan: 5.1, 5.2, 5.3, 5.4, 5.5, 5.6, 5.7_

- [ ] 6.3 Bangun UI manajemen layanan untuk mitra
  - Buat `lib/features/services/presentation/pages/mitra_services_page.dart`
  - Implementasikan form pembuatan layanan dengan pemilihan kategori, harga, dan durasi
  - Tambahkan fungsionalitas edit dan nonaktifkan layanan
  - Tampilkan layanan aktif dan tidak aktif secara terpisah
  - Validasi harga > 0 dan tangani satuan harga "estimate"
  - _Kebutuhan: 5.1, 5.2, 5.3, 5.4, 5.5, 5.6, 5.7_

- [ ] 6.4 Buat UI manajemen kategori untuk admin
  - Buat `lib/features/admin/presentation/pages/category_management_page.dart`
  - Implementasikan pembuatan kategori dengan nama, pembuatan slug, dan unggah ikon
  - Tambahkan konfigurasi urutan tampilan
  - Implementasikan aktivasi/nonaktivasi kategori
  - _Kebutuhan: 24.1, 24.2, 24.3, 24.4, 24.5, 24.6, 24.7_

### 7. Jadwal dan Ketersediaan Mitra

- [ ] 7.1 Buat lapisan domain dan data jadwal
  - Definisikan entitas `lib/features/schedule/domain/entities/mitra_schedule.dart`
  - Definisikan value object `lib/features/schedule/domain/entities/time_slot.dart`
  - Buat interface repository untuk operasi jadwal
  - Implementasikan use case: `CreateSchedule`, `UpdateSchedule`, `GetMitraSchedules`, `GenerateTimeSlots`
  - _Kebutuhan: 6.1, 6.2, 6.3, 6.4, 6.5, 6.6, 6.7_

- [ ] 7.2 Implementasikan UI manajemen jadwal
  - Buat `lib/features/schedule/presentation/pages/schedule_management_page.dart`
  - Bangun editor jadwal mingguan dengan pemilihan hari
  - Implementasikan time picker untuk waktu buka/tutup
  - Tambahkan konfigurasi durasi slot dan maks slot per hari
  - Validasi close_time > open_time dan slot_duration antara 15-480 menit
  - _Kebutuhan: 6.1, 6.2, 6.3, 6.4, 6.5, 6.6, 6.7_

- [ ] 7.3 Buat manajemen tanggal diblokir
  - Definisikan entitas `lib/features/schedule/domain/entities/blocked_date.dart`
  - Implementasikan use case: `BlockDate`, `UnblockDate`, `GetBlockedDates`
  - Buat `lib/features/schedule/presentation/pages/blocked_dates_page.dart`
  - Implementasikan tampilan kalender dengan fungsionalitas pemblokiran tanggal
  - Izinkan pemblokiran tanggal hingga 365 hari ke depan
  - _Kebutuhan: 7.1, 7.2, 7.3, 7.4, 7.5_

- [ ] 7.4 Implementasikan perhitungan ketersediaan slot
  - Buat `lib/features/schedule/domain/services/slot_availability_service.dart`
  - Implementasikan logika untuk menghasilkan slot waktu dari konfigurasi jadwal
  - Filter tanggal diblokir dan slot yang sudah dipesan
  - Hitung slot tersedia dengan mempertimbangkan batas `max_slots_per_day`
  - _Kebutuhan: 6.7, 7.3, 8.4, 8.8_

### 8. Pencarian Mitra Terdekat

- [ ] 8.1 Buat lapisan domain pencarian
  - Definisikan entitas `lib/features/search/domain/entities/mitra_search_result.dart`
  - Buat interface repository dengan metode `searchNearbyMitra()`
  - Implementasikan use case: `SearchNearbyMitra` dengan koordinat GPS dan radius
  - _Kebutuhan: 23.1, 23.2, 23.3, 23.4, 23.5, 23.6, 23.7, 23.8_

- [ ] 8.2 Implementasikan lapisan data pencarian dengan rumus Haversine
  - Buat `lib/features/search/data/repositories/search_repository_impl.dart`
  - Implementasikan perhitungan jarak Haversine dalam kueri PostgreSQL atau Dart
  - Filter mitra berdasarkan `is_verified: true`, `is_available: true`, dan jarak <= `service_radius`
  - Urutkan hasil berdasarkan jarak ascending
  - Terapkan filter kategori jika disediakan
  - _Kebutuhan: 23.1, 23.2, 23.3, 23.4, 23.5, 23.6, 23.7, 23.8_

- [ ] 8.3 Bangun UI pencarian dengan integrasi GPS
  - Buat `lib/features/search/presentation/pages/search_page.dart`
  - Minta izin lokasi menggunakan paket `geolocator`
  - Dapatkan koordinat GPS saat ini dan kirim ke use case pencarian
  - Tampilkan hasil mitra dengan jarak dalam kilometer (1 desimal)
  - Implementasikan chip filter kategori
  - Tampilkan kartu profil mitra dengan rating, layanan, dan jarak
  - _Kebutuhan: 23.1, 23.2, 23.3, 23.4, 23.5, 23.6, 23.7_

### 9. Pembuatan Pemesanan dan Penguncian Instan

- [ ] 9.1 Buat lapisan domain pemesanan
  - Definisikan entitas `lib/features/booking/domain/entities/booking.dart` dengan enum `BookingStatus`
  - Definisikan entitas `lib/features/booking/domain/entities/booking_item.dart`
  - Buat interface repository untuk operasi pemesanan
  - Implementasikan use case: `CreateBooking`, `GetUserBookings`, `GetBookingById`, `CancelBooking`
  - _Kebutuhan: 8.1, 8.2, 8.3, 8.4, 8.5, 8.6, 8.7, 8.8, 8.9, 8.10_

- [ ] 9.2 Implementasikan lapisan data pemesanan
  - Buat `lib/features/booking/data/models/booking_model.dart` dengan serialisasi JSON
  - Implementasikan `lib/features/booking/data/repositories/booking_repository_impl.dart`
  - Tangani pembuatan pemesanan dengan memanggil Supabase Edge Function `create-booking`
  - Parse respons pemesanan termasuk URL invoice pembayaran
  - _Kebutuhan: 8.1, 8.2, 8.3, 8.10_

- [ ] 9.3 Bangun UI pembuatan pemesanan
  - Buat `lib/features/booking/presentation/pages/booking_creation_page.dart`
  - Tampilkan layanan mitra yang dipilih dengan pemilihan kuantitas
  - Implementasikan pemilih tanggal dan slot waktu
  - Tambahkan input alamat dengan koordinat GPS
  - Tampilkan rincian harga: subtotal, biaya platform (10%), total
  - Tampilkan ringkasan pemesanan sebelum konfirmasi
  - _Kebutuhan: 8.1, 8.2, 8.6, 8.7_

- [ ] 9.4 Implementasikan validasi dan penguncian slot
  - Validasi slot yang dipilih tersedia sebelum membuat pemesanan
  - Validasi `scheduled_at` minimal 2 jam di masa depan
  - Tangani error ketidaktersediaan slot dengan saran slot alternatif
  - Tampilkan pesan error untuk kegagalan validasi
  - _Kebutuhan: 8.4, 8.5, 8.8, 8.9_


### 10. Supabase Edge Functions - Logika Pemesanan

- [ ] 10.1 Buat Edge Function `create-booking`
  - Siapkan `supabase/functions/create-booking/index.ts` dengan runtime Deno
  - Validasi ketersediaan slot dengan mengkueri pemesanan yang ada
  - Hitung subtotal, platform_fee (10%), dan total_amount
  - Hasilkan booking_code unik menggunakan format `JSN-YYYYMMDD-XXXX`
  - Buat record pemesanan dengan status "terjadwal"
  - Buat record booking_items dengan snapshot layanan
  - Kunci slot waktu untuk mencegah pemesanan ganda
  - Buat invoice Xendit dan kembalikan URL pembayaran
  - Tangani error dan kembalikan kode status HTTP yang sesuai
  - _Kebutuhan: 8.1, 8.2, 8.3, 8.4, 8.6, 8.7, 8.8, 8.10, 9.1, 9.2_

- [ ] 10.2 Buat Edge Function `cancel-booking`
  - Siapkan `supabase/functions/cancel-booking/index.ts`
  - Hitung persentase pengembalian dana berdasarkan waktu pembatalan: >H-2 (100%), H-1 (50%), H-0 (0%)
  - Perbarui status pemesanan ke "dibatalkan_pelanggan" atau "dibatalkan_mitra"
  - Catat `cancelled_by`, `cancel_reason`, `cancelled_at`, `refund_percentage`, `refund_amount`
  - Proses pengembalian dana Xendit jika refund_amount > 0
  - Terapkan penalti mitra jika dibatalkan oleh mitra (10 poin SP, -0.1 rating)
  - Lepaskan slot waktu yang dikunci
  - Kirim notifikasi ke pengguna, mitra, dan admin
  - _Kebutuhan: 11.1, 11.2, 11.3, 11.4, 11.5, 11.6, 11.7, 11.8, 11.9, 11.10, 12.1, 12.2, 12.3, 12.4, 12.5, 12.6, 12.7, 12.8, 12.9_

- [ ] 10.3 Buat Edge Function `release-escrow`
  - Siapkan `supabase/functions/release-escrow/index.ts` sebagai fungsi terjadwal (per jam)
  - Kueri `mitra_earnings` dengan status "pending", `is_disputed: false`, dan `escrow_release_at <= NOW()`
  - Perbarui status ke "available" untuk pendapatan yang memenuhi syarat
  - Kirim notifikasi FCM ke mitra tentang pendapatan tersedia
  - Buat entri audit log untuk setiap pelepasan
  - _Kebutuhan: 16.1, 16.2, 16.3, 16.4, 16.5, 16.6_

- [ ] 10.4 Buat Edge Function `process-no-show`
  - Siapkan `supabase/functions/process-no-show/index.ts`
  - Validasi koordinat GPS berada dalam 100 meter dari alamat pemesanan
  - Unggah foto no-show ke Supabase Storage
  - Perbarui status pemesanan ke "no_show"
  - Hitung no_show_compensation sebagai 20% dari total_amount
  - Buat record mitra_earnings dengan jumlah kompensasi
  - Atur escrow_release_at ke 24 jam dari sekarang
  - Kirim notifikasi ke pengguna dan admin
  - _Kebutuhan: 14.1, 14.2, 14.3, 14.4, 14.5, 14.6, 14.7, 14.8, 14.9_

### 11. Integrasi Pembayaran dengan Xendit

- [ ] 11.1 Buat lapisan domain pembayaran
  - Definisikan entitas `lib/features/payment/domain/entities/payment.dart` dengan enum `PaymentStatus`
  - Buat interface repository untuk operasi pembayaran
  - Implementasikan use case: `CreatePayment`, `GetPaymentByBookingId`, `ProcessWebhook`
  - _Kebutuhan: 9.1, 9.2, 9.3, 9.4, 9.5, 9.6, 9.7, 9.8, 9.9, 9.10_

- [ ] 11.2 Implementasikan wrapper layanan Xendit
  - Buat `lib/core/services/xendit_service.dart` dengan klien HTTP Dio
  - Implementasikan metode `createInvoice()` dengan Xendit API v2
  - Dukung metode pembayaran: QRIS, transfer bank, e-wallet, kartu
  - Atur durasi invoice ke 24 jam (86400 detik)
  - Konfigurasi URL redirect sukses/gagal dengan deep linking
  - Implementasikan metode `createDisbursement()` untuk penarikan pendapatan mitra
  - _Kebutuhan: 9.1, 9.2, 9.3, 9.4, 21.3, 21.4_

- [ ] 11.3 Bangun UI pembayaran
  - Buat `lib/features/payment/presentation/pages/payment_page.dart`
  - Tampilkan URL invoice Xendit di WebView atau buka di browser
  - Tampilkan opsi metode pembayaran (QRIS, transfer, e-wallet, kartu)
  - Tampilkan timer hitung mundur kedaluwarsa pembayaran
  - Tangani redirect sukses/gagal pembayaran dengan deep linking
  - _Kebutuhan: 9.2, 9.3, 9.4_

- [ ] 11.4 Buat Edge Function `xendit-webhook`
  - Siapkan `supabase/functions/xendit-webhook/index.ts`
  - Verifikasi tanda tangan webhook Xendit untuk keamanan
  - Implementasikan pemeriksaan idempotency menggunakan `xendit_payment_id`
  - Tangani status "PAID": perbarui pembayaran ke "paid", atur `paid_at`, perbarui pemesanan ke "terjadwal"
  - Tangani status "EXPIRED": perbarui pembayaran ke "expired", lepaskan slot yang dikunci
  - Simpan payload webhook lengkap di field `xendit_raw`
  - Kirim notifikasi FCM pada pembayaran sukses
  - _Kebutuhan: 9.5, 9.6, 9.7, 9.8, 9.9, 9.10_

### 12. Manajemen Status Pemesanan

- [ ] 12.1 Implementasikan progresi status pemesanan
  - Buat `lib/features/booking/domain/services/booking_status_service.dart`
  - Validasi transisi status: terjadwal → berlangsung → selesai
  - Implementasikan `startBooking()` untuk memperbarui status ke "berlangsung" dan mengatur `started_at`
  - Implementasikan `completeBooking()` untuk memperbarui status ke "selesai" dan mengatur `completed_at`
  - Cegah transisi status tidak valid dengan pesan error deskriptif
  - _Kebutuhan: 13.1, 13.2, 13.3, 13.4, 13.5, 13.7_

- [ ] 12.2 Bangun UI detail pemesanan
  - Buat `lib/features/booking/presentation/pages/booking_detail_page.dart`
  - Tampilkan informasi pemesanan: kode, mitra, layanan, jadwal, alamat, harga
  - Tampilkan status saat ini dengan badge berwarna
  - Tampilkan aksi spesifik status: "Mulai perjalanan", "Tandai selesai", "Batalkan"
  - Tampilkan timeline pemesanan dengan timestamp
  - _Kebutuhan: 13.1, 13.2, 13.3_

- [ ] 12.3 Implementasikan pembaruan pemesanan real-time
  - Berlangganan perubahan pemesanan menggunakan Supabase Realtime di halaman detail pemesanan
  - Perbarui UI secara otomatis saat status berubah
  - Tampilkan notifikasi toast untuk pembaruan status
  - _Kebutuhan: 13.6_

- [ ] 12.4 Buat UI daftar pemesanan
  - Buat `lib/features/booking/presentation/pages/booking_list_page.dart`
  - Tampilkan pemesanan pengguna difilter berdasarkan status: akan datang, berlangsung, selesai, dibatalkan
  - Tampilkan kartu pemesanan dengan informasi kunci: kode, nama mitra, tanggal, status
  - Implementasikan fungsionalitas tarik-untuk-refresh
  - _Kebutuhan: 8.1, 13.1_

### 13. Penyelesaian Layanan dan Bukti Foto

- [ ] 13.1 Implementasikan unggah foto penyelesaian
  - Buat `lib/features/booking/presentation/widgets/completion_photo_picker.dart`
  - Gunakan `image_picker` untuk mengambil atau memilih foto
  - Unggah foto ke bucket Supabase Storage `booking-photos`
  - Hasilkan nama file unik dengan prefix booking_id
  - _Kebutuhan: 15.1, 15.5_

- [ ] 13.2 Buat alur penyelesaian
  - Tambahkan tombol "Tandai selesai" di halaman detail pemesanan untuk mitra
  - Wajibkan unggah foto penyelesaian sebelum menandai selesai
  - Panggil repository pemesanan untuk memperbarui status ke "selesai" dengan `completion_photo_url`
  - Atur timestamp `completed_at` dan hitung `auto_complete_at` (24 jam kemudian)
  - _Kebutuhan: 15.1, 15.2, 15.3_

- [ ] 13.3 Picu pembuatan escrow saat penyelesaian
  - Buat record `mitra_earnings` ketika status pemesanan berubah ke "selesai"
  - Hitung `gross_amount` (total pemesanan), `commission` (10%), `net_amount`
  - Atur status ke "pending" dengan `escrow_release_at` 24 jam dari penyelesaian
  - _Kebutuhan: 10.1, 10.2, 10.3, 15.4, 20.1, 20.2, 20.3, 20.4_

- [ ] 13.4 Implementasikan notifikasi permintaan ulasan
  - Kirim notifikasi FCM ke pengguna ketika pemesanan ditandai selesai
  - Tipe notifikasi: "review_requested" dengan booking_id
  - Deep link ke halaman pembuatan ulasan
  - _Kebutuhan: 15.6_

### 14. Penanganan No-Show

- [ ] 14.1 Buat UI pelaporan no-show
  - Buat `lib/features/booking/presentation/pages/no_show_report_page.dart`
  - Tambahkan tombol "Laporkan No-Show" di detail pemesanan untuk mitra
  - Implementasikan pengambilan foto dengan metadata GPS menggunakan `image_picker` dan `geolocator`
  - Tampilkan koordinat GPS dan jarak dari alamat pemesanan
  - _Kebutuhan: 14.1, 14.2_

- [ ] 14.2 Implementasikan validasi GPS
  - Hitung jarak antara GPS foto dan alamat pemesanan menggunakan rumus Haversine
  - Validasi jarak dalam 100 meter
  - Tampilkan pesan error jika validasi GPS gagal
  - _Kebutuhan: 14.2, 14.9_

- [ ] 14.3 Proses laporan no-show
  - Panggil Edge Function `process-no-show` dengan foto dan data GPS
  - Tangani sukses: perbarui status pemesanan ke "no_show", tampilkan konfirmasi
  - Tangani gagal: tampilkan pesan error dengan alasan
  - _Kebutuhan: 14.3, 14.4, 14.5, 14.6, 14.7, 14.8_

### 15. Sistem Pembatalan

- [ ] 15.1 Implementasikan alur pembatalan pengguna
  - Tambahkan tombol "Batalkan" di halaman detail pemesanan untuk pengguna
  - Tampilkan dialog kebijakan pembatalan dengan persentase pengembalian dana berdasarkan waktu
  - Wajibkan input alasan pembatalan
  - Panggil Edge Function `cancel-booking` dengan `cancelled_by: "user"`
  - Tampilkan jumlah pengembalian dana dan waktu pemrosesan
  - _Kebutuhan: 11.1, 11.2, 11.3, 11.4, 11.5, 11.6, 11.7, 11.8, 11.9, 11.10_

- [ ] 15.2 Implementasikan alur pembatalan mitra
  - Tambahkan tombol "Batalkan" di halaman detail pemesanan untuk mitra
  - Tampilkan peringatan tentang penalti: pengurangan 10 poin SP dan -0.1 rating
  - Wajibkan input alasan pembatalan
  - Panggil Edge Function `cancel-booking` dengan `cancelled_by: "mitra"`
  - Tampilkan konfirmasi pengembalian dana penuh ke pengguna
  - _Kebutuhan: 12.1, 12.2, 12.3, 12.4, 12.5, 12.6, 12.7, 12.8, 12.9_

- [ ] 15.3 Tangani notifikasi pembatalan
  - Kirim notifikasi FCM ke pengguna dan mitra saat pembatalan
  - Sertakan alasan pembatalan dan detail pengembalian dana dalam notifikasi
  - Buat record notifikasi di database
  - _Kebutuhan: 11.9, 12.9_


### 16. Sistem Penalti

- [ ] 16.1 Buat lapisan domain penalti
  - Definisikan entitas `lib/features/penalty/domain/entities/mitra_penalty.dart` dengan enum `PenaltyType`
  - Buat interface repository untuk operasi penalti
  - Implementasikan use case: `ApplyPenalty`, `GetMitraPenalties`, `AdjustServiceProtectionPoints`
  - _Kebutuhan: 22.1, 22.2, 22.3, 22.4, 22.5, 22.6, 22.7, 22.8, 22.9, 22.10_

- [ ] 16.2 Implementasikan logika perhitungan penalti
  - Buat `lib/features/penalty/domain/services/penalty_service.dart`
  - Definisikan poin penalti: cancel (10), no_show (15), dispute_lost (20)
  - Implementasikan pengurangan poin SP dengan minimum 0
  - Implementasikan toggle otomatis `is_available` ketika SP < 50
  - Implementasikan toggle otomatis `is_active` ketika SP = 0
  - _Kebutuhan: 22.3, 22.4, 22.5, 22.6, 22.7, 22.8_

- [ ] 16.3 Bangun UI pelacakan penalti untuk mitra
  - Buat `lib/features/penalty/presentation/pages/penalty_history_page.dart`
  - Tampilkan riwayat penalti dengan tipe, poin, tanggal, dan catatan
  - Tampilkan Service Protection Points saat ini dengan indikator status
  - Tampilkan pesan peringatan ketika SP < 70
  - Tampilkan peringatan kritis ketika SP < 50 (akun ditangguhkan)
  - _Kebutuhan: 22.1, 22.2, 22.7_

- [ ] 16.4 Buat UI manajemen penalti admin
  - Buat `lib/features/admin/presentation/pages/penalty_management_page.dart`
  - Izinkan admin menyesuaikan poin SP secara manual dengan justifikasi
  - Tampilkan riwayat penalti mitra
  - Tampilkan log audit untuk semua operasi penalti
  - _Kebutuhan: 22.9, 22.10_

### 17. Sistem Ulasan dan Rating

- [ ] 17.1 Buat lapisan domain ulasan
  - Definisikan entitas `lib/features/review/domain/entities/review.dart`
  - Buat interface repository untuk operasi ulasan
  - Implementasikan use case: `CreateReview`, `GetBookingReview`, `GetMitraReviews`, `HideReview`, `AddAdminResponse`
  - _Kebutuhan: 17.1, 17.2, 17.3, 17.4, 17.5, 17.6, 17.7, 17.8, 17.9_

- [ ] 17.2 Implementasikan lapisan data ulasan
  - Buat `lib/features/review/data/models/review_model.dart`
  - Implementasikan repository dengan kueri Supabase
  - Validasi rating antara 1-5 bintang
  - Cegah ulasan duplikat untuk pemesanan yang sama
  - Picu perhitungan ulang rating pada insert/update ulasan (via trigger database)
  - _Kebutuhan: 17.1, 17.2, 17.3, 17.4, 17.5, 17.6_

- [ ] 17.3 Bangun UI pembuatan ulasan
  - Buat `lib/features/review/presentation/pages/create_review_page.dart`
  - Implementasikan widget rating bintang (1-5 bintang)
  - Tambahkan input teks untuk komentar ulasan
  - Validasi status pemesanan adalah "selesai" sebelum mengizinkan ulasan
  - Tampilkan pesan sukses dan navigasi kembali setelah pengiriman
  - _Kebutuhan: 17.1, 17.2, 17.3, 17.4_

- [ ] 17.4 Buat UI tampilan ulasan
  - Buat `lib/features/review/presentation/widgets/review_list.dart`
  - Tampilkan ulasan dengan rating, komentar, nama pengguna, tanggal
  - Tampilkan respons admin jika ada
  - Urutkan ulasan berdasarkan created_at menurun
  - Filter untuk menampilkan hanya ulasan yang terlihat (`is_visible: true`)
  - _Kebutuhan: 17.7, 17.8, 17.9_

- [ ] 17.5 Implementasikan moderasi ulasan admin
  - Buat `lib/features/admin/presentation/pages/review_moderation_page.dart`
  - Izinkan admin menyembunyikan ulasan yang tidak pantas
  - Implementasikan fungsionalitas respons admin
  - Picu perhitungan ulang rating ketika visibilitas ulasan berubah
  - _Kebutuhan: 17.7, 17.8_

### 18. Sistem Chat Real-Time

- [ ] 18.1 Buat lapisan domain chat
  - Definisikan entitas `lib/features/chat/domain/entities/chat_room.dart`
  - Definisikan entitas `lib/features/chat/domain/entities/message.dart` dengan enum `MessageType`
  - Buat interface repository untuk operasi chat
  - Implementasikan use case: `GetChatRoom`, `SendMessage`, `MarkAsRead`, `WatchMessages`
  - _Kebutuhan: 18.1, 18.2, 18.3, 18.4, 18.5, 18.6, 18.7, 18.8, 18.9, 18.10_

- [ ] 18.2 Implementasikan lapisan data chat dengan Supabase Realtime
  - Buat `lib/features/chat/data/datasources/chat_realtime_datasource.dart`
  - Implementasikan langganan Supabase Realtime untuk tabel messages
  - Stream pesan difilter berdasarkan room_id dan diurutkan berdasarkan created_at
  - Implementasikan pengiriman pesan dengan validasi tipe (text, image, system)
  - Tangani unggah gambar ke bucket Storage `chat-images`
  - _Kebutuhan: 18.2, 18.3, 18.4, 18.8_

- [ ] 18.3 Bangun UI chat
  - Buat `lib/features/chat/presentation/pages/chat_page.dart`
  - Tampilkan pesan dalam daftar scrollable dengan alignment pengirim (kiri/kanan)
  - Implementasikan field input pesan dengan tombol kirim
  - Tambahkan tombol image picker untuk mengirim foto
  - Tampilkan indikator status baca
  - Auto-scroll ke bawah pada pesan baru
  - _Kebutuhan: 18.2, 18.3, 18.4, 18.5, 18.6, 18.9_

- [ ] 18.4 Implementasikan pembuatan chat room
  - Otomatis buat chat_room ketika pemesanan dibuat
  - Hubungkan chat_room ke booking_id, user_id, dan mitra_id
  - Perbarui timestamp `last_message_at` pada setiap pesan
  - _Kebutuhan: 18.1, 18.7_

- [ ] 18.5 Implementasikan notifikasi chat
  - Kirim notifikasi FCM ketika pesan diterima dan aplikasi di background
  - Tipe notifikasi: "message_received" dengan booking_id
  - Deep link ke halaman chat saat notifikasi diketuk
  - _Kebutuhan: 18.10_

### 19. Pendapatan dan Pencairan Mitra

- [ ] 19.1 Buat lapisan domain pendapatan
  - Definisikan entitas `lib/features/earnings/domain/entities/mitra_earnings.dart` dengan enum `EarningsStatus`
  - Buat interface repository untuk operasi pendapatan
  - Implementasikan use case: `GetMitraEarnings`, `GetEarningsSummary`, `RequestDisbursement`
  - _Kebutuhan: 20.1, 20.2, 20.3, 20.4, 20.5, 20.6, 20.7, 21.1, 21.2, 21.3, 21.4, 21.5, 21.6, 21.7, 21.8_

- [ ] 19.2 Implementasikan pelacakan pendapatan
  - Buat record pendapatan saat pemesanan selesai (ditangani di Edge Function)
  - Hitung gross_amount, commission (10%), net_amount
  - Atur status ke "pending" dengan escrow_release_at
  - Perbarui status ke "available" setelah pelepasan escrow
  - _Kebutuhan: 20.1, 20.2, 20.3, 20.4, 20.5_

- [ ] 19.3 Bangun UI dashboard pendapatan
  - Buat `lib/features/earnings/presentation/pages/earnings_dashboard_page.dart`
  - Tampilkan ringkasan pendapatan: total pending, available, disbursed
  - Tampilkan daftar pendapatan dengan detail pemesanan, jumlah, dan status
  - Filter pendapatan berdasarkan status (pending, available, disbursed)
  - Tampilkan timer hitung mundur escrow untuk pendapatan pending
  - _Kebutuhan: 20.6, 20.7_

- [ ] 19.4 Implementasikan konfigurasi rekening bank
  - Buat `lib/features/earnings/presentation/pages/bank_account_setup_page.dart`
  - Field input untuk bank_name, bank_account, bank_holder
  - Enkripsi bank_account menggunakan AES-256 sebelum menyimpan
  - Validasi format rekening bank
  - _Kebutuhan: 21.1, 21.6_

- [ ] 19.5 Buat alur permintaan pencairan
  - Tambahkan tombol "Tarik Dana" di dashboard pendapatan
  - Validasi rekening bank sudah dikonfigurasi
  - Validasi total pendapatan tersedia >= penarikan minimum (50000 IDR)
  - Panggil API pencairan Xendit via Edge Function
  - Perbarui status pendapatan ke "disbursed" saat sukses
  - Tampilkan pesan error jika pencairan gagal
  - _Kebutuhan: 21.1, 21.2, 21.3, 21.4, 21.5, 21.7, 21.8_

- [ ] 19.6 Buat Edge Function `process-disbursement`
  - Siapkan `supabase/functions/process-disbursement/index.ts`
  - Validasi mitra memiliki rekening bank yang dikonfigurasi
  - Validasi total pendapatan tersedia >= minimum
  - Dekripsi bank_account untuk panggilan API Xendit
  - Buat transaksi pencairan Xendit
  - Perbarui record mitra_earnings ke status "disbursed"
  - Catat timestamp `disbursed_at` dan `xendit_disbursement_id`
  - Kirim notifikasi FCM saat sukses
  - _Kebutuhan: 21.3, 21.4, 21.5, 21.6_

### 20. Sistem Notifikasi Push

- [ ] 20.1 Implementasikan lapisan domain notifikasi
  - Definisikan entitas `lib/features/notification/domain/entities/notification.dart` dengan enum `NotificationType`
  - Buat interface repository untuk operasi notifikasi
  - Implementasikan use case: `GetNotifications`, `MarkAsRead`, `SendNotification`
  - _Kebutuhan: 19.1, 19.2, 19.3, 19.4, 19.5, 19.6, 19.7, 19.8, 19.9_

- [ ] 20.2 Implementasikan manajemen token FCM
  - Simpan token FCM di `users.fcm_token` saat peluncuran aplikasi
  - Tangani refresh token dan perbarui database
  - Hapus token saat logout
  - _Kebutuhan: 19.1, 19.8_

- [ ] 20.3 Buat layanan pengiriman notifikasi
  - Buat `lib/core/services/notification_service.dart`
  - Implementasikan metode untuk setiap tipe notifikasi: booking_new, booking_confirmed, payment_success, booking_started, booking_completed, booking_cancelled, message_received, review_requested
  - Sertakan booking_id di payload data untuk deep linking
  - Buat record notifikasi di database
  - Kirim notifikasi push FCM
  - _Kebutuhan: 19.2, 19.3, 19.4_

- [ ] 20.4 Implementasikan penanganan notifikasi
  - Tangani notifikasi foreground dengan banner dalam aplikasi
  - Tangani notifikasi background dengan system tray
  - Tangani ketukan notifikasi dengan deep linking ke layar yang relevan
  - Navigasi berdasarkan tipe notifikasi
  - _Kebutuhan: 19.5_

- [ ] 20.5 Bangun UI pusat notifikasi
  - Buat `lib/features/notification/presentation/pages/notification_center_page.dart`
  - Tampilkan daftar notifikasi dengan ikon, judul, isi, timestamp
  - Tandai notifikasi sebagai dibaca saat dilihat
  - Kelompokkan notifikasi berdasarkan tanggal
  - Tampilkan badge jumlah belum dibaca
  - _Kebutuhan: 19.6_

- [ ] 20.6 Implementasikan notifikasi pengingat pemesanan
  - Buat pekerjaan terjadwal untuk mengirim pengingat 24 jam sebelum `scheduled_at`
  - Filter pemesanan dengan status "terjadwal"
  - Kirim notifikasi ke pengguna dan mitra
  - _Kebutuhan: 19.7_

### 21. Konfigurasi dan Pengaturan Sistem

- [ ] 21.1 Buat lapisan domain pengaturan
  - Definisikan entitas `lib/features/settings/domain/entities/system_setting.dart` dengan enum `ValueType`
  - Buat interface repository untuk operasi pengaturan
  - Implementasikan use case: `GetSetting`, `UpdateSetting`, `GetAllSettings`
  - _Kebutuhan: 25.1, 25.2, 25.3, 25.4, 25.5, 25.6, 25.7, 25.8_

- [ ] 21.2 Implementasikan lapisan data pengaturan
  - Buat repository dengan kueri Supabase
  - Parse nilai pengaturan sesuai value_type (string, number, boolean, json)
  - Filter pengaturan berdasarkan is_public untuk akses sisi klien
  - _Kebutuhan: 25.2, 25.4, 25.5, 25.8_

- [ ] 21.3 Bangun UI manajemen pengaturan admin
  - Buat `lib/features/admin/presentation/pages/settings_management_page.dart`
  - Tampilkan semua pengaturan dengan nilai saat ini
  - Implementasikan editing untuk setiap pengaturan dengan input sesuai tipe
  - Validasi nilai cocok dengan value_type sebelum menyimpan
  - Catat updated_at dan updated_by pada perubahan
  - _Kebutuhan: 25.1, 25.2, 25.3, 25.7_

- [ ] 21.4 Buat pengaturan sistem default
  - Masukkan pengaturan default: commission_rate (10.0), no_show_compensation_percentage (20.0), minimum_withdrawal_amount (50000), booking_advance_hours (2), escrow_hold_hours (24)
  - Atur value_type yang sesuai untuk setiap pengaturan
  - Konfigurasi flag is_public
  - _Kebutuhan: 25.6_


### 22. Dashboard Mitra

- [ ] 22.1 Buat UI dashboard mitra
  - Buat `lib/features/mitra/presentation/pages/mitra_dashboard_page.dart`
  - Tampilkan metrik kunci: total pemesanan, tingkat penyelesaian, rata-rata rating, poin SP
  - Tampilkan daftar pemesanan akan datang
  - Tampilkan ringkasan pendapatan (pending, available)
  - Tambahkan tombol aksi cepat: kelola layanan, lihat jadwal, pendapatan
  - _Kebutuhan: 2.2, 13.1_

- [ ] 22.2 Implementasikan manajemen pemesanan untuk mitra
  - Buat `lib/features/mitra/presentation/pages/mitra_bookings_page.dart`
  - Tampilkan pemesanan difilter berdasarkan status: akan datang, berlangsung, selesai
  - Tampilkan kartu pemesanan dengan info pengguna, layanan, jadwal, alamat
  - Implementasikan aksi pembaruan status: mulai, selesai, batalkan
  - _Kebutuhan: 13.1, 13.2, 13.3_

- [ ] 22.3 Bangun halaman profil mitra
  - Buat `lib/features/mitra/presentation/pages/mitra_profile_page.dart`
  - Tampilkan informasi mitra: nama, bio, rating, total order, tingkat penyelesaian, poin SP
  - Tampilkan status verifikasi dan badge
  - Tampilkan katalog layanan
  - Tampilkan ulasan dari pengguna
  - _Kebutuhan: 4.1, 17.9_

### 23. Dashboard dan Beranda Pengguna

- [ ] 23.1 Buat halaman beranda pengguna
  - Buat `lib/features/home/presentation/pages/home_page.dart`
  - Tampilkan kategori layanan dengan ikon
  - Tampilkan mitra unggulan terdekat
  - Implementasikan search bar untuk layanan
  - Tambahkan akses cepat ke pemesanan aktif
  - _Kebutuhan: 2.1, 23.1_

- [ ] 23.2 Bangun riwayat pemesanan pengguna
  - Buat `lib/features/home/presentation/pages/user_bookings_page.dart`
  - Tampilkan riwayat pemesanan dengan filter: akan datang, selesai, dibatalkan
  - Tampilkan kartu pemesanan dengan info mitra, layanan, status
  - Implementasikan navigasi ke detail pemesanan
  - _Kebutuhan: 8.1, 13.1_

- [ ] 23.3 Buat halaman detail mitra
  - Buat `lib/features/home/presentation/pages/mitra_detail_page.dart`
  - Tampilkan informasi profil mitra
  - Tampilkan katalog layanan dengan harga
  - Tampilkan ulasan dan rating
  - Tampilkan kalender ketersediaan
  - Tambahkan tombol "Pesan Sekarang"
  - _Kebutuhan: 5.1, 6.1, 17.9, 23.1_

### 24. Dashboard Admin

- [ ] 24.1 Buat UI dashboard admin
  - Buat `lib/features/admin/presentation/pages/admin_dashboard_page.dart`
  - Tampilkan metrik platform: total pengguna, total mitra, total pemesanan, total pendapatan
  - Tampilkan jumlah permintaan verifikasi tertunda
  - Tampilkan sengketa terbaru
  - Tampilkan indikator kesehatan sistem
  - _Kebutuhan: 2.3_

- [ ] 24.2 Implementasikan manajemen sengketa
  - Buat `lib/features/admin/presentation/pages/dispute_management_page.dart`
  - Tampilkan pemesanan yang disengketakan dengan penahanan escrow
  - Tampilkan detail sengketa: info pemesanan, keluhan pengguna, respons mitra
  - Implementasikan aksi resolusi: setujui pengguna, setujui mitra, pengembalian dana parsial
  - Perbarui `is_disputed` dan `dispute_resolved_at` saat resolusi
  - _Kebutuhan: 10.5, 10.6, 16.5_

- [ ] 24.3 Buat penampil log audit
  - Buat `lib/features/admin/presentation/pages/audit_log_page.dart`
  - Tampilkan log audit dengan filter: tipe entitas, aksi, rentang tanggal
  - Tampilkan detail log: timestamp, pengguna, aksi, entitas, perubahan
  - Implementasikan fungsionalitas pencarian
  - _Kebutuhan: 4.5, 16.6, 22.10, 25.7_

### 25. Pengujian dan Jaminan Kualitas

- [ ] 25.1 Tulis unit test untuk lapisan domain
  - Test use case dengan mock repository
  - Test logika bisnis dan validasi entitas
  - Test value object (BookingCode, ServiceProtectionPoints)
  - Capai >80% cakupan kode untuk lapisan domain
  - _Kebutuhan: Semua logika domain_

- [ ] 25.2 Tulis unit test untuk lapisan data
  - Test implementasi repository dengan mock datasource
  - Test serialisasi/deserialisasi JSON model
  - Test mapper data antara model dan entitas
  - Capai >80% cakupan kode untuk lapisan data
  - _Kebutuhan: Semua operasi data_

- [ ] 25.3 Tulis widget test untuk lapisan presentasi
  - Test komponen UI kunci: form login, pembuatan pemesanan, antarmuka chat
  - Test manajemen state dengan provider Riverpod
  - Test alur navigasi
  - Test penanganan error dan state loading
  - _Kebutuhan: Semua fitur UI_

- [ ] 25.4 Tulis integration test untuk alur kritis
  - Test alur pembuatan pemesanan end-to-end
  - Test alur pemrosesan pembayaran
  - Test alur pembatalan dengan perhitungan pengembalian dana
  - Test pengiriman dan penerimaan pesan chat
  - _Kebutuhan: 8.1-8.10, 9.1-9.10, 11.1-11.10, 18.1-18.10_

- [ ] 25.5 Test Edge Functions secara lokal
  - Siapkan lingkungan pengembangan lokal Supabase
  - Test fungsi `create-booking` dengan berbagai skenario
  - Test fungsi `cancel-booking` dengan waktu berbeda
  - Test fungsi `xendit-webhook` dengan payload mock
  - Test fungsi `release-escrow` dengan eksekusi terjadwal
  - _Kebutuhan: 8.10, 11.1-11.10, 12.1-12.9, 16.1-16.6_

### 26. Deployment dan DevOps

- [ ] 26.1 Konfigurasi lingkungan produksi Supabase
  - Siapkan proyek Supabase produksi
  - Jalankan migrasi database
  - Konfigurasi kebijakan RLS
  - Siapkan bucket Storage dengan kebijakan
  - Deploy Edge Functions ke produksi
  - _Kebutuhan: Semua fitur backend_

- [ ] 26.2 Konfigurasi kredensial produksi Xendit
  - Dapatkan kunci API produksi dari Xendit
  - Konfigurasi URL webhook mengarah ke Edge Functions produksi
  - Test alur pembayaran dalam mode produksi
  - Verifikasi validasi tanda tangan webhook
  - _Kebutuhan: 9.1-9.10, 21.3-21.6_

- [ ] 26.3 Siapkan Firebase Cloud Messaging
  - Buat proyek Firebase untuk produksi
  - Konfigurasi FCM untuk Android dan iOS
  - Unggah sertifikat APNs untuk notifikasi push iOS
  - Test notifikasi push pada perangkat fisik
  - _Kebutuhan: 19.1-19.9_

- [ ] 26.4 Build dan deploy aplikasi mobile
  - Konfigurasi penandatanganan aplikasi untuk Android (keystore)
  - Konfigurasi penandatanganan aplikasi untuk iOS (sertifikat, provisioning profiles)
  - Build release APK untuk Android
  - Build release IPA untuk iOS
  - Unggah ke Google Play Store (internal testing)
  - Unggah ke Apple App Store (TestFlight)
  - _Kebutuhan: Semua fitur mobile_

- [ ] 26.5 Siapkan pemantauan dan logging
  - Konfigurasi Sentry untuk pelacakan error
  - Siapkan logging dan pemantauan Supabase
  - Konfigurasi analitik (Firebase Analytics atau Mixpanel)
  - Siapkan pemantauan uptime untuk Edge Functions
  - Buat aturan alerting untuk error kritis
  - _Kebutuhan: Stabilitas produksi_

### 27. Dokumentasi dan Serah Terima

- [ ] 27.1 Buat dokumentasi API
  - Dokumentasikan semua endpoint Edge Function dengan skema request/response
  - Dokumentasikan skema database Supabase dengan ERD
  - Dokumentasikan kebijakan RLS dan aturan keamanan
  - Buat koleksi Postman untuk pengujian API
  - _Kebutuhan: Serah terima developer_

- [ ] 27.2 Tulis dokumentasi pengguna
  - Buat panduan pengguna untuk memesan layanan
  - Buat panduan mitra untuk mengelola layanan dan pemesanan
  - Buat panduan admin untuk verifikasi dan penyelesaian sengketa
  - Dokumentasikan kebijakan pembatalan dan proses pengembalian dana
  - _Kebutuhan: Onboarding pengguna_

- [ ] 27.3 Buat panduan deployment
  - Dokumentasikan setup dan konfigurasi Supabase
  - Dokumentasikan setup integrasi Xendit
  - Dokumentasikan setup Firebase untuk FCM
  - Dokumentasikan proses build dan deployment aplikasi mobile
  - Buat referensi variabel lingkungan
  - _Kebutuhan: Serah terima DevOps_

- [ ] 27.4 Checkpoint akhir - Pengujian komprehensif
  - Pastikan semua unit test lulus
  - Pastikan semua integration test lulus
  - Verifikasi semua Edge Functions bekerja di produksi
  - Test perjalanan pengguna lengkap: registrasi → pencarian → pemesanan → pembayaran → penyelesaian → ulasan
  - Test perjalanan mitra lengkap: registrasi → verifikasi → setup layanan → penerimaan pemesanan → penyelesaian → penarikan pendapatan
  - Test alur kerja admin: verifikasi, penyelesaian sengketa, manajemen penalti
  - Tanyakan pengguna jika ada pertanyaan atau masalah ditemukan

## Catatan

- Tugas diorganisir dalam urutan dependensi logis: infrastruktur → autentikasi → fitur inti → fitur lanjutan → pengujian → deployment
- Setiap tugas mereferensikan kebutuhan spesifik untuk ketertelusuran
- Edge Functions menangani logika bisnis kritis di sisi server untuk keamanan dan konsistensi
- Fitur real-time menggunakan langganan Supabase Realtime
- Pemrosesan pembayaran ditangani via Xendit dengan integrasi webhook
- Clean Architecture memastikan pemisahan tanggung jawab dan kemampuan pengujian
- Riverpod 2.x menyediakan manajemen state reaktif
- Pengujian terintegrasi sepanjang pengembangan, bukan sebagai fase terpisah
- Platform menggunakan arsitektur "1 Aplikasi, 2 Mode" dengan routing berbasis peran
- Semua operasi sensitif (pembuatan pemesanan, pembatalan, pelepasan escrow) ditangani di Edge Functions
- Sistem penalti menjaga akuntabilitas mitra melalui Service Protection Points
- Sistem escrow 24 jam melindungi pengguna sambil memastikan mitra dibayar untuk pekerjaan yang selesai
