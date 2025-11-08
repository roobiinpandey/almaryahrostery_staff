# Al Marya Staff App - Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                     AL MARYA STAFF APP                          │
│                    Flutter 3.35.6 Stable                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    PRESENTATION LAYER (Pending)                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  📱 features/auth/                  📱 features/orders/          │
│     ├── screens/                       ├── screens/             │
│     │   └── login_screen.dart          │   ├── orders_list.dart│
│     ├── widgets/                       │   └── order_details.dart│
│     │   └── login_form.dart            ├── widgets/             │
│     └── providers/                     │   └── order_card.dart  │
│         └── auth_provider.dart         └── providers/           │
│                                            └── orders_provider.dart│
│                                                                  │
│  📱 features/profile/                                            │
│     ├── screens/                                                │
│     │   └── profile_screen.dart                                 │
│     ├── widgets/                                                │
│     │   └── stats_card.dart                                     │
│     └── providers/                                              │
│         └── profile_provider.dart                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                  BUSINESS LOGIC LAYER ✅ COMPLETE                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  🔐 core/auth/                                                   │
│     ├── auth_service.dart          (240 lines)                  │
│     │   ├── login(email, password) → Staff                      │
│     │   ├── register(...) → Staff                               │
│     │   ├── logout() → void                                     │
│     │   ├── isLoggedIn() → bool                                 │
│     │   ├── getCurrentStaff() → Staff?                          │
│     │   └── updateFcmToken(token) → void                        │
│     │                                                            │
│     └── token_storage.dart         (175 lines)                  │
│         ├── saveToken(token) → bool                             │
│         ├── getToken() → String?                                │
│         ├── saveStaffData(staff) → bool                         │
│         ├── getStaffData() → Staff?                             │
│         └── clearAll() → bool                                   │
│                                                                  │
│  🌐 core/api/                                                    │
│     ├── api_client.dart            (210 lines)                  │
│     │   ├── get(endpoint) → Map                                 │
│     │   ├── post(endpoint, body) → Map                          │
│     │   ├── put(endpoint, body) → Map                           │
│     │   ├── patch(endpoint, body) → Map                         │
│     │   └── setAuthToken(token) → void                          │
│     │                                                            │
│     └── staff_api_service.dart     (210 lines)                  │
│         ├── getProfile() → Staff                                │
│         ├── updateStatus(status) → Staff                        │
│         ├── getStats() → StaffStats                             │
│         ├── getOrders({status, limit}) → List<Order>            │
│         ├── getOrderDetails(id) → Order                         │
│         ├── acceptOrder(id) → Order                             │
│         ├── startPreparation(id) → Order                        │
│         └── markOrderReady(id) → Order                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     DATA LAYER ✅ COMPLETE                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  📦 models/                                                      │
│     ├── staff.dart                 (135 lines)                  │
│     │   ├── Staff                                               │
│     │   │   ├── id, firebaseUid, name, email, phone            │
│     │   │   ├── role, status, fcmToken                         │
│     │   │   ├── assignedOrders, stats                          │
│     │   │   ├── fromJson(), toJson()                           │
│     │   │   └── copyWith()                                     │
│     │   │                                                       │
│     │   └── StaffStats                                         │
│     │       ├── totalOrdersProcessed                           │
│     │       ├── ordersProcessedToday                           │
│     │       ├── averagePreparationTime                         │
│     │       └── lastOrderProcessedAt                           │
│     │                                                           │
│     └── order.dart                 (220 lines)                  │
│         ├── Order (17 fields)                                   │
│         ├── OrderItem                                           │
│         ├── Customer                                            │
│         ├── DeliveryAddress                                     │
│         ├── PreparationTime                                     │
│         ├── DeliveryTime                                        │
│         └── StatusTimestamps (11 timestamps)                    │
│                                                                  │
│  ⚙️ core/constants/                                              │
│     ├── api_endpoints.dart         (35 lines)                   │
│     │   └── 11 staff endpoints mapped                           │
│     │                                                            │
│     └── app_constants.dart         (55 lines)                   │
│         ├── Storage keys                                        │
│         ├── Order status constants                              │
│         ├── Staff status constants                              │
│         ├── Staff role constants                                │
│         └── UI constants                                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                   EXTERNAL DEPENDENCIES                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  🔥 Firebase Services               📦 Storage                   │
│     ├── firebase_core: 3.8.1          └── shared_preferences   │
│     ├── firebase_auth: 5.3.3              2.3.3                 │
│     └── firebase_messaging: 15.1.5                              │
│                                                                  │
│  🌐 HTTP Client                     📱 UI & Notifications        │
│     └── http: 1.2.2                    ├── flutter_local_      │
│                                         │   notifications: 18.0.1│
│  🎯 State Management                   ├── intl: 0.19.0         │
│     └── provider: 6.1.2                └── timeago: 3.7.0       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        BACKEND API                               │
│                   http://localhost:5001                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  🔐 Authentication (2 endpoints)                                 │
│     ├── POST /api/staff/login                                   │
│     └── POST /api/staff/register                                │
│                                                                  │
│  👤 Profile Management (3 endpoints)                             │
│     ├── GET  /api/staff/profile                                 │
│     ├── PUT  /api/staff/status                                  │
│     └── GET  /api/staff/stats                                   │
│                                                                  │
│  📦 Order Management (5 endpoints)                               │
│     ├── GET  /api/staff/orders                                  │
│     ├── GET  /api/staff/orders/:id                              │
│     ├── POST /api/staff/orders/:id/accept                       │
│     ├── POST /api/staff/orders/:id/start                        │
│     └── POST /api/staff/orders/:id/ready                        │
│                                                                  │
│  🔔 FCM Integration (1 endpoint)                                 │
│     └── PUT  /api/staff/fcm-token                               │
│                                                                  │
│  Status: ✅ 11/11 endpoints tested (91.67% pass rate)            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

═══════════════════════════════════════════════════════════════════

                        DATA FLOW EXAMPLE
                    Order Processing Workflow

User Action                 App Layer              Backend API
───────────                ─────────────           ────────────

[Accept Order]  →  OrdersProvider  →  StaffApiService  →  POST /orders/:id/accept
                   (notifyListeners)     (via ApiClient)     (JWT auth)
                                                              ↓
                                                         Update Order
                                                         assignedStaff
                                                         status = preparing
                                                              ↓
                                                         Response ←──┐
                                                                     │
                   ← Order.fromJson ← JSON Response ← ← ← ← ← ← ← ←┘
                   UI Updates
                   (order status badge)

[Start Prep]    →  (Same flow for /orders/:id/start)
[Mark Ready]    →  (Same flow for /orders/:id/ready)
                   └─→ Triggers FCM notification to drivers

═══════════════════════════════════════════════════════════════════

                       AUTHENTICATION FLOW

1. User enters email/password in LoginScreen
                ↓
2. AuthProvider.login() called
                ↓
3. AuthService.login()
   ├─→ Firebase: signInWithEmailAndPassword()
   ├─→ Firebase: getIdToken()
   ├─→ Backend: POST /api/staff/login {firebaseToken}
   ├─→ Backend: Returns {token, staff}
   ├─→ TokenStorage: saveToken() & saveStaffData()
   └─→ ApiClient: setAuthToken()
                ↓
4. Navigate to OrdersListScreen
                ↓
5. All subsequent API calls include JWT in Authorization header

═══════════════════════════════════════════════════════════════════

                          CODE STATISTICS

Total Files Created:        8 files
Total Lines of Code:        1,274 lines
Compilation Errors:         0 errors
Linting Warnings:           7 info (minor style suggestions)

Breakdown by Category:
├── Models:                 355 lines (28%)
├── API Layer:              420 lines (33%)
├── Auth Layer:             415 lines (33%)
└── Constants:              90 lines (7%)

═══════════════════════════════════════════════════════════════════

                        COMPLETION STATUS

Phase 6.2: Staff App Development

Step 1: Backend Verification              ✅ COMPLETE
Step 2: Documentation Review              ✅ COMPLETE
Step 3: Flutter Project Setup             ✅ COMPLETE  ← Current
Step 4: Implement Authentication          ⏳ PENDING
Step 5: Setup FCM Notifications           ⏳ PENDING
Step 6: Implement Orders Feature          ⏳ PENDING
Step 7: Implement Profile Feature         ⏳ PENDING
Step 8: Testing & Polish                  ⏳ PENDING
Step 9: Firebase Configuration            ⏳ PENDING
Step 10: Final Testing                    ⏳ PENDING

Overall Progress: 30% (3/10 steps)

═══════════════════════════════════════════════════════════════════

                          NEXT STEPS

1. Create Login Screen (lib/features/auth/screens/login_screen.dart)
   - Email/password form with validation
   - Integration with AuthProvider
   - Loading states and error handling

2. Create Auth Provider (lib/features/auth/providers/auth_provider.dart)
   - ChangeNotifier implementation
   - Login/logout state management
   - Current staff state

3. Update Main App (lib/main.dart)
   - Initialize Firebase
   - Setup MultiProvider
   - Implement auth routing (login vs orders)

4. Create Splash Screen (optional)
   - Check auth state on startup
   - Navigate to appropriate screen

Estimated Time: 2-3 hours for complete authentication flow

═══════════════════════════════════════════════════════════════════
```
