# 🔐 Staff Authentication System - Quick Reference

## Employee ID Format

| Role | Prefix | Example |
|------|--------|---------|
| Barista | BAR | BAR001, BAR002, BAR003 |
| Manager | MNG | MNG001, MNG002 |
| Cashier | CSH | CSH001, CSH002 |

**Format:** `ABC123` (3 letters + 3 digits)

---

## Authentication Methods

### Method 1: PIN Login (10 seconds)
1. Staff enters Employee ID (e.g., BAR001)
2. Staff enters 4-digit PIN on numeric keypad
3. System validates PIN (BCrypt comparison)
4. Returns JWT token (12-hour expiry)

**Default PIN:** 1234 (must change on first login)

### Method 2: QR Badge Scan (1-2 seconds)
1. Staff scans QR badge with camera
2. System decrypts and validates token
3. Returns JWT token (12-hour expiry)

**Badge Expiry:** 6 months from generation

---

## API Endpoints

### Public Endpoints (No Auth Required)

#### POST /api/staff/auth/login-pin
```bash
curl -X POST http://localhost:5001/api/staff/auth/login-pin \
  -H "Content-Type: application/json" \
  -d '{"employeeId":"BAR001","pin":"1234"}'
```

**Response:**
```json
{
  "success": true,
  "message": "Login successful",
  "token": "eyJhbGci...",
  "staff": { ... },
  "requirePinChange": true
}
```

**Error Codes:**
- 400: Validation error (invalid format)
- 401: Invalid credentials (wrong PIN)
- 403: Account inactive
- 423: PIN locked (too many attempts)

#### POST /api/staff/auth/login-qr
```bash
curl -X POST http://localhost:5001/api/staff/auth/login-qr \
  -H "Content-Type: application/json" \
  -d '{"qrToken":"encrypted_token_from_scan"}'
```

---

### Protected Endpoints (Require JWT Token)

#### POST /api/staff/auth/change-pin
```bash
curl -X POST http://localhost:5001/api/staff/auth/change-pin \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -d '{"currentPin":"1234","newPin":"5678"}'
```

#### GET /api/staff/auth/session
```bash
curl -X GET http://localhost:5001/api/staff/auth/session \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

#### POST /api/staff/auth/logout
```bash
curl -X POST http://localhost:5001/api/staff/auth/logout \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

---

## Security Features

### PIN Security
- **Format:** 4 digits (0000-9999)
- **Hashing:** BCrypt (10 rounds)
- **Lockout:** 3 failed attempts = 15-minute lockout
- **Force Change:** Must change default PIN (1234) on first login

### QR Badge Security
- **Encryption:** AES-256-CBC
- **Token Format:** Encrypted payload with timestamp + random
- **Expiry:** 6 months from generation
- **Unique:** One token per staff member

### Session Security
- **Token Type:** JWT
- **Expiry:** 12 hours
- **Signing:** HMAC SHA256
- **Payload:** staffId, employeeId, role, type

### Audit Trail
- Every login attempt logged
- Tracks: timestamp, method (PIN/QR), device, IP, success/failure
- Keeps last 50 entries per staff
- Failure reason logged for security analysis

---

## Staff Model Fields

### Authentication Fields
```javascript
{
  employeeId: "BAR001",           // Unique staff identifier
  pin: "$2b$10$...",              // BCrypt hashed PIN
  pinAttempts: 0,                 // Failed attempt counter
  pinLockedUntil: null,           // Lockout timestamp
  requirePinChange: true,         // Force PIN change flag
  qrBadgeToken: "encrypted...",   // Encrypted QR token
  qrBadgeGeneratedAt: Date,       // Token creation date
  qrBadgeExpiresAt: Date,         // Token expiry date (6 months)
  photo: "https://...",           // Staff photo (for badge)
  shiftStartTime: "08:00",        // Shift start
  shiftEndTime: "16:00",          // Shift end
  loginHistory: [...],            // Audit trail (last 50)
  lastLoginAt: Date               // Last successful login
}
```

### Virtual Fields (Computed)
```javascript
{
  isOnShift: false,         // Current time within shift hours
  isPinLocked: false,       // PIN currently locked
  isQrBadgeValid: true      // QR badge not expired
}
```

---

## Database Scripts

### Migration
```bash
node backend/scripts/migrate-to-pin-auth.js
```
Converts existing staff to PIN + QR badge system

### Check Status
```bash
node backend/scripts/check-staff-state.js
```
Shows current authentication status for all staff

### Test Login
```bash
./backend/start-and-test.sh
```
Starts server and tests PIN login

---

## Common Issues & Solutions

### Issue: "Invalid Employee ID format"
- **Cause:** Employee ID not in ABC123 format
- **Solution:** Use correct format (e.g., BAR001, MNG001, CSH001)

### Issue: "Invalid PIN. X attempts remaining"
- **Cause:** Wrong PIN entered
- **Solution:** Double-check PIN, reset if forgotten

### Issue: "PIN locked. Try again in X minutes"
- **Cause:** 3 failed login attempts
- **Solution:** Wait 15 minutes or admin unlocks

### Issue: "QR badge expired"
- **Cause:** Badge is older than 6 months
- **Solution:** Admin generates new QR badge

### Issue: "Staff account inactive"
- **Cause:** Status set to inactive
- **Solution:** Admin activates account

---

## Migration Summary

**Migrated Staff (5 total):**
- BAR001 - Test Barista
- BAR002 - Test Staff FCM
- BAR003 - Test Staff Notification
- BAR004 - Test Staff 1762341516
- BAR005 - Smoke Test Staff

**All staff:**
- ✅ Have unique Employee IDs
- ✅ Have default PIN (1234)
- ✅ Must change PIN on first login
- ✅ Have valid QR badges (expires in 6 months)
- ✅ Retain Firebase credentials (backward compatible)

---

## Next Steps

### For Developers (Day 2: Admin Panel)
1. Create admin staff management routes
2. Build badge PDF generator
3. Implement PIN reset interface
4. Add login history view

### For Developers (Day 3: Flutter App)
1. Create PIN login screen with numeric keypad
2. Implement QR scanner screen
3. Update AuthService to use new endpoints
4. Add offline PIN caching

### For Admins
1. Generate and print QR badges for all staff
2. Distribute badges to staff members
3. Train staff on new login methods (PIN + QR)
4. Monitor login history for issues

---

**Last Updated:** November 5, 2025  
**Version:** 1.0  
**Status:** Day 1 Complete
