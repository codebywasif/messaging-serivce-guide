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
* **Direct Messaging**: 1-to-1 messaging between Customer<->Driver, Driver<->Operator.
* **Unified Operator Entity**: Operators represent the "company". If a driver messages the operator, any operator can respond, and the read state synchronizes.
* **Broadcasts**: Operators can broadcast messages to all drivers, online drivers, or all operators.
* **Message Deleting**: Operators can soft-delete messages or entire conversations, which instantly propagates via Socket.IO limits.
* **Chat History**: Infinite scrolling via `/history` endpoints.
* **Typing Indicators & Read Receipts**: Real-time status syncing.
* **Offline Handling**: Fallback mechanisms when WebSockets disconnect (FCM triggers).

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
- [Android Integration Guide](./blob/main/integration_android.md)
- [iOS Integration Guide](./docs/integration_ios.md)
- [Angular/Web Integration Guide](./docs/integration_angular.md)
