# API Contracts: Authentication

All endpoints prefixed under `/api/v1/auth`.

---

### 1. Register Student
- **Method**: `POST`
- **Path**: `/api/v1/auth/register`
- **Rate Limit**: 10 requests / 15 minutes per IP
- **Request Body**:
```json
{
  "email": "student@example.com",
  "password": "Password123!",
  "name": "Ahmed Wael"
}
```
- **Responses**:
  - `201 Created`:
  ```json
  {
    "user": {
      "id": "uuid-v4",
      "email": "student@example.com",
      "name": "Ahmed Wael",
      "role": "STUDENT"
    },
    "accessToken": "jwt-token-string"
  }
  ```
  - `400 Bad Request`: Validation failure (e.g., password too weak).
  - `409 Conflict`: Email already exists.

---

### 2. Login
- **Method**: `POST`
- **Path**: `/api/v1/auth/login`
- **Rate Limit**: 5 attempts / 15 minutes per IP/account
- **Request Body**:
```json
{
  "email": "user@example.com",
  "password": "Password123!"
}
```
- **Responses**:
  - `200 OK`:
  ```json
  {
    "user": {
      "id": "uuid-v4",
      "email": "user@example.com",
      "name": "User Name",
      "role": "ADMIN | STUDENT"
    },
    "accessToken": "jwt-token-string"
  }
  ```
  - Note: Sets `refreshToken` as an `HttpOnly`, `SameSite=Strict`, `Secure` cookie.
  - `401 Unauthorized`: Invalid credentials.

---

### 3. Refresh Access Token
- **Method**: `POST`
- **Path**: `/api/v1/auth/refresh`
- **Cookie Required**: `refreshToken`
- **Responses**:
  - `200 OK`:
  ```json
  {
    "accessToken": "new-jwt-token-string"
  }
  ```
  - `401 Unauthorized`: Token expired or revoked.

---

### 4. Logout
- **Method**: `POST`
- **Path**: `/api/v1/auth/logout`
- **Authorization**: Bearer JWT
- **Responses**:
  - `200 OK`: Clears cookie and marks refreshToken revoked in database.

---

### 5. Get Current User Profile
- **Method**: `GET`
- **Path**: `/api/v1/auth/me`
- **Authorization**: Bearer JWT
- **Responses**:
  - `200 OK`: Returns current user entity with role and active enrollment IDs.
