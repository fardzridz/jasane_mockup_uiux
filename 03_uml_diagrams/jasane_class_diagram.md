# JASAne - Class Diagram (Flutter Clean Architecture)

## 📐 Architecture Overview

```
lib/
├── core/                    # Shared utilities & constants
├── features/                # Feature modules (Clean Architecture)
│   ├── auth/
│   ├── booking/
│   ├── mitra/
│   └── chat/
└── shared/                  # Shared widgets & providers
```

---

## 🏗️ Clean Architecture Layers

### **1. Presentation Layer** (UI + State Management)
- **Widgets** (Screens & Components)
- **Riverpod Providers** (State Management)
- **View Models** (Business Logic for UI)

### **2. Domain Layer** (Business Logic)
- **Entities** (Pure Dart models)
- **Use Cases** (Single responsibility actions)
- **Repository Interfaces** (Contracts)

### **3. Data Layer** (External Data Sources)
- **Models** (JSON serialization)
- **Repository Implementations**
- **Data Sources** (Supabase, Local Storage)

---

## 📦 Core Module Classes

### **core/constants/**

```dart
class AppColors {
  static const Color primary = Color(0xFF1D9E75);
  static const Color secondary = Color(0xFFBA7517);
  static const Color error = Color(0xFFB85450);
  static const Color warning = Color(0xFFD79B00);
}

class ApiEndpoints {
  static const String createBooking = '/functions/v1/create-booking';
  static const String confirmBooking = '/functions/v1/confirm-booking';
  static const String xenditWebhook = '/functions/v1/xendit-webhook';
  static const String payoutMitra = '/functions/v1/payout-mitra';
}

class AppRoutes {
  static const String splash = '/splash';
  static const String login = '/login';
  static const String home = '/home';
  static const String mitraDashboard = '/mitra/dashboard';
  static const String adminDashboard = '/admin/dashboard';
}
```

### **core/network/**

```dart
class SupabaseClient {
  static final SupabaseClient instance = SupabaseClient._internal();
  late final Supabase _supabase;
  
  factory SupabaseClient() => instance;
  
  SupabaseClient._internal();
  
  Future<void> initialize(String url, String anonKey) async {
    await Supabase.initialize(url: url, anonKey: anonKey);
    _supabase = Supabase.instance;
  }
  
  SupabaseClient get client => _supabase.client;
  User? get currentUser => _supabase.client.auth.currentUser;
}
```

### **core/errors/**

```dart
abstract class Failure {
  final String message;
  const Failure(this.message);
}

class ServerFailure extends Failure {
  const ServerFailure(String message) : super(message);
}

class NetworkFailure extends Failure {
  const NetworkFailure(String message) : super(message);
}

class AuthFailure extends Failure {
  const AuthFailure(String message) : super(message);
}
```

---

## 🔐 Auth Feature Classes

### **Domain Layer**

```dart
// Entity
class UserEntity {
  final String id;
  final String fullName;
  final String? phone;
  final String? avatarUrl;
  final UserRole role;
  final bool isActive;
  
  const UserEntity({
    required this.id,
    required this.fullName,
    this.phone,
    this.avatarUrl,
    required this.role,
    required this.isActive,
  });
}

enum UserRole { user, mitra, admin }

// Use Case
class LoginWithEmailUseCase {
  final AuthRepository repository;
  
  const LoginWithEmailUseCase(this.repository);
  
  Future<Either<Failure, UserEntity>> call(String email, String password) {
    return repository.loginWithEmail(email, password);
  }
}

// Repository Interface
abstract class AuthRepository {
  Future<Either<Failure, UserEntity>> loginWithEmail(String email, String password);
  Future<Either<Failure, UserEntity>> loginWithGoogle();
  Future<Either<Failure, void>> logout();
  Future<Either<Failure, UserEntity>> getCurrentUser();
}
```

### **Data Layer**

```dart
// Model (extends Entity)
class UserModel extends UserEntity {
  const UserModel({
    required String id,
    required String fullName,
    String? phone,
    String? avatarUrl,
    required UserRole role,
    required bool isActive,
  }) : super(
    id: id,
    fullName: fullName,
    phone: phone,
    avatarUrl: avatarUrl,
    role: role,
    isActive: isActive,
  );
  
  factory UserModel.fromJson(Map<String, dynamic> json) {
    return UserModel(
      id: json['id'],
      fullName: json['full_name'],
      phone: json['phone'],
      avatarUrl: json['avatar_url'],
      role: UserRole.values.byName(json['role']),
      isActive: json['is_active'] ?? true,
    );
  }
  
  Map<String, dynamic> toJson() {
    return {
      'id': id,
      'full_name': fullName,
      'phone': phone,
      'avatar_url': avatarUrl,
      'role': role.name,
      'is_active': isActive,
    };
  }
}

// Repository Implementation
class AuthRepositoryImpl implements AuthRepository {
  final AuthRemoteDataSource remoteDataSource;
  final AuthLocalDataSource localDataSource;
  
  const AuthRepositoryImpl({
    required this.remoteDataSource,
    required this.localDataSource,
  });
  
  @override
  Future<Either<Failure, UserEntity>> loginWithEmail(String email, String password) async {
    try {
      final user = await remoteDataSource.loginWithEmail(email, password);
      await localDataSource.cacheUser(user);
      return Right(user);
    } on ServerException catch (e) {
      return Left(ServerFailure(e.message));
    }
  }
  
  // ... other methods
}

// Data Source
abstract class AuthRemoteDataSource {
  Future<UserModel> loginWithEmail(String email, String password);
  Future<UserModel> loginWithGoogle();
  Future<void> logout();
}

class AuthRemoteDataSourceImpl implements AuthRemoteDataSource {
  final SupabaseClient supabase;
  
  const AuthRemoteDataSourceImpl(this.supabase);
  
  @override
  Future<UserModel> loginWithEmail(String email, String password) async {
    final response = await supabase.auth.signInWithPassword(
      email: email,
      password: password,
    );
    
    if (response.user == null) {
      throw ServerException('Login failed');
    }
    
    // Fetch user data from users table
    final userData = await supabase
        .from('users')
        .select()
        .eq('id', response.user!.id)
        .single();
    
    return UserModel.fromJson(userData);
  }
  
  // ... other methods
}
```

### **Presentation Layer**

```dart
// Riverpod Providers
final authRepositoryProvider = Provider<AuthRepository>((ref) {
  return AuthRepositoryImpl(
    remoteDataSource: ref.read(authRemoteDataSourceProvider),
    localDataSource: ref.read(authLocalDataSourceProvider),
  );
});

final currentUserProvider = StreamProvider<UserEntity?>((ref) {
  return ref.watch(authRepositoryProvider).watchAuthState();
});

final roleProvider = FutureProvider<UserRole?>((ref) async {
  final user = ref.watch(currentUserProvider).value;
  if (user == null) return null;
  return user.role;
});

// Screen
class LoginScreen extends ConsumerWidget {
  const LoginScreen({Key? key}) : super(key: key);
  
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final authState = ref.watch(authStateProvider);
    
    return Scaffold(
      body: authState.when(
        data: (user) => _buildLoginForm(context, ref),
        loading: () => const CircularProgressIndicator(),
        error: (error, stack) => Text('Error: $error'),
      ),
    );
  }
}
```

---

## 📅 Booking Feature Classes

### **Domain Layer**

```dart
// Entity
class BookingEntity {
  final String id;
  final String bookingCode;
  final String userId;
  final String mitraId;
  final BookingStatus status;
  final DateTime scheduledAt;
  final String address;
  final double subtotal;
  final double platformFee;
  final double totalAmount;
  final List<BookingItemEntity> items;
  
  const BookingEntity({
    required this.id,
    required this.bookingCode,
    required this.userId,
    required this.mitraId,
    required this.status,
    required this.scheduledAt,
    required this.address,
    required this.subtotal,
    required this.platformFee,
    required this.totalAmount,
    required this.items,
  });
}

enum BookingStatus {
  terjadwal,
  berlangsung,
  selesai,
  dibatalkanPelanggan,
  dibatalkanMitra,
  noShow,
}

class BookingItemEntity {
  final String serviceId;
  final String serviceName;
  final double price;
  final int quantity;
  final double subtotal;
  
  const BookingItemEntity({
    required this.serviceId,
    required this.serviceName,
    required this.price,
    required this.quantity,
    required this.subtotal,
  });
}

// Use Case
class CreateBookingUseCase {
  final BookingRepository repository;
  
  const CreateBookingUseCase(this.repository);
  
  Future<Either<Failure, BookingEntity>> call(CreateBookingParams params) {
    return repository.createBooking(params);
  }
}

class CreateBookingParams {
  final String mitraId;
  final List<String> serviceIds;
  final List<int> quantities;
  final DateTime scheduledAt;
  final String address;
  final String? notes;
  
  const CreateBookingParams({
    required this.mitraId,
    required this.serviceIds,
    required this.quantities,
    required this.scheduledAt,
    required this.address,
    this.notes,
  });
}

// Repository Interface
abstract class BookingRepository {
  Future<Either<Failure, BookingEntity>> createBooking(CreateBookingParams params);
  Future<Either<Failure, List<BookingEntity>>> getUserBookings(String userId);
  Future<Either<Failure, BookingEntity>> getBookingById(String bookingId);
  Future<Either<Failure, void>> cancelBooking(String bookingId, String reason);
  Stream<BookingEntity> watchBooking(String bookingId);
}
```

### **Data Layer**

```dart
// Model
class BookingModel extends BookingEntity {
  const BookingModel({
    required String id,
    required String bookingCode,
    required String userId,
    required String mitraId,
    required BookingStatus status,
    required DateTime scheduledAt,
    required String address,
    required double subtotal,
    required double platformFee,
    required double totalAmount,
    required List<BookingItemEntity> items,
  }) : super(
    id: id,
    bookingCode: bookingCode,
    userId: userId,
    mitraId: mitraId,
    status: status,
    scheduledAt: scheduledAt,
    address: address,
    subtotal: subtotal,
    platformFee: platformFee,
    totalAmount: totalAmount,
    items: items,
  );
  
  factory BookingModel.fromJson(Map<String, dynamic> json) {
    return BookingModel(
      id: json['id'],
      bookingCode: json['booking_code'],
      userId: json['user_id'],
      mitraId: json['mitra_id'],
      status: BookingStatus.values.byName(json['status']),
      scheduledAt: DateTime.parse(json['scheduled_at']),
      address: json['address'],
      subtotal: (json['subtotal'] as num).toDouble(),
      platformFee: (json['platform_fee'] as num).toDouble(),
      totalAmount: (json['total_amount'] as num).toDouble(),
      items: (json['booking_items'] as List)
          .map((item) => BookingItemModel.fromJson(item))
          .toList(),
    );
  }
}

// Repository Implementation
class BookingRepositoryImpl implements BookingRepository {
  final SupabaseClient supabase;
  
  const BookingRepositoryImpl(this.supabase);
  
  @override
  Future<Either<Failure, BookingEntity>> createBooking(CreateBookingParams params) async {
    try {
      final response = await supabase.functions.invoke(
        'create-booking',
        body: {
          'mitra_id': params.mitraId,
          'service_ids': params.serviceIds,
          'quantities': params.quantities,
          'scheduled_at': params.scheduledAt.toIso8601String(),
          'address': params.address,
          'notes': params.notes,
        },
      );
      
      if (response.status != 200) {
        throw ServerException(response.data['error']);
      }
      
      final bookingId = response.data['booking_id'];
      final booking = await getBookingById(bookingId);
      
      return booking;
    } on FunctionException catch (e) {
      return Left(ServerFailure(e.details.toString()));
    }
  }
  
  @override
  Stream<BookingEntity> watchBooking(String bookingId) {
    return supabase
        .from('bookings')
        .stream(primaryKey: ['id'])
        .eq('id', bookingId)
        .map((data) => BookingModel.fromJson(data.first));
  }
}
```

### **Presentation Layer**

```dart
// Providers
final bookingRepositoryProvider = Provider<BookingRepository>((ref) {
  return BookingRepositoryImpl(ref.read(supabaseClientProvider));
});

final createBookingProvider = StateNotifierProvider<CreateBookingNotifier, AsyncValue<BookingEntity?>>((ref) {
  return CreateBookingNotifier(ref.read(bookingRepositoryProvider));
});

final userBookingsProvider = FutureProvider.family<List<BookingEntity>, String>((ref, userId) async {
  final result = await ref.read(bookingRepositoryProvider).getUserBookings(userId);
  return result.fold(
    (failure) => throw Exception(failure.message),
    (bookings) => bookings,
  );
});

final bookingStreamProvider = StreamProvider.family<BookingEntity, String>((ref, bookingId) {
  return ref.read(bookingRepositoryProvider).watchBooking(bookingId);
});

// State Notifier
class CreateBookingNotifier extends StateNotifier<AsyncValue<BookingEntity?>> {
  final BookingRepository _repository;
  
  CreateBookingNotifier(this._repository) : super(const AsyncValue.data(null));
  
  Future<void> createBooking(CreateBookingParams params) async {
    state = const AsyncValue.loading();
    
    final result = await _repository.createBooking(params);
    
    state = result.fold(
      (failure) => AsyncValue.error(failure.message, StackTrace.current),
      (booking) => AsyncValue.data(booking),
    );
  }
}

// Screen
class BookingScreen extends ConsumerWidget {
  const BookingScreen({Key? key}) : super(key: key);
  
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final bookingState = ref.watch(createBookingProvider);
    
    return Scaffold(
      appBar: AppBar(title: const Text('Buat Booking')),
      body: bookingState.when(
        data: (booking) {
          if (booking != null) {
            // Navigate to payment
            return PaymentScreen(booking: booking);
          }
          return _buildBookingForm(context, ref);
        },
        loading: () => const Center(child: CircularProgressIndicator()),
        error: (error, stack) => ErrorWidget(error: error.toString()),
      ),
    );
  }
  
  Widget _buildBookingForm(BuildContext context, WidgetRef ref) {
    return Column(
      children: [
        // Form fields...
        ElevatedButton(
          onPressed: () {
            final params = CreateBookingParams(/* ... */);
            ref.read(createBookingProvider.notifier).createBooking(params);
          },
          child: const Text('Bayar'),
        ),
      ],
    );
  }
}
```

---

## 👷 Mitra Feature Classes

### **Domain Layer**

```dart
// Entity
class MitraProfileEntity {
  final String id;
  final String userId;
  final String? bio;
  final bool isVerified;
  final double ratingAvg;
  final int totalOrders;
  final double completionRate;
  final int serviceProtectionPoints;
  final double? latitude;
  final double? longitude;
  final int serviceRadius;
  final bool isAvailable;
  
  const MitraProfileEntity({
    required this.id,
    required this.userId,
    this.bio,
    required this.isVerified,
    required this.ratingAvg,
    required this.totalOrders,
    required this.completionRate,
    required this.serviceProtectionPoints,
    this.latitude,
    this.longitude,
    required this.serviceRadius,
    required this.isAvailable,
  });
}

class MitraServiceEntity {
  final String id;
  final String mitraId;
  final String categoryId;
  final String name;
  final String? description;
  final double price;
  final PriceUnit priceUnit;
  final int? durationMinutes;
  final bool isActive;
  
  const MitraServiceEntity({
    required this.id,
    required this.mitraId,
    required this.categoryId,
    required this.name,
    this.description,
    required this.price,
    required this.priceUnit,
    this.durationMinutes,
    required this.isActive,
  });
}

enum PriceUnit { perJob, perHour, perUnit, estimate }

// Use Case
class GetNearbyMitrasUseCase {
  final MitraRepository repository;
  
  const GetNearbyMitrasUseCase(this.repository);
  
  Future<Either<Failure, List<MitraProfileEntity>>> call(
    double latitude,
    double longitude,
    int radiusKm,
  ) {
    return repository.getNearbyMitras(latitude, longitude, radiusKm);
  }
}
```

---

## 💬 Chat Feature Classes

### **Domain Layer**

```dart
// Entity
class ChatRoomEntity {
  final String id;
  final String bookingId;
  final String userId;
  final String mitraId;
  final bool isActive;
  final DateTime? lastMessageAt;
  
  const ChatRoomEntity({
    required this.id,
    required this.bookingId,
    required this.userId,
    required this.mitraId,
    required this.isActive,
    this.lastMessageAt,
  });
}

class MessageEntity {
  final String id;
  final String roomId;
  final String senderId;
  final String? content;
  final MessageType type;
  final String? imageUrl;
  final bool isRead;
  final DateTime createdAt;
  
  const MessageEntity({
    required this.id,
    required this.roomId,
    required this.senderId,
    this.content,
    required this.type,
    this.imageUrl,
    required this.isRead,
    required this.createdAt,
  });
}

enum MessageType { text, image, system }

// Repository Interface
abstract class ChatRepository {
  Stream<List<MessageEntity>> watchMessages(String roomId);
  Future<Either<Failure, void>> sendMessage(String roomId, String content, MessageType type);
  Future<Either<Failure, void>> markAsRead(String messageId);
}
```

---

## 🎨 Shared Widgets

```dart
// Custom Button
class PrimaryButton extends StatelessWidget {
  final String text;
  final VoidCallback? onPressed;
  final bool isLoading;
  
  const PrimaryButton({
    Key? key,
    required this.text,
    this.onPressed,
    this.isLoading = false,
  }) : super(key: key);
  
  @override
  Widget build(BuildContext context) {
    return ElevatedButton(
      onPressed: isLoading ? null : onPressed,
      style: ElevatedButton.styleFrom(
        backgroundColor: AppColors.primary,
        minimumSize: const Size(double.infinity, 50),
      ),
      child: isLoading
          ? const CircularProgressIndicator(color: Colors.white)
          : Text(text),
    );
  }
}

// Mitra Card
class MitraCard extends StatelessWidget {
  final MitraProfileEntity mitra;
  final VoidCallback onTap;
  
  const MitraCard({
    Key? key,
    required this.mitra,
    required this.onTap,
  }) : super(key: key);
  
  @override
  Widget build(BuildContext context) {
    return Card(
      child: ListTile(
        leading: CircleAvatar(
          backgroundImage: NetworkImage(mitra.avatarUrl ?? ''),
        ),
        title: Text(mitra.name),
        subtitle: Row(
          children: [
            const Icon(Icons.star, size: 16, color: Colors.amber),
            Text('${mitra.ratingAvg} (${mitra.totalOrders} orders)'),
          ],
        ),
        trailing: mitra.isVerified
            ? const Icon(Icons.verified, color: Colors.blue)
            : null,
        onTap: onTap,
      ),
    );
  }
}
```

---

## 🔄 Routing (GoRouter)

```dart
final routerProvider = Provider<GoRouter>((ref) {
  final role = ref.watch(roleProvider);
  
  return GoRouter(
    initialLocation: '/splash',
    redirect: (context, state) {
      final isLoggedIn = ref.read(currentUserProvider).value != null;
      
      if (!isLoggedIn && state.location != '/login') {
        return '/login';
      }
      
      if (isLoggedIn && state.location == '/splash') {
        return role.when(
          data: (r) {
            switch (r) {
              case UserRole.mitra:
                return '/mitra/dashboard';
              case UserRole.admin:
                return '/admin/dashboard';
              case UserRole.user:
              default:
                return '/home';
            }
          },
          loading: () => '/splash',
          error: (_, __) => '/login',
        );
      }
      
      return null;
    },
    routes: [
      GoRoute(
        path: '/splash',
        builder: (context, state) => const SplashScreen(),
      ),
      GoRoute(
        path: '/login',
        builder: (context, state) => const LoginScreen(),
      ),
      GoRoute(
        path: '/home',
        builder: (context, state) => const HomeScreen(),
      ),
      GoRoute(
        path: '/mitra/dashboard',
        builder: (context, state) => const MitraDashboardScreen(),
      ),
      GoRoute(
        path: '/admin/dashboard',
        builder: (context, state) => const AdminDashboardScreen(),
      ),
    ],
  );
});
```

---

## 📊 Class Diagram Summary

### **Key Patterns Used:**

✅ **Clean Architecture** (Presentation → Domain → Data)  
✅ **Repository Pattern** (Abstract interfaces + Implementations)  
✅ **Use Case Pattern** (Single responsibility per action)  
✅ **Provider Pattern** (Riverpod for state management)  
✅ **Entity-Model Separation** (Domain entities vs Data models)  
✅ **Either Pattern** (Functional error handling with dartz)  
✅ **Stream Pattern** (Real-time updates via Supabase Realtime)  

### **Dependencies:**

```yaml
dependencies:
  flutter_riverpod: ^2.x.x
  supabase_flutter: ^2.x.x
  go_router: ^13.x.x
  dartz: ^0.10.x
  equatable: ^2.x.x
  freezed_annotation: ^2.x.x
  json_annotation: ^4.x.x
```

---

Ini adalah struktur class yang konsisten dengan Clean Architecture dan best practices Flutter bang! 🚀
