# Three-Entity Messaging Service Backend

A scalable, real-time messaging system tailored for a three-entity business model: 
1. **Operators** (Company Admin/Dispatch)
2. **Drivers** (Service Providers)
3. **Customers** (End Users)

## Architecture

* **API & Real-time Layer**: Node.js, Express, Socket.IO
* **Database**: MS SQL Server (via TypeORM) for storing messages/history
* **In-Memory Store**: Redis 
  * Pub/Sub for scaling multiple node instances
  * Set/Hash storage for tracking online/offline states and driver locations
* **Authentication**: JWT containing `userId`, `role`, and `companyId` for multi-tenant isolation.

## Features

* **Multi-tenancy**: All data and socket rooms are scoped by `companyId`.
* **Direct Messaging**: 1-to-1 messaging between Customer↔Driver, Driver↔Operator.
* **Unified Operator Entity**: Operators represent the "company". If a driver messages the operator, any operator can respond, and the read state synchronizes.
* **Broadcasts**: Operators can broadcast messages to all drivers, online drivers, or all operators.
* **Message Deleting**: Operators can soft-delete messages or entire conversations, which instantly propagates via Socket.IO.
* **Chat History**: Infinite scrolling via `/history` endpoints.
* **Typing Indicators & Read Receipts**: Real-time status syncing.
* **Offline Handling**: Fallback mechanisms when WebSockets disconnect (FCM triggers).

---

## API Endpoints

**Base URL:** `http://<host>:3000/api`

All endpoints (except webhooks and health check) require a JWT in the `Authorization: Bearer <token>` header. The JWT payload must contain `userId`, `role` (`driver` | `operator` | `customer`), and `companyId`.

---

### 🚗 Sending Messages — Driver

A driver can send messages to an operator or a customer.

#### Driver → Operator

```
POST /api/chat/driver/{driverId}/send
```

| Field | Location | Required | Description |
|---|---|---|---|
| `driverId` | URL param | Yes | The driver's ID (must match JWT `userId`) |
| `content` | Body | Yes | Message text |
| `recipientType` | Body | Yes | Set to `"operator"` |
| `recipientId` | Body | Yes | Target operator ID, or `"all"` to open a general operator chat |
| `jobId` | Body | No | If set, message is linked to a job conversation |
| `messageType` | Body | No | `"text"` (default), `"image"`, `"location"` |
| `isHighPriority` | Body | No | `true` to flag as high priority |
| `metadata` | Body | No | Arbitrary JSON metadata |

**Example:**
```json
POST /api/chat/driver/DRV001/send

{
  "content": "I have arrived at the pickup location",
  "recipientType": "operator",
  "recipientId": "all",
  "jobId": "JOB-5432"
}
```

#### Driver → Customer

```
POST /api/chat/driver/{driverId}/send
```

| Field | Location | Required | Description |
|---|---|---|---|
| `driverId` | URL param | Yes | The driver's ID |
| `content` | Body | Yes | Message text |
| `recipientType` | Body | Yes | Set to `"customer"` |
| `recipientId` | Body | Yes | Target customer ID |
| `jobId` | Body | No | Associated job ID |
| `messageType` | Body | No | `"text"` (default), `"image"`, `"location"` |

**Example:**
```json
POST /api/chat/driver/DRV001/send

{
  "content": "I am 5 minutes away",
  "recipientType": "customer",
  "recipientId": "CUST-1234",
  "jobId": "JOB-5432"
}
```

---

### 👤 Sending Messages — Customer

A customer can send messages to a driver (typically for an active job).

#### Customer → Driver

```
POST /api/chat/customer/{customerId}/send
```

| Field | Location | Required | Description |
|---|---|---|---|
| `customerId` | URL param | Yes | The customer's ID (must match JWT `userId`) |
| `content` | Body | Yes | Message text |
| `recipientType` | Body | Yes | Set to `"driver"` |
| `recipientId` | Body | Yes | Target driver ID |
| `jobId` | Body | No | Associated job ID |
| `messageType` | Body | No | `"text"` (default), `"image"`, `"location"` |

**Example:**
```json
POST /api/chat/customer/CUST-1234/send

{
  "content": "I am waiting at the main entrance",
  "recipientType": "driver",
  "recipientId": "DRV001",
  "jobId": "JOB-5432"
}
```

---

### 🎧 Sending Messages — Operator

Operators can message drivers, customers, and broadcast to groups.

#### Operator → Driver

```
POST /api/chat/operator/{operatorId}/send
```

| Field | Location | Required | Description |
|---|---|---|---|
| `operatorId` | URL param | Yes | The operator's ID (must match JWT `userId`) |
| `content` | Body | Yes | Message text |
| `recipientType` | Body | Yes | Set to `"driver"` |
| `recipientId` | Body | Yes | Target driver ID |
| `jobId` | Body | No | Associated job ID |
| `messageType` | Body | No | `"text"` (default), `"image"`, `"location"` |
| `isHighPriority` | Body | No | `true` to flag as high priority |

**Example:**
```json
POST /api/chat/operator/OP-01/send

{
  "content": "Please head to the airport terminal 2",
  "recipientType": "driver",
  "recipientId": "DRV001",
  "jobId": "JOB-5432"
}
```

#### Operator → Customer

```
POST /api/chat/operator/{operatorId}/send
```

| Field | Location | Required | Description |
|---|---|---|---|
| `operatorId` | URL param | Yes | The operator's ID |
| `content` | Body | Yes | Message text |
| `recipientType` | Body | Yes | Set to `"customer"` |
| `recipientId` | Body | Yes | Target customer ID |
| `jobId` | Body | No | Associated job ID |

**Example:**
```json
POST /api/chat/operator/OP-01/send

{
  "content": "Your driver is on the way",
  "recipientType": "customer",
  "recipientId": "CUST-1234"
}
```

#### Operator → Broadcast

```
POST /api/chat/broadcast
```
*Role required: `operator` or `admin`*

| Field | Location | Required | Description |
|---|---|---|---|
| `target` | Body | Yes | `"all_drivers"`, `"online_drivers"`, or `"all_operators"` |
| `content` | Body | Yes | Broadcast message text |
| `messageType` | Body | No | `"text"` (default) |
| `isHighPriority` | Body | No | `true` to flag as high priority |

**Example:**
```json
POST /api/chat/broadcast

{
  "target": "all_drivers",
  "content": "Road closure on Main St — avoid the area"
}
```

---

### ✅ Message Status Updates

#### Mark as Delivered

```
PUT /api/chat/message/{messageId}/delivered
```

Updates a message's status to `"delivered"` and records the timestamp. Sends a delivery acknowledgement back to the sender via WebSocket.

#### Mark as Read

```
PUT /api/chat/message/{messageId}/read
```

Updates a message's status to `"read"` and records the timestamp. Sends a read acknowledgement back to the sender. If the reader is an operator, read status syncs to all operators.

---

### 🗑️ Delete (Operator Only)

#### Delete a Message

```
DELETE /api/chat/message/{messageId}
```

Soft-deletes a message. Propagates the deletion event in real-time via WebSocket.

#### Delete a Conversation

```
DELETE /api/chat/conversation/{conversationId}
```

Soft-deletes an entire conversation. Propagates the deletion event in real-time via WebSocket.

---

### 📥 Offline Messages

```
GET /api/chat/offline
```

Retrieves all queued messages for the authenticated user that were sent while they were offline. **Clears the queue** after fetching.

---

### 📜 Chat History

#### Recent Conversations (Operator Only)

```
GET /api/history/recent
```

Returns the 50 most recent active conversations for the company. Ordered by last updated.

#### Job History

```
GET /api/history/job/{jobId}
```

Returns all messages for a specific job conversation, ordered chronologically.

#### Conversation Messages

```
GET /api/history/conversation/{conversationId}/messages
```

Returns all messages within a specific conversation, ordered chronologically.

---

### 👥 User Management (Operator Only)

#### Block a User

```
POST /api/users/block/{userId}
```

Blocks a user (driver or customer) from sending messages and connecting via WebSocket.

#### Unblock a User

```
POST /api/users/unblock/{userId}
```

Removes the block on a user.

#### List Online Drivers

```
GET /api/users/online-drivers
```

Returns the set of currently online driver IDs for the company.

---

### 🔗 Webhooks (Public — No Auth)

#### Twilio SMS Inbound

```
POST /api/webhooks/twilio/sms
```

Receives inbound SMS messages from Twilio. Configure this URL as your Twilio SMS webhook.

#### WhatsApp Inbound

```
GET  /api/webhooks/whatsapp   ← Verification handshake
POST /api/webhooks/whatsapp   ← Incoming messages
```

Receives inbound WhatsApp messages. Configure this URL as your WhatsApp Business API webhook.

---

### ❤️ Health Check

```
GET /api/health
```

Returns `{ "status": "ok", "timestamp": "..." }`.

---

## WebSocket Events (Socket.IO)

Connect with: `io("http://<host>:3000", { auth: { token: "<JWT>" } })`

### Client → Server

| Event | Payload | Description |
|---|---|---|
| `room:join` | `jobId` (string) | Join a job-specific chat room |
| `typing:start` | `{ room?: string, recipientId?: string }` | Notify others that the user is typing |
| `typing:stop` | `{ room?: string, recipientId?: string }` | Notify others that the user stopped typing |

### Server → Client

| Event | Description |
|---|---|
| `message:receive` | New message received (direct, job, or broadcast) |
| `message:deleted` | A message was soft-deleted |
| `conversation:deleted` | A conversation was soft-deleted |
| `delivery_ack` | Delivery acknowledgement for a sent message |
| `read_ack` | Read acknowledgement for a sent message |
| `read_ack_sync` | Operator-wide read sync notification |
| `typing:status` | `{ userId, role, isTyping }` — typing indicator update |
| `error` | Error message (e.g., blocked user) |

---

## Running Locally

1. Ensure you have Docker running (for Redis).
2. Configure your `.env` file with `DATABASE_URL`, `REDIS_URL`, and `JWT_SECRET`.
3. Start the services:
```bash
docker-compose up -d
npm install
npm run dev
```

## Documentation

Check the `/docs` directory for frontend/app integration guides:
- [Android Integration Guide](./docs/integration_android.md)
- [iOS Integration Guide](./docs/integration_ios.md)
- [Angular/Web Integration Guide](./docs/integration_angular.md)
