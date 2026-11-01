# API Contracts: Payments (InstaPay Manual Verification)

All endpoints prefixed under `/api/v1/payments`.

---

### 1. Request Screenshot Upload Presigned URL
- **Method**: `POST`
- **Path**: `/api/v1/payments/upload-url`
- **Access**: Student (`requireRole('STUDENT')`)
- **Request Body**:
```json
{
  "courseId": "course-uuid",
  "fileName": "receipt.png",
  "mimeType": "image/png",
  "fileSize": 1250000
}
```
- **Response `200 OK`**:
```json
{
  "uploadUrl": "https://s3.amazonaws.com/bucket/payments/receipt-uuid.png?AWSAccessKeyId=...",
  "s3Key": "payments/receipt-uuid.png"
}
```

---

### 2. Submit Manual Payment Proof
- **Method**: `POST`
- **Path**: `/api/v1/payments`
- **Access**: Student
- **Rules**:
  - Cannot submit if user already has a `PENDING` payment for this course.
  - Computes `priceSnapshot` from current `Course.price`.
  - Computes or receives `screenshotHash` (SHA-256) of receipt.
- **Request Body**:
```json
{
  "courseId": "course-uuid",
  "declaredAmount": 499.00,
  "transactionRef": "INSTA-987654321",
  "screenshotKey": "payments/receipt-uuid.png",
  "screenshotHash": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"
}
```
- **Response `201 Created`**:
```json
{
  "id": "payment-uuid",
  "status": "PENDING",
  "declaredAmount": "499.00",
  "priceSnapshot": "499.00",
  "createdAt": "2026-10-30T22:00:00Z"
}
```

---

### 3. Admin: List Payment Submissions
- **Method**: `GET`
- **Path**: `/api/v1/payments/admin/queue`
- **Access**: Admin (`requireRole('ADMIN')`)
- **Query Params**: `status=PENDING&page=1&limit=20`
- **Response `200 OK`**:
```json
{
  "items": [
    {
      "id": "payment-uuid",
      "student": {
        "id": "user-uuid",
        "name": "Ahmed Wael",
        "email": "ahmed@example.com"
      },
      "course": {
        "id": "course-uuid",
        "title": "Full Stack Mastery"
      },
      "declaredAmount": "450.00",
      "priceSnapshot": "499.00",
      "hasAmountMismatch": true,
      "isDuplicateScreenshot": true,
      "screenshotSignedUrl": "https://s3.../receipt.png?token=...",
      "transactionRef": "INSTA-987654321",
      "status": "PENDING",
      "createdAt": "2026-10-30T21:45:00Z"
    }
  ]
}
```

---

### 4. Admin: Approve or Decline Payment
- **Method**: `PATCH`
- **Path**: `/api/v1/payments/admin/:id/status`
- **Access**: Admin
- **Request Body**:
```json
{
  "status": "APPROVED | DECLINED",
  "declineReason": "Amount declared is less than required course fee (450 vs 499 EGP)."
}
```
- **Atomic Execution**:
  - If `APPROVED`: Sets `status: APPROVED`, records `reviewedById`, creates/reactivates `Enrollment` in single Prisma transaction, and emits `payment:status_updated` socket event + Notification.
  - If `DECLINED`: Sets `status: DECLINED`, saves `declineReason`, emits notification to student.
