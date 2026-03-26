# Infocabs Messaging Service

A real-time chat system for taxi/ride-hailing dispatch operations, facilitating communication between **Customers**, **Operators**, and **Drivers** through a hybrid architecture combining REST APIs and WebSocket connections.

## Architecture

| Channel | Transport | Use Case |
|---|---|---|
| Driver ↔ Operator | WebSocket (Socket.IO) | Real-time communication |
| Customer ↔ System | REST API | Asynchronous via SMS, WhatsApp, Mobile App |

## Technology Stack

| Component | Technology |
|---|---|
| Runtime | Node.js with TypeScript |
| API Framework | Express.js |
| Real-time Engine | Socket.IO |
| Database | Microsoft SQL Server (via TypeORM) |
| Cache / Pub-Sub | Redis |
| Authentication | JWT (jsonwebtoken + bcrypt) |

## Getting Started

### Prerequisites

- Node.js v18+
- Redis server running
- Microsoft SQL Server instance
- Docker (optional, for containerized deployment)

### Installation

```bash
npm install
```

### Environment Variables

Create a `.env` file in the project root:

```env
PORT=3001
DATABASE_URL=mssql://sa:YourPassword@localhost:1433/messagingservice
REDIS_URL=redis://localhost:6379
JWT_SECRET=super_secret_jwt_key_here
```

### Running in Development

```bash
npm run dev
```

### Building for Production

```bash
npm run build
npm start
```

### Docker

```bash
docker-compose up
```

## Database Schema

### `msg_conversations`

| Column | Type | Description |
|---|---|---|
| `conversation_id` | BIGINT (PK) | Auto-increment primary key |
| `company_id` | BIGINT | Company identifier |
| `division_id` | BIGINT | Division identifier (sub-company) |
| `job_id` | BIGINT (nullable) | Associated job/booking number |
| `context_key` | VARCHAR(50) (nullable) | Identifier for non-job conversations (e.g., broadcast target, driver ping) |
| `conversation_type` | VARCHAR(20) | `job`, `open`, `direct`, `broadcast` |
| `is_highlighted` | BIT | Operator alert flag |
| `is_blocked` | BIT | Blocked status |
| `is_deleted` | BIT | Soft delete flag |
| `created_at` | DATETIME | Creation timestamp |
| `updated_at` | DATETIME | Last update timestamp |

### `msg_messages`

| Column | Type | Description |
|---|---|---|
| `message_id` | BIGINT (PK) | Auto-increment primary key |
| `company_id` | BIGINT | Company identifier |
| `division_id` | BIGINT | Division identifier (sub-company) |
| `conversation_id` | BIGINT (FK) | Links to conversations |
| `sender_id` | BIGINT | Sender identifier |
| `sender_type` | VARCHAR(20) | `customer`, `driver`, `operator` |
| `recipient_id` | BIGINT (nullable) | Recipient identifier |
| `recipient_type` | VARCHAR(20) | Recipient entity type |
| `content` | TEXT | Message body |
| `message_type` | VARCHAR(20) | `text`, `sms`, `whatsapp`, `image` |
| `status` | VARCHAR(20) | `sent`, `delivered`, `read` |
| `is_high_priority` | BIT | Priority flag |
| `is_deleted` | BIT | Soft delete flag |
| `metadata` | JSON (nullable) | Additional data |
| `created_at` | DATETIME | Creation timestamp |
| `delivered_at` | DATETIME (nullable) | Delivery timestamp |
| `read_at` | DATETIME (nullable) | Read timestamp |

### Conversation Types

Conversations are classified by `conversation_type` and use different identifier columns:

| Type | `job_id` | `context_key` | Description |
|---|---|---|---|
| `job` | Job number (BIGINT) | — | Two-way chat tied to a booking/job |
| `open` | — | `driver_{id}` | Driver-initiated open chat with operators |
| `broadcast` | — | `all_drivers`, `online_drivers`, `all_operators` | One-to-many operator announcement |
| `direct` | — | — | Direct message between two users |

> **Note:** All ID fields (`company_id`, `division_id`, `sender_id`, `recipient_id`, `job_id`) are **BIGINT** and must be numeric values.

## Multi-Tenancy

The system is multi-tenant, scoped by **Company** and **Division**:

- Every company has sub-companies called **divisions**
- All data (conversations, messages) is scoped to a specific `company_id` + `division_id`
- Redis keys follow the pattern: `company:{companyId}:division:{divisionId}:...`
- Socket.IO rooms follow the same pattern for isolation

### JWT Token Requirements

All JWT tokens must include:

```json
{
  "companyId": 200,
  "divisionId": 30,
  "userId": 805,
  "role": "operator"
}
```

## API Endpoints

### Chat

| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/chat/:senderType/:senderId/send` | Send a message |
| POST | `/api/chat/broadcast` | Send broadcast (operators only) |
| PUT | `/api/chat/message/:messageId/delivered` | Mark message as delivered |
| PUT | `/api/chat/message/:messageId/read` | Mark message as read |
| DELETE | `/api/chat/message/:messageId` | Delete a message (operators only) |
| DELETE | `/api/chat/conversation/:conversationId` | Delete a conversation (operators only) |
| GET | `/api/chat/offline-messages` | Get queued offline messages |

### History

| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/history/recent` | Recent conversations (all roles) |
| GET | `/api/history/job/:jobId` | Chat history for a job |
| GET | `/api/history/conversation/:conversationId/messages` | Messages for a conversation |

### Users

| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/users/block/:userId` | Block a user (operators only) |
| POST | `/api/users/unblock/:userId` | Unblock a user (operators only) |
| GET | `/api/users/online-drivers` | List online drivers |

### Webhooks

| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/webhooks/twilio/sms` | Twilio SMS webhook receiver |
| GET/POST | `/api/webhooks/whatsapp` | WhatsApp Business API webhook |

## WebSocket Events

### Client → Server

| Event | Description |
|---|---|
| `room:join` | Join a job room |
| `typing:start` | Start typing indicator |
| `typing:stop` | Stop typing indicator |

### Server → Client

| Event | Description |
|---|---|
| `message:receive` | New message received |
| `message:deleted` | Message was deleted |
| `conversation:deleted` | Conversation was deleted |
| `delivery_ack` | Message delivery confirmed |
| `read_ack` | Message read confirmed |
| `typing:status` | Typing indicator update |

## Test Web App

A React + Vite test client is included in the `test-web-app/` directory for manual testing.

```bash
cd test-web-app
npm install
npm run dev
```

The test app provides:
- Mock JWT login with Company ID, Division ID, Role, and User ID (all IDs must be numeric)
- Real-time WebSocket chat interface
- Message history browsing (available for all roles)
- Online drivers list (operators/admin only)
- Broadcast messaging (operators/admin only)
- Message deletion (operators/admin only)

## Project Structure

```
├── src/
│   ├── config/          # Database and Redis configuration
│   ├── controllers/     # Route handlers
│   ├── middlewares/      # JWT auth and role authorization
│   ├── models/          # TypeORM entities
│   ├── routes/          # Express route definitions
│   ├── services/        # Push notification service
│   ├── sockets/         # Socket.IO event handlers
│   ├── utils/           # Utility functions
│   └── index.ts         # Application entry point
├── test-web-app/        # React test client
├── tests/               # Test files
├── docs/                # Documentation
├── scripts/             # Utility scripts
├── docker-compose.yml   # Docker configuration
└── Dockerfile           # Container build file
```
