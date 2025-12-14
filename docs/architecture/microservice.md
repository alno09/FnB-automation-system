# Microservices Architecture Detail

## Service Overview

### 1. Order Service (.NET 9)
**Port**: 5001  
**Database**: PostgreSQL `orders_db`  
**Cache**: Redis (product prices, isDispensable flags)

**Responsibilities:**
- Order CRUD operations
- Status management (Pending → Paid → Completed)
- Sync calls to Product & Inventory services
- Async events to RabbitMQ

**Key Endpoints:**
- POST /api/orders
- GET /api/orders/{id}
- PATCH /api/orders/{id}
- DELETE /api/orders/{id}

**Cache Strategy:**
```csharp
// Check Redis first for product info
var price = await _redisCache.GetAsync<decimal>($"product:price:{productId}");
if (price == null) {
    price = await _productService.GetPriceAsync(productId);
    await _redisCache.SetAsync($"product:price:{productId}", price, TimeSpan.FromMinutes(5));
}
```

**Event Publishing:**
```csharp
// Publish to RabbitMQ when order completed
await _eventBus.PublishAsync(new OrderCompletedEvent {
    OrderId = order.Id,
    Items = order.Items,
    Timestamp = DateTime.UtcNow
});
```

### 2. Payment Service (.NET 9)
**Port**: 5002  
**Database**: PostgreSQL `payments_db`  

**Responsibilities:**
- Payment processing (Cash, QRIS - mock in prototype)
- Transaction logging
- Payment status tracking
- Publishes "PaymentCompleted" event to RabbitMQ

**Key Endpoints:**
- GET    /api/all                            - Get all payment
- POST   /api/Payment                        - Create payment record
- POST   /api/Payment/{PaymentId}/confirm    - Confirm payment
- GET    /api/Payment/order/{OrderId}        - Get payment status by order id

**Payment logic example:**
```csharp
public async Task<PaymentResult> ProcessPaymentAsync(Order order, PaymentInfo info) {
    // Simulate payment gateway
    var success = await _paymentService.ChargeAsync(info, order.TotalAmount);
    if (success) {
        await _eventBus.PublishAsync(new PaymentCompletedEvent {
            OrderId = order.Id,
            Amount = order.TotalAmount,
            Timestamp = DateTime.UtcNow
        });
    }
    return new PaymentResult { Success = success };
}
```

### 3. Product Service (.NET 9)
**Port**: 5003  
**Database**: PostgreSQL `products_db`  

**Responsibilities:**
- Product catalog management
- Category management
- Pricing
- Availability status
- `isDispensable` flag for QR generation

**Key Endpoints:**
- GET    /api/Product                               - List product
- GET    /api/Product/availability                  - List available product
- POST   /api/Product                               - Create product
- PUT    /api/Product/{ProductId}/availability      - Update product availability

**Model example:**
```csharp
public class Product {
    public Guid Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
    public bool IsDispensable { get; set; } // Determines if ESP32 can dispense
}
```

### 4. Inventory Service (.NET 9)
**Port**: 5004  
**Database**: PostgreSQL `inventories_db`  

**Responsibilities:**
- Real-time stock tracking
- Stock reservation during order
- Low stock alerts
- Inventory history logging
- Listens to "OrderCompleted" events

**Key Endpoints:**
- GET    /api/Inventory                        - Get stock level
- PUT    /api/Inventory/{id}                   - Update stock data
- GET    /api/StockMovement/{inventoryId}      - Get stock movement

**Event Handler:**
```csharp
public async Task Handle(OrderCompletedEvent evt) {
    foreach (var item in evt.Items) {
        await _stockManager.DecrementStockAsync(item.ProductId, item.Quantity);
    }
}
```
**Stock Movement Example:**
```csharp
public async Task DecrementStockAsync(Guid productId, int qty) {
    var stock = await _db.Stocks.FindAsync(productId);
    stock.Quantity -= qty;
    await _db.SaveChangesAsync();
}
```

### 5. QR Service (.NET 9)
**Port**: 5005  
**Database**: PostgreSQL `qr_db`  

**Responsibilities:**
- QR code generation (15-minute expiry)
- QR validation for ESP32 dispenser
- Prevent replay attacks (one-time use)

**Key Endpoints:**
- POST   /api/QrCode/generate        - Generate QrCode for order item
- POST   /api/QrCode/validate        - Validate from dispenser

**QR Generation Example:**
```csharp
public QRCode Generate(Order order) {
    var qr = new QRCode {
        Id = Guid.NewGuid(),
        OrderId = order.Id,
        ExpiresAt = DateTime.UtcNow.AddMinutes(15)
    };
    _db.QRCodes.Add(qr);
    _db.SaveChanges();
    return qr;
}
```
**QR Validation Example:**
```csharp
public bool Validate(Guid qrId) {
    var qr = _db.QRCodes.Find(qrId);
    return qr != null && qr.ExpiresAt > DateTime.UtcNow;
}
```

### 6. Analytics Service (.NET 9)
**Port**: 8000  
**Database**: PostgreSQL `analytic_db`  

**Responsibilities:**
- ETL pipeline from PostgreSQL to ClickHouse
- Data transformation and aggregation
- Analytics API for POS dashboard

**ETL Flow:**
```
PostgreSQL (all 5 services) 
    ↓ Extract (every 1 hour)
FastAPI ETL Worker
    ↓ Transform & Aggregate
ClickHouse (denormalized tables)
    ↓ Query (<100ms)
POS Analytics Dashboard
```

**Key Endpoints:**
- GET    /api/v1/reports/reports/summary          - Today's summary
- GET    /api/v1/reports/reports/favorites        - Best selling product
- GET    /api/v1/reports/reports/sales-trend      - revenue trend
- GET    /api/v1/reports/reports/recent-orders    - all order

**ETL Example:**
```python
def extract_orders():
    # Pull order events from RabbitMQ or log storage
    return events

def transform(events):
    # Aggregate sales, revenue, and top products
    return aggregated_data

def load(aggregated_data):
    # Insert into ClickHouse for analytics queries
    client.insert("analytics.orders", aggregated_data)
```
**API Example:**
```python
@app.get("/analytics/top-products")
def get_top_products():
    query = "SELECT product_id, SUM(quantity) AS total FROM orders GROUP BY product_id ORDER BY total DESC LIMIT 10"
    return client.execute(query)
```