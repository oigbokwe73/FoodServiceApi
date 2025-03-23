Based on the solution you described, we'll need to design a relational SQL database that supports the client's logistics, order, delivery, and IoT monitoring capabilities. Here's a breakdown of the SQL database tables required to meet the needs of:

1. **Customers placing orders via web or call center**  
2. **Drivers delivering goods using IoT-connected devices**  
3. **Bluetooth temperature sensors sending real-time readings every 10 seconds**  
4. **Analytics and monitoring to identify deliveries with abnormal temperatures**

---

## ✅ Database Tables (with relationships)

### 1. `Customers`
Stores information about each customer.

```sql
CREATE TABLE Customers (
    CustomerID INT PRIMARY KEY IDENTITY,
    Name NVARCHAR(100),
    BusinessType NVARCHAR(50),
    Address NVARCHAR(255),
    City NVARCHAR(50),
    PostalCode NVARCHAR(20),
    PhoneNumber NVARCHAR(20),
    Email NVARCHAR(100)
);
```

---

### 2. `Orders`
Tracks individual orders placed by customers.

```sql
CREATE TABLE Orders (
    OrderID INT PRIMARY KEY IDENTITY,
    CustomerID INT FOREIGN KEY REFERENCES Customers(CustomerID),
    OrderDate DATETIME,
    OrderChannel NVARCHAR(20), -- 'Web' or 'Call Center'
    TotalAmount DECIMAL(10,2),
    DeliveryStatus NVARCHAR(50),
    ScheduledDeliveryTime DATETIME
);
```

---

### 3. `OrderItems`
Stores items associated with each order.

```sql
CREATE TABLE OrderItems (
    OrderItemID INT PRIMARY KEY IDENTITY,
    OrderID INT FOREIGN KEY REFERENCES Orders(OrderID),
    ItemName NVARCHAR(100),
    Quantity INT,
    Unit NVARCHAR(10), -- e.g., 'kg', 'ltr'
    PricePerUnit DECIMAL(10,2)
);
```

---

### 4. `Drivers`
Stores driver information who deliver the orders.

```sql
CREATE TABLE Drivers (
    DriverID INT PRIMARY KEY IDENTITY,
    Name NVARCHAR(100),
    PhoneNumber NVARCHAR(20),
    VehicleNumber NVARCHAR(20)
);
```

---

### 5. `Deliveries`
Represents each delivery attempt or session.

```sql
CREATE TABLE Deliveries (
    DeliveryID INT PRIMARY KEY IDENTITY,
    OrderID INT FOREIGN KEY REFERENCES Orders(OrderID),
    DriverID INT FOREIGN KEY REFERENCES Drivers(DriverID),
    DeliveryStartTime DATETIME,
    DeliveryEndTime DATETIME,
    DeliveryStatus NVARCHAR(50)
);
```

---

### 6. `BluetoothDevices`
Stores IoT device metadata (printer or temperature sensor).

```sql
CREATE TABLE BluetoothDevices (
    DeviceID INT PRIMARY KEY IDENTITY,
    DeviceType NVARCHAR(50), -- 'TemperatureSensor', 'Printer'
    DeviceSerialNumber NVARCHAR(100),
    VehicleNumber NVARCHAR(20), -- Mounted truck ID
    Status NVARCHAR(50) -- 'Active', 'Inactive'
);
```

---

### 7. `TemperatureReadings`
Stores real-time temperature data from the sensors every 10 seconds.

```sql
CREATE TABLE TemperatureReadings (
    ReadingID BIGINT PRIMARY KEY IDENTITY,
    DeviceID INT FOREIGN KEY REFERENCES BluetoothDevices(DeviceID),
    DeliveryID INT FOREIGN KEY REFERENCES Deliveries(DeliveryID),
    RecordedTime DATETIME,
    TemperatureCelsius DECIMAL(5,2),
    IsOutOfRange BIT -- 1 if outside optimal range
);
```

---

### 8. `TemperatureThresholds`
Stores optimal and threshold temperature ranges.

```sql
CREATE TABLE TemperatureThresholds (
    ThresholdID INT PRIMARY KEY IDENTITY,
    MinTemp DECIMAL(5,2),
    MaxTemp DECIMAL(5,2),
    EffectiveFrom DATETIME,
    EffectiveTo DATETIME NULL
);
```

---

### 9. `Alerts`
Stores any temperature alert records for analytics and notification.

```sql
CREATE TABLE Alerts (
    AlertID INT PRIMARY KEY IDENTITY,
    ReadingID BIGINT FOREIGN KEY REFERENCES TemperatureReadings(ReadingID),
    AlertTime DATETIME,
    TemperatureCelsius DECIMAL(5,2),
    AlertMessage NVARCHAR(255)
);
```

---

## 📊 Relationships Summary
- `Customers` → `Orders` → `OrderItems`
- `Orders` → `Deliveries` → `TemperatureReadings`
- `Drivers` assigned to `Deliveries`
- `BluetoothDevices` mounted in trucks send `TemperatureReadings`
- `TemperatureReadings` compared against `TemperatureThresholds`
- Out-of-threshold readings generate `Alerts`

---

Let me know if you want:
- Sample data inserts
- Power BI schema for visualizing sensor readings
- Stored procedures for recording and alerting on sensor data
- Real-time ingestion pipeline using Azure Functions or IoT Hub

Want a diagram to go along with this?

Great! Since you want to **use Azure Service Bus and Azure Blob Storage** in the solution, we’ll adjust the architecture to incorporate:

- **Azure Service Bus** for real-time **temperature reading ingestion**  
- **Azure Blob Storage** for long-term **cold storage of raw sensor data**

Let me show you how the system flows and then update the SQL schema accordingly. We'll keep the relational structure but add integration points with Azure services.

---

## ✅ Updated IoT Delivery Tracking Solution – Azure-Based Architecture

### 🧭 Architecture Overview

```mermaid
graph TD
    A[Temperature Sensor (IoT Device)] -->|Every 10s| B[Azure Function - Ingest]
    B --> C[Azure Service Bus Topic]
    C --> D[Azure Function - DB Writer]
    D --> E[Azure SQL Database]
    B --> F[Azure Blob Storage - Raw JSON Logs]
    E --> G[Power BI or Azure Data Explorer]
    E --> H[Alerts Table for Anomaly Detection]
```

---

### 🔹 Key Azure Services

| Service | Role |
|--------|------|
| **Azure Service Bus** | Temporarily buffers real-time temperature readings |
| **Azure Blob Storage** | Stores raw telemetry data (JSON) for long-term, low-cost storage |
| **Azure Functions** | Serverless handlers for ingestion, processing, and database writing |
| **Azure SQL Database** | Main transactional database for analytics/reporting |
| **Power BI** | Dashboards and visual insights |
| **Azure Monitor / Logic App** | Optionally used for triggering alerts |

---

## 📦 Database Tables (Updated with Azure Metadata)

### 1. `TemperatureReadings` – Updated for Azure Ingestion

```sql
CREATE TABLE TemperatureReadings (
    ReadingID BIGINT PRIMARY KEY IDENTITY,
    DeliveryID INT FOREIGN KEY REFERENCES Deliveries(DeliveryID),
    DeviceID INT FOREIGN KEY REFERENCES BluetoothDevices(DeviceID),
    ServiceBusMessageID UNIQUEIDENTIFIER, -- message tracking
    RecordedTime DATETIME,
    TemperatureCelsius DECIMAL(5,2),
    IsOutOfRange BIT,
    BlobUri NVARCHAR(2083) -- link to raw JSON stored in Azure Blob
);
```

### 2. `BlobStorageLogs`

Optional metadata tracking of blobs stored (for analytics or cost tracking).

```sql
CREATE TABLE BlobStorageLogs (
    BlobID INT PRIMARY KEY IDENTITY,
    BlobUri NVARCHAR(2083),
    BlobContainer NVARCHAR(100),
    BlobName NVARCHAR(255),
    ContentType NVARCHAR(50),
    UploadedOn DATETIME,
    DeviceID INT,
    DeliveryID INT
);
```

### 3. `ServiceBusLogs`

To log message flow and diagnostics from Azure Service Bus.

```sql
CREATE TABLE ServiceBusLogs (
    LogID INT PRIMARY KEY IDENTITY,
    MessageID UNIQUEIDENTIFIER,
    EnqueuedTimeUtc DATETIME,
    ProcessedTimeUtc DATETIME,
    Status NVARCHAR(50), -- Success, Failed, Duplicate, etc.
    ErrorMessage NVARCHAR(1000) NULL
);
```

---

## 🛠 How It Works – End to End Flow

1. **IoT device** captures temperature every 10 seconds.
2. Azure Function picks it up and:
   - Publishes JSON message to **Azure Service Bus Topic**.
   - Simultaneously logs the raw JSON into **Azure Blob Storage**.
3. Another Azure Function (triggered by Service Bus):
   - Reads the message.
   - Inserts into `TemperatureReadings`, updates `BlobUri`, logs in `ServiceBusLogs`.
4. Optional: An alert rule runs that:
   - Checks if `IsOutOfRange = 1`
   - Creates entries in `Alerts` table
   - Sends email or SMS via Logic App

---

## 📊 Reporting via Power BI

Power BI connects to:
- **Azure SQL** for live metrics, trends, delivery exceptions
- **Blob Storage** (via Dataflow or Synapse) for historical trend analysis

---

Would you like:
- Sample JSON payload for Service Bus message?
- Azure Function code to process messages and insert into SQL?
- Bicep/Terraform deployment template for the resources?

Let me know! I can also generate a full animated diagram or deployment flow if that helps.
