# Staff App - Quick Reference Guide

## 🚀 Getting Started

### Prerequisites
- Flutter 3.35.6 or higher
- Backend server running on `localhost:5001`
- Firebase project configured
- iOS/Android development environment

### Installation
```bash
cd "/Volumes/PERSONAL/Al Marya Rostery APP/al_marya_staff_app"
flutter pub get
flutter run
```

---

## 📁 Project Structure

```
lib/
├── core/                    # Core functionality (services, utilities)
│   ├── api/                # API clients and network layer
│   ├── auth/               # Authentication services
│   ├── constants/          # App-wide constants
│   └── utils/              # Utility functions
│
├── features/               # Feature-based modules
│   ├── auth/              # Authentication feature
│   ├── orders/            # Orders management feature
│   └── profile/           # Profile management feature
│
├── models/                # Data models
│   ├── staff.dart
│   └── order.dart
│
├── shared/                # Shared components
│   └── widgets/          # Reusable UI widgets
│
└── main.dart             # App entry point
```

---

## 🔑 Key Components

### Authentication Flow

```dart
// 1. Initialize auth on app start
await AuthService().initializeAuth();

// 2. Login
final staff = await AuthService().login(
  'staff@example.com',
  'password123',
);

// 3. Check login status
final isLoggedIn = await AuthService().isLoggedIn();

// 4. Get current staff
final staff = await AuthService().getCurrentStaff();

// 5. Logout
await AuthService().logout();
```

### API Calls

```dart
// Initialize API service
final staffApi = StaffApiService();

// Get orders
final orders = await staffApi.getOrders(
  status: 'pending',
  limit: 20,
);

// Accept order
final order = await staffApi.acceptOrder(orderId);

// Start preparation
final order = await staffApi.startPreparation(orderId);

// Mark ready (triggers driver notification)
final order = await staffApi.markOrderReady(orderId);

// Complete workflow
final order = await staffApi.processOrder(orderId);
```

### Local Storage

```dart
// Save token
await TokenStorage.saveToken(jwtToken);

// Get token
final token = await TokenStorage.getToken();

// Save staff data
await TokenStorage.saveStaffData(staff);

// Get staff data
final staff = await TokenStorage.getStaffData();

// Clear all data (logout)
await TokenStorage.clearAll();

// Check if logged in
final isLoggedIn = await TokenStorage.isLoggedIn();
```

---

## 📦 Models

### Staff Model

```dart
class Staff {
  final String id;
  final String firebaseUid;
  final String name;
  final String email;
  final String phone;
  final String role;              // barista, manager, cashier
  final String status;            // active, on_break, inactive
  final String? fcmToken;
  final List<String> assignedOrders;
  final StaffStats stats;
  final DateTime createdAt;
  final DateTime updatedAt;
}

// Usage
final staff = Staff.fromJson(jsonData);
final json = staff.toJson();
final updated = staff.copyWith(status: 'on_break');
```

### Order Model

```dart
class Order {
  final String id;
  final String orderNumber;
  final String status;
  final String deliveryMethod;
  final double totalAmount;
  final List<OrderItem> items;
  final Customer? user;
  final String? assignedStaff;
  final String? assignedDriver;
  final StatusTimestamps statusTimestamps;
  
  // Helper getters
  bool get isAssignedToStaff;
  bool get isPreparing;
  bool get isReady;
  bool get isPending;
}

// Usage
final order = Order.fromJson(jsonData);
if (order.isPending) {
  await staffApi.acceptOrder(order.id);
}
```

---

## 🌐 API Endpoints

### Base URL
```dart
http://localhost:5001
```

### Authentication
```
POST   /api/staff/login           # Login with Firebase token
POST   /api/staff/register        # Register new staff
PUT    /api/staff/fcm-token       # Update FCM token
```

### Profile
```
GET    /api/staff/profile         # Get staff profile
PUT    /api/staff/status          # Update status (active/break/inactive)
GET    /api/staff/stats           # Get staff statistics
```

### Orders
```
GET    /api/staff/orders          # List orders (with filters)
GET    /api/staff/orders/:id      # Get order details
POST   /api/staff/orders/:id/accept      # Accept order
POST   /api/staff/orders/:id/start       # Start preparation
POST   /api/staff/orders/:id/ready       # Mark ready (notify drivers)
```

---

## 🔧 Constants

### Order Status
```dart
AppConstants.statusPending       // 'pending'
AppConstants.statusConfirmed     // 'confirmed'
AppConstants.statusPreparing     // 'preparing'
AppConstants.statusReady         // 'ready'
AppConstants.statusPickedUp      // 'picked_up'
AppConstants.statusDelivered     // 'delivered'
AppConstants.statusCancelled     // 'cancelled'
```

### Staff Status
```dart
AppConstants.staffStatusActive       // 'active'
AppConstants.staffStatusOnBreak      // 'on_break'
AppConstants.staffStatusInactive     // 'inactive'
```

### Staff Roles
```dart
AppConstants.roleBarista         // 'barista'
AppConstants.roleManager         // 'manager'
AppConstants.roleCashier         // 'cashier'
```

### Storage Keys
```dart
AppConstants.tokenKey            // 'auth_token'
AppConstants.staffIdKey          // 'staff_id'
AppConstants.staffEmailKey       // 'staff_email'
AppConstants.staffNameKey        // 'staff_name'
AppConstants.staffRoleKey        // 'staff_role'
AppConstants.fcmTokenKey         // 'fcm_token'
```

---

## 🎨 UI Constants

```dart
AppConstants.defaultPadding      // 16.0
AppConstants.smallPadding        // 8.0
AppConstants.largePadding        // 24.0
AppConstants.borderRadius        // 12.0
```

---

## 🔔 FCM Notifications

### Setup (TODO)
```dart
// 1. Initialize Firebase Messaging
final messaging = FirebaseMessaging.instance;

// 2. Request permissions
await messaging.requestPermission();

// 3. Get FCM token
final fcmToken = await messaging.getToken();

// 4. Send to backend
await AuthService().updateFcmToken(fcmToken);

// 5. Listen for messages
FirebaseMessaging.onMessage.listen((RemoteMessage message) {
  // Handle foreground message
});

// 6. Handle background messages
FirebaseMessaging.onBackgroundMessage(_firebaseMessagingBackgroundHandler);
```

---

## 🧪 Testing

### Run Tests
```bash
flutter test
```

### Run Specific Test
```bash
flutter test test/models/staff_test.dart
```

### Check for Issues
```bash
flutter analyze
```

### Format Code
```bash
flutter format .
```

---

## 🐛 Error Handling

### API Errors
```dart
try {
  final orders = await staffApi.getOrders();
} on ApiException catch (e) {
  print('API Error: ${e.message}');
} on UnauthorizedException catch (e) {
  // Token expired or invalid
  await AuthService().logout();
  // Navigate to login
}
```

### Auth Errors
```dart
try {
  await AuthService().login(email, password);
} on AuthException catch (e) {
  // Handle auth-specific errors
  print('Auth Error: ${e.message}');
  // Show error to user
}
```

---

## 📝 Common Tasks

### Add New Order Status
1. Add constant to `app_constants.dart`
2. Update backend if needed
3. Update UI to handle new status

### Add New Staff Role
1. Add constant to `app_constants.dart`
2. Update backend if needed
3. Update UI role selection

### Add New API Endpoint
1. Add to `api_endpoints.dart`
2. Add method to `staff_api_service.dart`
3. Use in feature providers

### Update Staff Model
1. Modify `models/staff.dart`
2. Update `fromJson()` and `toJson()`
3. Update backend model to match
4. Update tests

---

## 🔍 Debugging

### Print API Responses
```dart
// In api_client.dart, add debug print:
print('Response: ${response.body}');
```

### Check Stored Data
```dart
await TokenStorage.printAllData();
```

### Check Auth State
```dart
final isLoggedIn = await AuthService().isLoggedIn();
final staff = await TokenStorage.getStaffData();
print('Logged in: $isLoggedIn, Staff: ${staff?.name}');
```

### Backend Health Check
```dart
final apiClient = ApiClient();
final isHealthy = await apiClient.healthCheck();
print('Backend healthy: $isHealthy');
```

---

## 🚨 Common Issues & Solutions

### Issue: "No internet connection"
**Solution:** Check if backend server is running on `localhost:5001`
```bash
curl http://localhost:5001/api/health
```

### Issue: "Unauthorized" (401 error)
**Solution:** Token expired or invalid
```dart
await AuthService().logout();
// Redirect to login
```

### Issue: Firebase not initialized
**Solution:** Ensure Firebase is initialized in `main.dart`
```dart
await Firebase.initializeApp();
```

### Issue: SharedPreferences not initialized
**Solution:** Initialize TokenStorage
```dart
await TokenStorage.init();
```

---

## 📊 Backend Testing

### Test Staff Login
```bash
curl -X POST http://localhost:5001/api/staff/login \
  -H "Content-Type: application/json" \
  -d '{"firebaseToken": "YOUR_FIREBASE_TOKEN"}'
```

### Test Orders Endpoint (with JWT)
```bash
curl -X GET http://localhost:5001/api/staff/orders \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

---

## 🎯 Next Steps (Development Roadmap)

### Phase 1: Authentication UI ⏳
- [ ] Login screen
- [ ] Auth provider
- [ ] Main app initialization
- [ ] Splash screen
- [ ] Error handling UI

### Phase 2: Orders Feature ⏳
- [ ] Orders list screen
- [ ] Order details screen
- [ ] Order status cards
- [ ] Accept/Start/Ready buttons
- [ ] Real-time updates
- [ ] Pull to refresh

### Phase 3: Profile Feature ⏳
- [ ] Profile screen
- [ ] Status toggle (active/break)
- [ ] Statistics display
- [ ] Logout button
- [ ] Theme settings

### Phase 4: Notifications ⏳
- [ ] FCM setup
- [ ] Foreground notifications
- [ ] Background notifications
- [ ] Notification actions
- [ ] Sound/vibration

### Phase 5: Polish ⏳
- [ ] Loading states
- [ ] Empty states
- [ ] Error states
- [ ] Animations
- [ ] Dark mode
- [ ] Localization (if needed)

### Phase 6: Testing ⏳
- [ ] Unit tests
- [ ] Widget tests
- [ ] Integration tests
- [ ] E2E tests

---

## 📚 Dependencies Reference

```yaml
firebase_core: ^3.8.1              # Firebase initialization
firebase_auth: ^5.3.3              # Firebase authentication
firebase_messaging: ^15.1.5        # Push notifications
provider: ^6.1.2                   # State management
http: ^1.2.2                       # HTTP client
shared_preferences: ^2.3.3         # Local storage
flutter_local_notifications: ^18.0.1  # Local notifications
intl: ^0.19.0                      # Internationalization
timeago: ^3.7.0                    # Relative time formatting
```

---

## 🔗 Useful Links

- [Backend Documentation](../backend/README.md)
- [Phase 6.1 Complete Report](../backend/PHASE_6.1_COMPLETE.md)
- [Phase 6.2 Architecture](./ARCHITECTURE_DIAGRAM.md)
- [Flutter Documentation](https://docs.flutter.dev/)
- [Provider Package](https://pub.dev/packages/provider)
- [Firebase Flutter](https://firebase.google.com/docs/flutter/setup)

---

## 👥 Contact & Support

For issues or questions:
1. Check backend logs: `backend/server.log`
2. Check Flutter logs: `flutter logs`
3. Review test scripts: `backend/test-staff-endpoints.sh`
4. Review backend routes: `backend/routes/staff.js`

---

**Last Updated:** Current Session  
**Flutter Version:** 3.35.6  
**Backend API Version:** Phase 6.1 Complete  
**Status:** Core infrastructure complete, ready for UI development
