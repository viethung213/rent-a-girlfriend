# Danh sách và Cấu trúc Sự kiện (Events) - Identity Service

Tài liệu này mô tả chi tiết các sự kiện (Domain Events) được phát ra từ Identity Service thông qua hệ thống **Transactional Outbox** lên Kafka.

## 1. Thông tin chung
- **Kafka Topic**: `identity-events`
- **Format**: JSON
- **Nguyên tắc**: CloudEvents-like structure (Type, Payload, Timestamp).

---

## 2. Danh sách Sự kiện

### 2.1. UserRegistered
Phát ra khi một tài khoản người dùng mới được tạo (thông qua Google Login lần đầu).
- **Type**: `com.rentagf.identity.UserRegistered.v1`
- **Payload**:
```json
{
  "userId": "uuid",
  "email": "user@example.com",
  "role": "CLIENT",
  "googleId": "123456789",
  "timestamp": "2026-05-11T04:17:00Z"
}
```

### 2.2. ViolationRecorded
Phát ra khi một vi phạm (violation) được ghi nhận cho người dùng (ví dụ: hủy lịch sát giờ).
- **Type**: `com.rentagf.identity.ViolationRecorded.v1`
- **Payload**:
```json
{
  "userId": "uuid",
  "currentCount": 1,
  "reason": "Late cancellation",
  "bookingId": "uuid-booking",
  "timestamp": "2026-05-11T04:17:00Z"
}
```

### 2.3. AccountLocked
Phát ra khi tài khoản bị khóa (do Admin hoặc do vượt quá số lần vi phạm).
- **Type**: `com.rentagf.identity.AccountLocked.v1`
- **Payload**:
```json
{
  "userId": "uuid",
  "reason": "Exceeded violation threshold",
  "lockedBy": "system-or-admin-id",
  "timestamp": "2026-05-11T04:17:00Z"
}
```

### 2.4. AccountUnlocked
Phát ra khi tài khoản được mở khóa bởi Admin.
- **Type**: `com.rentagf.identity.AccountUnlocked.v1`
- **Payload**:
```json
{
  "userId": "uuid",
  "unlockedBy": "admin-id",
  "timestamp": "2026-05-11T04:17:00Z"
}
```

### 2.5. CompanionUpgradeRequested
Phát ra khi một Client gửi yêu cầu nâng cấp lên Companion.
- **Type**: `com.rentagf.identity.CompanionUpgradeRequested.v1`
- **Payload**:
```json
{
  "requestId": "uuid-request",
  "userId": "uuid",
  "reason": "I want to be a companion",
  "timestamp": "2026-05-11T04:17:00Z"
}
```

### 2.6. CompanionUpgradeApproved
Phát ra khi Admin phê duyệt yêu cầu nâng cấp.
- **Type**: `com.rentagf.identity.CompanionUpgradeApproved.v1`
- **Payload**:
```json
{
  "requestId": "uuid-request",
  "userId": "uuid",
  "approvedBy": "admin-id",
  "timestamp": "2026-05-11T04:17:00Z"
}
```

### 2.7. CompanionUpgradeRejected
Phát ra khi Admin từ chối yêu cầu nâng cấp.
- **Type**: `com.rentagf.identity.CompanionUpgradeRejected.v1`
- **Payload**:
```json
{
  "requestId": "uuid-request",
  "userId": "uuid",
  "rejectedBy": "admin-id",
  "rejectReason": "Profile incomplete",
  "timestamp": "2026-05-11T04:17:00Z"
}
```

### 2.8. RoleUpgraded
Phát ra khi quyền hạn của người dùng thay đổi (thường đi kèm với `CompanionUpgradeApproved`).
- **Type**: `com.rentagf.identity.RoleUpgraded.v1`
- **Payload**:
```json
{
  "userId": "uuid",
  "oldRole": "CLIENT",
  "newRole": "COMPANION",
  "timestamp": "2026-05-11T04:17:00Z"
}
```
