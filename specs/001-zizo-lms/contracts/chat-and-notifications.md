# API & Realtime Contracts: Chat, Comments & Notifications

Endpoints under `/api/v1/chat`, `/api/v1/comments`, `/api/v1/notifications`.

---

## 1. REST Endpoints

### 1.1 Video Comments
- `GET /api/v1/comments/video/:videoId` - List comments with one-level nested replies.
- `POST /api/v1/comments/video/:videoId` - Post new comment (enforces `parentId` depth ≤ 1).
- `DELETE /api/v1/comments/:id` - Delete comment (cascades to all child replies).

### 1.2 Conversations & Messages
- `GET /api/v1/chat/conversations` - List conversations (Admin sees all student chats; Student sees admin chat).
- `GET /api/v1/chat/conversations/:id/messages` - Paginated message history (immutable).
- `POST /api/v1/chat/conversations/:id/messages` - Send message (creates message + emits socket event).

### 1.3 Notifications
- `GET /api/v1/notifications` - Get user notifications (read/unread).
- `PATCH /api/v1/notifications/:id/read` - Mark notification as read.

---

## 2. Socket.IO Realtime Events

### Connection Handshake
- **Auth**: Token passed in `auth: { token: "jwt-string" }`.
- **Rooms Joined**:
  - `user:<userId>`
  - `admins` (if role is ADMIN)

### Events
| Event Name | Direction | Payload | Description |
|---|---|---|---|
| `message:send` | Client -> Server | `{ conversationId, content }` | Send chat message |
| `message:new` | Server -> Client | `{ id, conversationId, senderId, content, createdAt }` | Broadcast new message to conversation room |
| `payment:submitted` | Server -> `admins` | `{ paymentId, studentName, courseTitle, declaredAmount }` | Realtime notification for Admin review queue |
| `payment:status_changed` | Server -> `user:<id>` | `{ paymentId, status, courseId, declineReason }` | Instant alert to student on approval/decline |
| `notification:new` | Server -> `user:<id>` | `{ id, title, body, type, createdAt }` | In-app notification popup |
