# System Architecture Documentation

## Overview

The F&B Automation System is built using a microservices architecture pattern, enabling independent scaling, fault isolation, and technology diversity. The system consists of three main subsystems: POS tablet application, Self-Service Kiosk, and IoT Smart Dispenser, all orchestrated by six backend microservices.

---

## Architecture Diagrams

### 1. High-Level System Architecture

![System Architecture](../diagrams/system-architecture.png)

**[View Full Resolution Diagram](../diagrams/system-architecture.png)**

```
┌─────────────────────────────────────────────────────────────────────┐
│                         CLIENT LAYER                                 │
├────────────────────┬────────────────────┬────────────────────────────┤
│                    │                    │                            │
│  📱 POS Tablet     │  📱 Kiosk Tablet   │  🤖 ESP32 Dispenser       │
│  (Android)         │  (Android)         │  (IoT Device)              │
│                    │                    │                            │
│  • Order Mgmt      │  • Self-Service    │  • QR Scanning (GM65)     │
│  • Payment         │  • Menu Browse     │  • API Validation         │
│  • Products        │  • Cart            │  • LCD Display            │
│  • Analytics UI    │  • Mock Payment    │  • Pump Control           │
│  • BT Printer      │  • BT Printer      │  • WiFi Direct            │
│                    │  • Idle Screen     │                            │
│  Jetpack Compose   │  Jetpack Compose   │  Arduino/C++              │
│  MVVM Pattern      │  MVVM Pattern      │                            │
└──────────┬─────────┴──────────┬─────────┴──────────┬─────────────────┘
           │                    │                    │
           │         REST API (Direct Calls)         │
           │                    │                    │
           └────────────────────┼────────────────────┘
                                │
    ┌───────────────────────────┼───────────────────────────┐
    │                           │                           │
    │              MICROSERVICES LAYER                      │
    │           (6 Independent Services)                    │
    │                           │                           │
    ├───────────┬───────────────┼───────────────┬───────────┤
    │           │               │               │           │
┌───▼────┐  ┌──▼─────┐  ┌──────▼──────┐  ┌────▼──────┐  ┌▼────────┐
│ Order  │  │Payment │  │   Product   │  │ Inventory │  │   QR    │
│Service │  │Service │  │   Service   │  │  Service  │  │ Service │
│        │  │        │  │             │  │           │  │         │
│ :5001  │  │ :5002  │  │    :5003    │  │   :5004   │  │  :5005  │
│ .NET 9 │  │ .NET 9 │  │   .NET 9    │  │  .NET 9   │  │ .NET 9  │
└────┬───┘  └────┬───┘  └──────┬──────┘  └─────┬─────┘  └────┬────┘
     │           │             │               │             │
     │           │             │               │             │
     │    ┌──────▼─────────────▼───────────────▼─────────────▼──┐
     │    │                                                      │
     │    │            RabbitMQ Message Broker                  │
     │    │         (Event-Driven Communication)                │
     │    │                                                      │
     │    │  Exchanges:  fb_automation_events (Topic)           │
     │    │  Queues:     order_events, payment_events,          │
     │    │              inventory_events, qr_events,           │
     │    │              analytics_events                        │
     │    │                                                      │
     │    └──────┬─────────────┬───────────────┬─────────────┬──┘
     │           │             │               │             │
     └───────────┼─────────────┼───────────────┼─────────────┤
                 │             │               │             │
            ┌────▼────┐   ┌────▼────┐     ┌───▼──────┐      │
            │Analytics│   │  Redis  │     │PostgreSQL│      │
            │ Service │   │ (Cache) │     │ (5 DBs)  │      │
            │         │   │         │     │          │      │
            │  :8000  │   │  :6379  │     │  :5432   │      │
            │ FastAPI │   │         │     │          │      │
            └────┬────┘   └────┬────┘     └────┬─────┘      │
                 │             │               │            │
                 │             │               │            │
            ┌────▼─────────────▼───────────────▼────────────▼───┐
            │                                                    │
            │              DATA LAYER                            │
            │                                                    │
            ├────────────────────────┬───────────────────────────┤
            │                        │                           │
       ┌────▼─────┐            ┌────▼──────┐              ┌─────▼─────┐
       │PostgreSQL│            │   Redis   │              │ClickHouse │
       │  (OLTP)  │            │  (Cache)  │              │  (OLAP)   │
       │          │            │           │              │           │
       │ • orders │            │ Receipt   │              │ Analytics │
       │ • payments│           │  Data:    │              │  Tables:  │
       │ • products│           │           │              │           │
       │ • inventory│          │ • receipt:│              │ • sales_  │
       │ • qr_codes│           │   {id}    │              │   fact    │
       │          │            │           │              │ • product_│
       │ Separate │            │ Product   │              │   perf    │
       │ DB per   │            │  Cache:   │              │ • inventory│
       │ service  │            │           │              │   _snapshot│
       │          │            │ • product:│              │           │
       │          │            │   price:  │              │ Columnar  │
       │          │            │   {id}    │              │ Storage   │
       │          │            │ • product:│              │ Fast OLAP │
       │          │            │   disp:{id}│             │ Queries   │
       └──────────┘            └───────────┘              └───────────┘
```

**Diagram Files:**
- **Source (Editable)**: `docs/diagrams/system-architecture.drawio` or `.excalidraw`
- **High-Res PNG**: `docs/diagrams/system-architecture.png`
- **PDF Version**: `docs/diagrams/system-architecture.pdf`

---

### 2. Complete System Communication Flow

![Communication Flow](../diagrams/communication-flow.png)

**[View Full Resolution Diagram](../diagrams/communication-flow.png)**

#### System Startup

```
┌─────────────┐
│   Product   │  On Service Start
│   Service   │  ─────────────────►  Cache all products to Redis
│             │                      • product:price:{id}
└─────────────┘                      • product:dispensable:{id}
```

#### Order Creation to Receipt Generation Flow

```
┌──────────┐
│POS/Kiosk │
│  User    │
└─────┬────┘
      │
      │ 1. POST /api/Order
      │    { items: [{productId, quantity}], totalAmount }
      ▼
┌─────────────┐
│   Order     │  2. Validate total amount
│   Service   │     ─────────────────────►  ┌───────┐
│             │  3. Get cached product data │ Redis │
│             │◄────────────────────────────┤       │
│             │     (price, isDispensable)  └───────┘
│             │
│             │  4. Calculate server-side total
│             │     Compare with client totalAmount
│             │     If mismatch → reject order
│             │
│             │  5. Create order record in PostgreSQL
│             │     status: PENDING
│             │
│             │  6. Create receipt in Redis    ┌───────┐
│             │────────────────────────────────►│ Redis │
│             │     receipt:{orderId}           │       │
│             │     {                           │receipt│
│             │       orderId,                  │ :id   │
│             │       items: [...],             │       │
│             │       total,                    │PENDING│
│             │       paymentStatus: "PENDING", │qr:null│
│             │       qrCode: null              │       │
│             │     }                           └───────┘
│             │
│             │  7. Check if any item isDispensable
│             │     If YES → Publish to RabbitMQ
│             │
│             │  8. Publish "OrderCreated" event
│             │     (for items with isDispensable=true)
└──────┬──────┘
       │          ┌─────────────────────┐
       └─────────►│     RabbitMQ        │
                  │                     │
                  └──────────┬──────────┘
                             │
                             │ OrderCreated event
                             │ { orderId, dispensableItems }
                             ▼
                       ┌─────────┐
                       │   QR    │  9. Consumer receives event
                       │ Service │
                       │         │  10. Generate QR code
                       │         │      code: "QR-20241213-ABC"
                       │         │      expiresAt: now + 15 min
                       │         │
                       │         │  11. Save to PostgreSQL
                       │         │
                       │         │  12. Update receipt in Redis
                       │         │      ──────────────────────►  ┌───────┐
                       │         │      receipt:{orderId}       │ Redis │
                       │         │      {                       │       │
                       │         │        ...existing data,     │receipt│
                       │         │        qrCode: "QR-xxx",     │ :id   │
                       │         │        qrExpiresAt: "..."    │       │
                       │         │      }                       │PENDING│
                       └─────────┘                              │qr:YES │
                                                                └───────┘
                             │
       ┌─────────────────────┘
       │
       │ 13. Return order to client
       │     { orderId, status: "PENDING" }
       ▼
┌──────────┐
│   POS    │  Order created successfully
│  /Kiosk  │  Now proceed to payment...
└──────────┘
```

#### Payment Processing Flow

```
┌──────────┐
│POS/Kiosk │
│  User    │  14. User chooses payment method
└─────┬────┘      (Cash / QRIS)
      │
      │ 15. POST /api/Payment
      │     {
      │       orderId,
      │       method: "QRIS",
      │       amount
      │     }
      ▼
┌─────────────┐
│  Payment    │  16. Create payment record
│  Service    │      in PostgreSQL
│             │
│             │  17. Validate payment (MOCK)
│             │      - Simulate payment gateway
│             │      - Return success
│             │
│             │  18. Publish "PaymentCompleted" event
│             │      to RabbitMQ
│             │      { orderId, status: "PAID" }
│             │
│             │  19. Update receipt in Redis
│             │      ──────────────────────►  ┌───────┐
│             │      receipt:{orderId}       │ Redis │
│             │      {                       │       │
│             │        ...existing data,     │receipt│
│             │        paymentStatus: "PAID",│ :id   │
│             │        paymentMethod: "QRIS",│       │
│             │        paidAt: timestamp     │ PAID  │
│             │      }                       │qr:YES │
└──────┬──────┘                              └───────┘
       │
       │ 20. Return complete receipt data to client
       │     (includes QR code if dispensable)
       ▼
┌──────────┐      ┌─────────────────────┐
│   POS    │      │     RabbitMQ        │
│  /Kiosk  │      │                     │
└─────┬────┘      └──────────┬──────────┘
      │                      │
      │                      │ PaymentCompleted event
      │                      ▼
      │                 ┌─────────┐
      │                 │ Order   │  21. Consumer receives event
      │                 │ Service │
      │                 │         │  22. Update order status
      │                 │         │      PENDING → PROCESSING
      │                 │         │      in PostgreSQL
      │                 └─────────┘
      │
      │ 23. Format receipt for thermal printer
      │     - Order number
      │     - Items list
      │     - Total amount
      │     - Payment method
      │     - QR code (if dispensable item)
      │     - Timestamp
      │
      │ 24. Send to Bluetooth thermal printer
      ▼
┌──────────────┐
│   Bluetooth  │
│   Thermal    │  25. Print receipt
│   Printer    │
└──────────────┘
         │
         │ 26. Receipt printed
         ▼
    📄 Physical Receipt
         │
         │ 27. User takes receipt
         ▼
    👤 Customer
```

#### Dispenser Flow

```
    👤 Customer
         │
         │ 28. If receipt has QR code
         │     Customer goes to dispenser
         ▼
┌─────────────────┐
│   ESP32         │
│   Dispenser     │
│                 │  Current state: IDLE
│   LCD: "READY  │
│         SCAN QR"│
└────────┬────────┘
         │
         │ 29. Customer scans QR code
         ▼
┌─────────────────┐
│   GM65 QR       │  30. Scanner reads QR code
│   Scanner       │      Sends data via UART to ESP32
│   (UART)        │      data: "QR-20241213-ABC"
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   ESP32         │  31. LCD: "Scanning..."
│                 │
│                 │  32. Parse QR code string
│                 │
│                 │  33. HTTP POST request via WiFi
│                 │      POST /api/QrCode/validate
│                 │      { code: "QR-20241213-ABC" }
│                 │
│                 │  ────────────────────────►
└─────────────────┘
                              │
                              ▼
                        ┌─────────┐
                        │   QR    │  34. Validate QR code
                        │ Service │      - Check exists in DB
                        │         │      - Check not expired (<15 min)
                        │         │      - Check not used (status!=USED)
                        │         │
                        │         │  35. If VALID:
                        │         │      Return { 
                        │         │        valid: true,
                        │         │        productName: "...",
                        │         │        dispenseDuration: 8
                        │         │      }
                        │         │
                        │         │  36. Mark QR as USED in DB
                        │         │      (prevent replay attack)
                        └────┬────┘
                             │
                             │ Response
         ┌───────────────────┘
         │
         ▼
┌─────────────────┐
│   ESP32         │  37. Receive validation response
│                 │
│                 │  38. If valid = true:
│                 │      - State: READY_TO_DISPENSE
│                 │      - LCD: "Place cup"
│                 │
│                 │  39. Check ultrasonic sensor
│                 │      ──────────────────►  ┌──────────────┐
│                 │                           │  Ultrasonic  │
│                 │◄──────────────────────────│   Sensor     │
│                 │      distance < 10cm?     └──────────────┘
│                 │
│                 │  40. If cup detected:
│                 │      - State: DISPENSING
│                 │      - LCD: "Dispensing..."
│                 │
│                 │  41. Activate relay
│                 │      ─────────────►  ┌──────────┐
│                 │                     │  Relay   │
│                 │                     │  Module  │
│                 │                     └────┬─────┘
│                 │                          │
│                 │                          │ ON
│                 │                          ▼
│                 │                     ┌──────────┐
│                 │                     │ Diaphragm│
│                 │                     │   Pump   │
│                 │                     │  (12V)   │
│                 │                     └────┬─────┘
│                 │                          │
│                 │  42. Dispense for X seconds    │
│                 │      (from API response)       │
│                 │      delay(duration * 1000)    │
│                 │                          │
│                 │                     ┌────▼─────┐
│                 │                     │  Liquid  │
│                 │                     │  Output  │
│                 │                     └────┬─────┘
│                 │                          │
│                 │                          ▼
│                 │                      ☕ Beverage
│                 │
│                 │  43. Deactivate relay
│                 │      Pump OFF
│                 │
│                 │  44. LCD: "Complete!"
│                 │
│                 │  45. After 3 seconds
│                 │      - State: IDLE
│                 │      - LCD: "READY SCAN QR"
│                 │
└─────────────────┘


If validation fails (invalid/expired/used):
┌─────────────────┐
│   ESP32         │  - LCD: "Invalid QR!"
│                 │  - Display error for 5 sec
│                 │  - Return to IDLE
└─────────────────┘
```

#### Background: Analytic ETL

```
┌─────────────────────┐
│  Analytics Service  │  Running on schedule (every 1 hour)
│     (FastAPI)       │
│                     │
│  1. Extract data    │  ──────────────►  ┌──────────────┐
│     from PostgreSQL │                   │ PostgreSQL   │
│     - orders_db     │◄──────────────────│ (5 databases)│
│     - payments_db   │                   └──────────────┘
│     - products_db   │
│     - inventory_db  │
│     - qr_db         │
│                     │
│  2. Transform       │
│     - Join tables   │
│     - Aggregate     │
│     - Denormalize   │
│                     │
│  3. Load to         │  ──────────────►  ┌──────────────┐
│     ClickHouse      │                   │  ClickHouse  │
│                     │                   │  (Analytics) │
└─────────────────────┘                   └──────┬───────┘
                                                 │
                                                 │
                                          Fast OLAP Queries
                                                 │
                                                 ▼
                                          ┌──────────────┐
                                          │     POS      │
                                          │  Analytics   │
                                          │  Dashboard   │
                                          └──────────────┘
```

### 3. Data Flow Diagram

![Data Flow](../diagrams/data-flow.png)

**[View Full Resolution Diagram](../diagrams/data-flow.png)**

```
┌─────────────────────────────────────────────────────────────┐
│                    TRANSACTIONAL DATA                        │
│                   (PostgreSQL - OLTP)                        │
├─────────────┬──────────────┬─────────────┬──────────────────┤
│  orders_db  │ payments_db  │ products_db │  inventories_db  │
│             │              │             │                  │
│ • orders    │ • payments   │ • products  │ • inventories    │
│ • order_    │ • trans-     │ • categories│ • stock_moves    │
│   items     │   actions    │             │                  │
└──────┬──────┴──────┬───────┴──────┬──────┴────────┬─────────┘
       │             │              │               │
       │ + qr_db (qr_codes, dispense_logs)         │
       │             │              │               │
       │   ┌─────────▼──────────────▼───────────────▼─────┐
       │   │                                               │
       │   │       ETL Pipeline (FastAPI Analytics Svc)    │
       │   │                                               │
       │   │  Schedule: Every 1 hour                      │
       │   │  Process:                                     │
       │   │   1. Extract new/updated records              │
       │   │   2. Join across databases                    │
       │   │   3. Transform to denormalized format         │
       │   │   4. Calculate aggregates                     │
       │   │   5. Load to ClickHouse                       │
       │   │                                               │
       │   └───────────────────┬───────────────────────────┘
       │                       │
       │                       ▼
       │   ┌─────────────────────────────────────────────┐
       │   │       ANALYTICAL DATA                       │
       │   │      (ClickHouse - OLAP)                    │
       │   │                                             │
       │   │  • sales_fact (denormalized)                │
       │   │  • product_performance                      │
       │   │  • inventory_snapshot                       │
       │   │  • dispense_analytics                       │
       │   │                                             │
       │   │  Optimizations:                             │
       │   │  - Columnar storage                         │
       │   │  - Data compression (10x)                   │
       │   │  - Partitioning by date                     │
       │   │  - Parallel query execution                 │
       │   │                                             │
       │   └────────────────────┬────────────────────────┘
       │                        │
       │                        │ Fast Queries
       │                        ▼
       │                 ┌─────────────┐
       │                 │  Analytics  │
       │                 │   Service   │
       │                 │   (FastAPI) │
       │                 └──────┬──────┘
       │                        │
       │                        │ REST API
       │                        ▼
       │                 ┌─────────────┐
       │                 │ POS App     │
       │                 │ Analytics   │
       │                 │ Dashboard   │
       │                 └─────────────┘
       │
       │   ┌─────────────────────────────────────────────┐
       │   │           CACHE LAYER (Redis)               │
       │   │                                             │
       │   │  Purpose: Fast access & data sharing       │
       │   │                                             │
       │   │  Product Cache (on service startup):       │
       │   │  • product:price:{id} → decimal            │
       │   │  • product:dispensable:{id} → boolean      │
       │   │  TTL: Forever (manual refresh)             │
       │   │                                             │
       │   │  Receipt Data (dynamic):                   │
       │   │  • receipt:{orderId} → JSON                │
       │   │    - Created by Order Service              │
       │   │    - Updated by Payment Service            │
       │   │    - Updated by QR Service                 │
       │   │  TTL: 24 hours                             │
       │   │                                             │
       └───┴─────────────────────────────────────────────┘
```

---

### 4. Docker Container Architecture

![Docker Architecture](../diagrams/docker-architecture.png)

**[View Full Resolution Diagram](../diagrams/docker-architecture.png)**

```
┌────────────────────────────────────────────────────────────────┐
│                    Docker Host (Local Dev)                      │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────── Docker Network: fb_network ───────────────┐ │
│  │                                                             │ │
│  │  ┌─────────────────────────────────────────────────────┐  │ │
│  │  │         Application Services                        │  │ │
│  │  ├──────────────┬──────────────┬──────────────────────┤  │ │
│  │  │ order-svc    │ payment-svc  │ product-svc          │  │ │
│  │  │ :5001        │ :5002        │ :5003                │  │ │
│  │  │ .NET 9       │ .NET 9       │ .NET 9               │  │ │
│  │  ├──────────────┼──────────────┼──────────────────────┤  │ │
│  │  │ inventory-svc│ qr-svc       │ analytics-svc        │  │ │
│  │  │ :5004        │ :5005        │ :8000                │  │ │
│  │  │ .NET 9       │ .NET 9       │ FastAPI              │  │ │
│  │  └──────────────┴──────────────┴──────────────────────┘  │ │
│  │                         │                                 │ │
│  │  ┌─────────────────────┼──────────────────────────────┐  │ │
│  │  │    Infrastructure Services                         │  │ │
│  │  ├─────────────────────────────────────────────────────┤  │ │
│  │  │  PostgreSQL (:5432)                                 │  │ │
│  │  │  ├─ orders_db                                       │  │ │
│  │  │  ├─ payments_db                                     │  │ │
│  │  │  ├─ products_db                                     │  │ │
│  │  │  ├─ inventory_db                                    │  │ │
│  │  │  └─ qr_db                                           │  │ │
│  │  │                                                      │  │ │
│  │  │  Redis (:6379)                                      │  │ │
│  │  │  └─ Cache & Receipt storage                        │  │ │
│  │  │                                                      │  │ │
│  │  │  RabbitMQ (:5672, :15672)                          │  │ │
│  │  │  └─ Message broker                                  │  │ │
│  │  │                                                      │  │ │
│  │  │  ClickHouse (:8123, :9000)                         │  │ │
│  │  │  └─ Analytics database                              │  │ │
│  │  └──────────────────────────────────────────────────────┘  │ │
│  │                                                             │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                 │
│  Volume Mounts:                                                 │
│  • postgres_data → /var/lib/postgresql/data                    │
│  • redis_data → /data                                           │
│  • rabbitmq_data → /var/lib/rabbitmq                           │
│  • clickhouse_data → /var/lib/clickhouse                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
         │                    │                    │
         │                    │                    │
    ┌────▼────┐          ┌────▼────┐         ┌────▼────┐
    │ POS App │          │  Kiosk  │         │  ESP32  │
    │ (WiFi)  │          │  (WiFi) │         │ (WiFi)  │
    └─────────┘          └─────────┘         └─────────┘
```

---

## Architecture Principles

### 1. Microservices Architecture

**Why Microservices?**
- **Independent Scaling**: Each service scales based on its load
  - Order Service: 3-5 instances (high traffic)
  - QR Service: 2 instances (burst during peak hours)
  - Analytics: 1 instance (scheduled jobs)
  
- **Technology Diversity**: Use the right tool for the job
  - .NET 9 for business services (performance, type safety)
  - FastAPI for analytics (Python data processing libraries)
  
- **Fault Isolation**: One service failure doesn't crash the system
  - If Analytics crashes, orders still process
  - If QR Service is slow, doesn't affect order creation
  
- **Team Scalability**: Different teams can own different services (future)

### 2. Event-Driven Architecture

**Asynchronous Communication via RabbitMQ:**

- **Decoupling**: Services don't need to know about each other
- **Resilience**: Messages queued if consumer is temporarily down
- **Scalability**: Multiple consumers can process messages in parallel
- **Audit Trail**: All events logged for debugging

**Example Events:**
```
PaymentCompleted → Order, Inventory, Analytics
OrderCompleted → QR, Inventory, Analytics
InventoryLow → Notification (future)
```

### 3. Cache-Aside Pattern

**Redis for Performance:**

```
1. Application checks cache first
2. Cache miss → fetch from source (database/service)
3. Store in cache for future requests
4. Set appropriate TTL (Time To Live)
```

**Benefits:**
- Reduces database load by 80%+
- Improves response time (Redis: <1ms, PostgreSQL: 10-50ms)
- Reduces network calls between services

### 4. Database Per Service

**Each service owns its data:**

- **Autonomy**: Service can change its schema without affecting others
- **Technology Choice**: Can use different databases if needed
- **Scalability**: Can optimize/scale databases independently
- **Fault Isolation**: Database failure affects only one service

**Trade-off:**
- Need ETL for cross-service analytics
- Eventual consistency instead of ACID transactions across services

---

## Technology Decisions

### Backend: .NET 9

**Why .NET?**
- ✅ High performance (faster than Node.js, Python)
- ✅ Strong typing (fewer runtime errors)
- ✅ Mature ecosystem (libraries, tools)
- ✅ Excellent async/await support
- ✅ Built-in dependency injection
- ✅ Good Docker support

### Analytics: .NET 9

**Why .NET for Analytics Service?**
- ✅ Consistency with other services (same stack)
- ✅ High performance ETL processing
- ✅ Strong typing for data transformations
- ✅ Good ClickHouse .NET client (ClickHouse.Client)
- ✅ Easy database access (Dapper, EF Core)
- ✅ Background job scheduling (Hangfire, Quartz.NET)

**ETL Schedule:** Every 1 hour (configurable)

### Mobile: Jetpack Compose (Kotlin)

**Why Compose?**
- ✅ Modern declarative UI (less boilerplate than XML)
- ✅ Type-safe (Kotlin)
- ✅ Better performance (optimized rendering)
- ✅ Code reusability between POS and Kiosk
- ✅ Material Design 3 support

### Analytics DB: ClickHouse

**Why ClickHouse over PostgreSQL?**
- ✅ 100-1000x faster for analytical queries
- ✅ Columnar storage (only read needed columns)
- ✅ Excellent compression (10x data reduction)
- ✅ Parallel query execution
- ✅ Perfect for time-series data

**Example Performance:**
```
Query: Sales by product for last 30 days (1M rows)
PostgreSQL: 5-8 seconds
ClickHouse: 72 milliseconds
```

### Message Queue: RabbitMQ

**Why RabbitMQ?**
- ✅ Reliable message delivery
- ✅ Easy Docker setup
- ✅ Good .NET client (MassTransit, RabbitMQ.Client)
- ✅ Management UI included
- ✅ Topic-based routing

### IoT: ESP32 + Arduino

**Why ESP32?**
- ✅ Built-in WiFi
- ✅ Low cost (~$3-5)
- ✅ Large community support
- ✅ Arduino compatibility (easy development)
- ✅ Multiple GPIOs for sensors/actuators

---

## Scalability Considerations

### Current (MVP)
- All services on single Docker host (local dev)
- Sufficient for 10-50 orders/day
- ~10 concurrent users

### Future (Production)

**Horizontal Scaling:**
```
Load Balancer
├─ Order Service (3 instances)
├─ Payment Service (2 instances)
├─ Product Service (2 instances)
├─ Inventory Service (2 instances)
├─ QR Service (2 instances)
└─ Analytics Service (1 instance)
```

**Database Scaling:**
- PostgreSQL: Read replicas for heavy services
- ClickHouse: Distributed tables across nodes
- Redis: Redis Cluster for high availability

**Infrastructure:**
- Deploy to Kubernetes
- Auto-scaling based on CPU/memory
- API Gateway (Kong/NGINX)