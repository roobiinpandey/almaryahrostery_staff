# 🎯 QR Badge + PIN Authentication System - Implementation Plan

## Project Overview
Transform Staff App authentication from Firebase email/password to a hybrid QR Badge + PIN system optimized for coffee shop operations.

---

## 📋 Day 1: Backend Foundation (6-8 hours)

### Phase 1.1: Database Schema Updates
**File:** `backend/models/Staff.js`

**Add New Fields:**
```javascript
// New Authentication Fields
employeeId: {
  type: String,
  required: true,
  unique: true,
  index: true,
  uppercase: true,
  match: /^[A-Z]{3}\d{3}$/  // Format: EMP001, BAR001, MNG001
},

pin: {
  type: String,
  required: true,
  minlength: 60  // BCrypt hashed
},

pinAttempts: {
  type: Number,
  default: 0,
  max: 3
},

pinLockedUntil: {
  type: Date,
  default: null
},

qrBadgeToken: {
  type: String,
  unique: true,
  index: true,
  sparse: true  // Allows null values
},

qrBadgeGeneratedAt: {
  type: Date,
  default: null
},

qrBadgeExpiresAt: {
  type: Date,
  default: null  // Optional: Rotate QR codes every 6 months
},

photo: {
  type: String,  // Cloudinary URL
  default: null
},

shiftStartTime: {
  type: String,  // "08:00"
  default: "08:00"
},

shiftEndTime: {
  type: String,  // "16:00"
  default: "16:00"
},

// Modified: Make these optional for backward compatibility
firebaseUid: {
  type: String,
  required: false,  // Changed from true
  unique: true,
  sparse: true,  // Allows multiple null values
  index: true,
  trim: true
},

email: {
  type: String,
  required: false,  // Changed from true
  unique: true,
  sparse: true,
  lowercase: true,
  trim: true,
  match: [/^\S+@\S+\.\S+$/, 'Please enter a valid email address']
},

// Audit Trail
loginHistory: [{
  timestamp: Date,
  method: String,  // 'qr', 'pin', 'firebase'
  deviceInfo: String,
  ipAddress: String,
  success: Boolean
}]
```

**Virtual Fields:**
```javascript
// Virtual field for active session check
staffSchema.virtual('isOnShift').get(function() {
  const now = new Date();
  const currentTime = now.getHours() * 100 + now.getMinutes();
  const [startHour, startMin] = this.shiftStartTime.split(':').map(Number);
  const [endHour, endMin] = this.shiftEndTime.split(':').map(Number);
  const shiftStart = startHour * 100 + startMin;
  const shiftEnd = endHour * 100 + endMin;
  return currentTime >= shiftStart && currentTime <= shiftEnd;
});
```

**Methods:**
```javascript
// Generate unique employee ID
staffSchema.statics.generateEmployeeId = async function(role) {
  const rolePrefix = {
    'barista': 'BAR',
    'manager': 'MNG',
    'cashier': 'CSH'
  };
  const prefix = rolePrefix[role] || 'EMP';
  
  // Find last ID with this prefix
  const lastStaff = await this.findOne({ 
    employeeId: new RegExp(`^${prefix}`) 
  }).sort('-employeeId');
  
  const lastNum = lastStaff ? parseInt(lastStaff.employeeId.slice(3)) : 0;
  const newNum = (lastNum + 1).toString().padStart(3, '0');
  return `${prefix}${newNum}`;
};

// Validate PIN
staffSchema.methods.validatePin = async function(pin) {
  const bcrypt = require('bcrypt');
  return await bcrypt.compare(pin, this.pin);
};

// Hash PIN before save
staffSchema.pre('save', async function(next) {
  if (this.isModified('pin') && this.pin.length === 4) {
    const bcrypt = require('bcrypt');
    this.pin = await bcrypt.hash(this.pin, 10);
  }
  next();
});
```

---

### Phase 1.2: New Authentication Routes
**File:** `backend/routes/staffAuth.js` (NEW)

**Endpoints:**

#### **POST /api/staff/auth/login-pin**
```javascript
/**
 * Login with Employee ID + PIN
 * No Firebase required
 */
{
  employeeId: "BAR001",
  pin: "1234"
}
→ Returns: { token, staff, sessionExpiry }
```

#### **POST /api/staff/auth/login-qr**
```javascript
/**
 * Login with QR Badge scan
 * Instant authentication
 */
{
  qrToken: "encrypted_qr_badge_token"
}
→ Returns: { token, staff, sessionExpiry }
```

#### **POST /api/staff/auth/validate-pin**
```javascript
/**
 * Check if PIN is correct (for admin reset, etc)
 */
{
  employeeId: "BAR001",
  pin: "1234"
}
→ Returns: { valid: true/false }
```

#### **POST /api/staff/auth/reset-pin** (Admin only)
```javascript
/**
 * Admin resets staff PIN
 */
{
  employeeId: "BAR001",
  newPin: "5678",
  requireChange: true  // Force change on next login
}
→ Returns: { success: true }
```

#### **POST /api/staff/auth/generate-qr** (Admin only)
```javascript
/**
 * Generate/regenerate QR badge for staff
 */
{
  employeeId: "BAR001"
}
→ Returns: { 
  qrToken: "...", 
  qrCodeDataUrl: "data:image/png;base64,...",
  badgePdfUrl: "..."
}
```

#### **GET /api/staff/auth/session**
```javascript
/**
 * Check current session validity
 */
Headers: { Authorization: "Bearer jwt" }
→ Returns: { 
  staff: {...}, 
  expiresAt: Date,
  isOnShift: true/false 
}
```

#### **POST /api/staff/auth/logout**
```javascript
/**
 * End staff session
 */
Headers: { Authorization: "Bearer jwt" }
→ Returns: { success: true }
```

---

### Phase 1.3: QR Code Generation Service
**File:** `backend/services/qrBadgeService.js` (NEW)

```javascript
const QRCode = require('qrcode');
const crypto = require('crypto');
const PDFDocument = require('pdfkit');
const cloudinary = require('cloudinary').v2;

class QRBadgeService {
  
  // Generate secure QR token
  generateQRToken(employeeId) {
    const payload = {
      employeeId,
      timestamp: Date.now(),
      random: crypto.randomBytes(16).toString('hex')
    };
    
    // Encrypt payload
    const secret = process.env.QR_BADGE_SECRET || process.env.JWT_SECRET;
    const encrypted = this.encrypt(JSON.stringify(payload), secret);
    return encrypted;
  }
  
  // Decrypt and validate QR token
  validateQRToken(qrToken) {
    try {
      const secret = process.env.QR_BADGE_SECRET || process.env.JWT_SECRET;
      const decrypted = this.decrypt(qrToken, secret);
      const payload = JSON.parse(decrypted);
      
      // Optional: Check token age (e.g., expire after 6 months)
      const sixMonths = 6 * 30 * 24 * 60 * 60 * 1000;
      if (Date.now() - payload.timestamp > sixMonths) {
        return { valid: false, reason: 'expired' };
      }
      
      return { valid: true, employeeId: payload.employeeId };
    } catch (error) {
      return { valid: false, reason: 'invalid' };
    }
  }
  
  // Generate QR code image
  async generateQRCodeImage(qrToken) {
    const qrDataUrl = await QRCode.toDataURL(qrToken, {
      errorCorrectionLevel: 'H',
      type: 'image/png',
      quality: 1,
      margin: 2,
      width: 400,
      color: {
        dark: '#000000',
        light: '#FFFFFF'
      }
    });
    return qrDataUrl;
  }
  
  // Generate staff badge PDF
  async generateBadgePDF(staff, qrCodeDataUrl) {
    return new Promise((resolve, reject) => {
      const doc = new PDFDocument({ size: [250, 350] });
      const chunks = [];
      
      doc.on('data', chunk => chunks.push(chunk));
      doc.on('end', () => resolve(Buffer.concat(chunks)));
      
      // Badge design
      doc.rect(0, 0, 250, 350).fill('#A89A6A');  // Olive gold background
      doc.rect(10, 10, 230, 330).fill('#FFFFFF');  // White card
      
      // Logo/Header
      doc.fontSize(18).fillColor('#A89A6A')
         .text('AL MARYA ROSTERY', 20, 30, { align: 'center' });
      
      // Staff photo (if available)
      if (staff.photo) {
        // Add photo (requires image buffer)
        doc.image(staff.photo, 75, 70, { width: 100, height: 100 });
      } else {
        // Placeholder circle
        doc.circle(125, 120, 50).stroke('#CCCCCC');
      }
      
      // Staff details
      doc.fontSize(16).fillColor('#000000')
         .text(staff.name, 20, 180, { align: 'center' });
      
      doc.fontSize(12).fillColor('#666666')
         .text(staff.role.toUpperCase(), 20, 205, { align: 'center' });
      
      doc.fontSize(14).fillColor('#A89A6A')
         .text(staff.employeeId, 20, 230, { align: 'center' });
      
      // QR Code
      doc.image(qrCodeDataUrl, 50, 250, { width: 150, height: 150 });
      
      doc.end();
    });
  }
  
  // Encryption helpers
  encrypt(text, secret) {
    const algorithm = 'aes-256-cbc';
    const key = crypto.scryptSync(secret, 'salt', 32);
    const iv = crypto.randomBytes(16);
    const cipher = crypto.createCipheriv(algorithm, key, iv);
    let encrypted = cipher.update(text, 'utf8', 'hex');
    encrypted += cipher.final('hex');
    return iv.toString('hex') + ':' + encrypted;
  }
  
  decrypt(encryptedText, secret) {
    const algorithm = 'aes-256-cbc';
    const key = crypto.scryptSync(secret, 'salt', 32);
    const parts = encryptedText.split(':');
    const iv = Buffer.from(parts[0], 'hex');
    const encrypted = parts[1];
    const decipher = crypto.createDecipheriv(algorithm, key, iv);
    let decrypted = decipher.update(encrypted, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    return decrypted;
  }
}

module.exports = new QRBadgeService();
```

---

## 📋 Day 2: Admin Panel Integration (6-8 hours)

### Phase 2.1: Admin Staff Management UI
**Files:** `backend/views/admin/staff/` (EJS templates)

**Features:**
1. **Staff List View**
   - Display Employee ID, Name, Role, Status
   - QR badge status (generated/not generated)
   - PIN status (set/not set, locked/unlocked)
   - Quick actions: Reset PIN, Generate Badge

2. **Create Staff Form**
   - Auto-generate Employee ID based on role
   - Set initial PIN (default: 1234, must change)
   - Upload photo
   - Set shift times
   - Generate QR badge on creation

3. **Staff Detail View**
   - Full staff information
   - Download/Print QR badge button
   - Reset PIN button
   - View login history
   - Regenerate QR badge

4. **Badge Management**
   - Bulk badge generation
   - Print all badges (PDF export)
   - Badge preview before printing

### Phase 2.2: Admin API Endpoints
**File:** `backend/routes/admin.js`

```javascript
// Staff management endpoints for admin
GET  /admin/staff → List all staff
GET  /admin/staff/:id → Staff details
POST /admin/staff/create → Create new staff
PUT  /admin/staff/:id → Update staff
DELETE /admin/staff/:id → Soft delete staff
POST /admin/staff/:id/reset-pin → Reset PIN
POST /admin/staff/:id/generate-badge → Generate QR badge
GET  /admin/staff/:id/badge-pdf → Download badge PDF
GET  /admin/staff/:id/login-history → View login history
```

---

## 📋 Day 3: Flutter Staff App UI (8-10 hours)

### Phase 3.1: New Login Screen
**File:** `lib/features/auth/screens/pin_login_screen.dart`

**UI Components:**
```dart
- Tab switcher: [QR Scan] [PIN Login]
- Employee ID dropdown/autocomplete (cached from previous logins)
- Large numeric keypad (0-9)
- PIN dots display (●●●●)
- Clear/Backspace button
- Login button
- Offline indicator
- "Forgot PIN?" link (shows admin contact)
```

### Phase 3.2: QR Scanner Screen
**File:** `lib/features/auth/screens/qr_scanner_screen.dart`

**Features:**
- Camera permission handling
- QR code scanner (using `qr_code_scanner` package)
- Scan guidance overlay
- Fallback to PIN if camera unavailable
- Success/error feedback

### Phase 3.3: Updated Auth Service
**File:** `lib/core/auth/pin_auth_service.dart` (NEW)

```dart
class PinAuthService {
  // Login with PIN
  Future<Staff> loginWithPin(String employeeId, String pin);
  
  // Login with QR
  Future<Staff> loginWithQR(String qrToken);
  
  // Validate session
  Future<bool> validateSession();
  
  // Logout
  Future<void> logout();
  
  // Cache employee ID for quick login
  Future<void> cacheEmployeeId(String employeeId);
  Future<String?> getCachedEmployeeId();
}
```

### Phase 3.4: Offline Support
**File:** `lib/core/storage/offline_auth_storage.dart` (NEW)

```dart
// Encrypted local storage for offline login
- Cache last successful login (encrypted)
- Store JWT token securely
- Sync login attempts when online
- Handle token refresh
```

---

## 📋 Day 4: Testing & Polish (6-8 hours)

### Phase 4.1: Integration Testing
1. Test PIN login flow
2. Test QR scan flow
3. Test offline mode
4. Test PIN lockout (3 failed attempts)
5. Test session expiry
6. Test admin badge generation
7. Test badge printing

### Phase 4.2: Migration Script
**File:** `backend/scripts/migrate-to-pin-auth.js`

```javascript
// Migrate existing Firebase staff to PIN system
- Generate Employee IDs for all staff
- Set default PIN: 1234 (require change on first login)
- Generate QR badges for all staff
- Keep Firebase as fallback temporarily
```

### Phase 4.3: Documentation
- Update API documentation
- Create staff login guide (with screenshots)
- Create admin badge management guide
- Update SECURE_CREDENTIALS.md

---

## 🔧 Dependencies to Install

### Backend:
```bash
npm install qrcode
npm install pdfkit
npm install bcrypt  # Already installed
```

### Flutter:
```yaml
dependencies:
  qr_code_scanner: ^1.0.1
  qr_flutter: ^4.1.0  # For displaying QR codes
  flutter_secure_storage: ^9.0.0  # For secure PIN storage
  pin_code_fields: ^8.0.1  # Beautiful PIN input
```

---

## 📊 Testing Checklist

### Backend Tests:
- [ ] Employee ID generation (unique, sequential)
- [ ] PIN hashing and validation
- [ ] QR token encryption/decryption
- [ ] QR badge PDF generation
- [ ] PIN lockout after 3 failed attempts
- [ ] Session JWT validation
- [ ] Admin endpoints (auth required)

### App Tests:
- [ ] PIN login success
- [ ] PIN login failure (wrong PIN)
- [ ] PIN lockout UI
- [ ] QR scan success
- [ ] QR scan invalid code
- [ ] Offline login (cached)
- [ ] Session expiry handling
- [ ] Camera permission denied
- [ ] Numeric keypad responsiveness

---

## 🚀 Deployment Steps

1. **Backup database**
2. **Deploy backend updates**
3. **Run migration script**
4. **Generate QR badges for all staff**
5. **Print badges**
6. **Deploy Flutter app update**
7. **Train staff on new login**
8. **Monitor login issues for 1 week**
9. **Remove Firebase dependencies** (optional)

---

## 📱 Sample Staff Badge Design

```
┌─────────────────────────────────┐
│                                 │
│     AL MARYA ROSTERY            │
│     ═══════════════             │
│                                 │
│        ┌───────────┐            │
│        │           │            │
│        │   PHOTO   │            │
│        │           │            │
│        └───────────┘            │
│                                 │
│      AHMED HASSAN               │
│      BARISTA                    │
│      BAR001                     │
│                                 │
│      ┌───────────────┐          │
│      │               │          │
│      │   QR  CODE    │          │
│      │               │          │
│      └───────────────┘          │
│                                 │
│   Scan to login to Staff App   │
│                                 │
└─────────────────────────────────┘
```

---

## 🎯 Success Metrics

- ✅ Login time < 3 seconds (QR)
- ✅ Login time < 10 seconds (PIN)
- ✅ 99%+ uptime for offline mode
- ✅ Zero forgotten PIN issues (admin can reset)
- ✅ Zero shared credential issues (badges are physical)
- ✅ Staff satisfaction > 90%

---

## 📝 Next Steps

**Ready to start?** I'll begin with:

1. **Day 1, Phase 1.1** - Update Staff model with new fields
2. **Day 1, Phase 1.2** - Create authentication routes
3. **Day 1, Phase 1.3** - Build QR badge service

Would you like me to proceed with the implementation? 🚀
