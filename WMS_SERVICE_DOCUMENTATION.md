# ![[Pasted image 20250816171101.png|23]] Omniful WMS Service Documentation

## Table of Contents
1. [Overview](#overview)
2. [Architecture & Design](#architecture--design)
3. [Core Functionality](#core-functionality)
4. [API Endpoints](#api-endpoints)
5. [Data Models & Collections](#data-models--collections)
6. [Worker System](#worker-system)
7. [Integration Patterns](#integration-patterns)
8. [Configuration](#configuration)
9. [Development & Deployment](#development--deployment)
10. [Common Use Cases](#common-use-cases)
11. [Troubleshooting](#troubleshooting)

---

## Overview

The **WMS Service** (Warehouse Management Service) is a critical microservice in the Omniful platform responsible for managing all warehouse operations, inventory management, order fulfillment, and warehouse logistics.

### Key Responsibilities
- **Warehouse Operations**: Manage warehouse layout, zones, aisles, racks, shelves, and bins
- **Inventory Management**: Track inventory levels, locations, batches, and stock movements
- **Order Fulfillment**: Handle picking, packing, and shipping operations
- **Goods Receipt**: Process incoming goods, GRN (Goods Receipt Note) management
- **Cycle Counting**: Manage inventory audits and stock verification
- **Batch Management**: Handle batch tracking, expiry dates, and serialization
- **Location Management**: Manage SKU-to-location mappings and storage optimization
- **Reporting**: Generate warehouse reports and analytics
- **Barcode Management**: Handle barcode generation and scanning operations

### Service Characteristics
- **Database**: PostgreSQL (primary), Redis (caching)
- **Message Queue**: AWS SQS for asynchronous processing
- **Event Streaming**: Kafka for real-time updates
- **API Style**: RESTful HTTP APIs with JWT authentication
- **Multi-mode Operation**: HTTP server, worker mode, and migration mode
- **File Processing**: Excel, CSV, and PDF generation capabilities

---

## Architecture & Design

### Service Architecture
```
┌─────────────────────────────────────────────────────────────┐
│                   WMS Service                               │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │   HTTP API  │  │   Workers   │  │  Migration  │        │
│  │   Server    │  │   System    │  │   System    │        │
│  └─────────────┘  └─────────────┘  └─────────────┘        │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │ Controllers │  │   Services  │  │   Models    │        │
│  │             │  │             │  │             │        │
│  └─────────────┘  └─────────────┘  └─────────────┘        │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │ PostgreSQL  │  │    Redis    │  │    Kafka    │        │
│  │             │  │             │  │             │        │
│  └─────────────┘  └─────────────┘  └─────────────┘        │
└─────────────────────────────────────────────────────────────┘
```

### Directory Structure
```
wms-service/
├── main.go                           # Service entry point
├── configs/                          # Configuration files
├── router/                           # API routing
├── internal/                         # Core business logic
│   ├── aisle/                       # Aisle management
│   ├── attributes/                  # SKU attributes
│   ├── barcodes/                    # Barcode operations
│   ├── batch/                       # Batch management
│   ├── bin/                         # Bin management
│   ├── consolidation/               # Order consolidation
│   ├── controllers/                 # HTTP controllers
│   ├── cycle_count/                 # Cycle counting
│   ├── domain/                      # Domain models
│   ├── events/                      # Event handling
│   ├── floor/                       # Floor management
│   ├── gate_entry/                  # Gate entry operations
│   ├── grn/                         # Goods Receipt Note
│   ├── hub/                         # Hub/warehouse management
│   ├── hub_inventory/               # Hub inventory tracking
│   ├── hub_layout/                  # Hub layout management
│   ├── hub_skus/                    # Hub SKU management
│   ├── inventory_logs/              # Inventory audit logs
│   ├── inwarding_logs/              # Inwarding operations
│   ├── location/                    # Location management
│   ├── location_inventory/          # Location-based inventory
│   ├── location_sku/                # SKU-location mapping
│   ├── middlewares/                 # HTTP middlewares
│   ├── non_serialised_sku_barcodes/ # Non-serialized barcodes
│   ├── packing_tags/                # Packing operations
│   ├── picklist/                    # Picking operations
│   ├── purchase_order/              # Purchase order management
│   ├── rack/                        # Rack management
│   ├── reason/                      # Reason codes
│   ├── report/                      # Reporting system
│   ├── serialised_skus/             # Serialized SKU management
│   ├── shelf/                       # Shelf management
│   ├── station/                     # Workstation management
│   ├── stock_ownership_transfer/    # Stock ownership transfers
│   ├── stock_transfer_order/        # Stock transfer orders
│   ├── supplier/                    # Supplier management
│   ├── utils/                       # Utility functions
│   ├── warehouse_order/             # Warehouse order management
│   ├── warehouse_sku/               # Warehouse SKU management
│   ├── warehousecart/               # Warehouse cart operations
│   ├── wave/                        # Wave management
│   ├── workers/                     # Background job processors
│   └── zone/                        # Zone management
├── pkg/                             # Shared packages
└── deployment/                      # Deployment configurations
```

---

## Core Functionality

### 1. Warehouse Layout Management
- **Zone Management**: Create and manage warehouse zones
- **Floor Management**: Handle multi-floor warehouse layouts
- **Aisle Management**: Organize warehouse aisles
- **Rack Management**: Manage rack structures and configurations
- **Shelf Management**: Handle shelf organization and capacity
- **Bin Management**: Manage storage bins and locations
- **Station Management**: Handle workstation configurations

### 2. Inventory Management
- **Location Inventory**: Track inventory at specific locations
- **Hub Inventory**: Manage inventory across warehouse hubs
- **Batch Tracking**: Handle batch-based inventory management
- **Serialization**: Manage serialized SKUs and tracking
- **Stock Transfers**: Handle stock movements between locations
- **Stock Ownership**: Manage stock ownership transfers
- **Safety Stock**: Configure and monitor safety stock levels

### 3. Order Fulfillment
- **Picklist Management**: Generate and manage picking lists
- **Wave Management**: Create and manage picking waves
- **Packing Operations**: Handle order packing and consolidation
- **Cart Management**: Manage warehouse carts for operations
- **Order Processing**: Handle warehouse order workflows

### 4. Goods Receipt
- **GRN Management**: Process Goods Receipt Notes
- **Inwarding Operations**: Handle incoming goods processing
- **Gate Entry**: Manage warehouse gate operations
- **Supplier Management**: Handle supplier relationships
- **Purchase Order Integration**: Process purchase orders

### 5. Quality Control
- **Cycle Counting**: Manage inventory audits
- **QC Operations**: Handle quality control processes
- **Discrepancy Management**: Track inventory discrepancies
- **Expiry Management**: Monitor product expiry dates

---

## API Endpoints

### Core Endpoints

#### Warehouse Layout
```
GET    /api/v1/zones                    # Get zones
POST   /api/v1/zones                    # Create zone
GET    /api/v1/floors                   # Get floors
POST   /api/v1/floors                   # Create floor
GET    /api/v1/aisles                   # Get aisles
POST   /api/v1/aisles                   # Create aisle
GET    /api/v1/racks                    # Get racks
POST   /api/v1/racks                    # Create rack
GET    /api/v1/shelves                  # Get shelves
POST   /api/v1/shelves                  # Create shelf
GET    /api/v1/bins                     # Get bins
POST   /api/v1/bins                     # Create bin
```

#### Inventory Management
```
GET    /api/v1/location-inventory       # Get location inventory
POST   /api/v1/location-inventory       # Update location inventory
GET    /api/v1/hub-inventory            # Get hub inventory
POST   /api/v1/hub-inventory            # Update hub inventory
GET    /api/v1/location-sku             # Get SKU-location mapping
POST   /api/v1/location-sku             # Create SKU-location mapping
GET    /api/v1/batches                  # Get batches
POST   /api/v1/batches                  # Create batch
```

#### Order Fulfillment
```
GET    /api/v1/picklists                # Get picklists
POST   /api/v1/picklists                # Create picklist
GET    /api/v1/waves                    # Get waves
POST   /api/v1/waves                    # Create wave
GET    /api/v1/warehouse-carts          # Get warehouse carts
POST   /api/v1/warehouse-carts          # Create warehouse cart
```

#### Goods Receipt
```
GET    /api/v1/grn                      # Get GRNs
POST   /api/v1/grn                      # Create GRN
GET    /api/v1/purchase-orders          # Get purchase orders
POST   /api/v1/purchase-orders          # Create purchase order
GET    /api/v1/suppliers                # Get suppliers
POST   /api/v1/suppliers                # Create supplier
```

#### Quality Control
```
GET    /api/v1/cycle-counts             # Get cycle counts
POST   /api/v1/cycle-counts             # Create cycle count
GET    /api/v1/reports                  # Get reports
POST   /api/v1/reports                  # Generate report
```

---

## Data Models & Collections

### Core Database Tables

#### Warehouse Layout Tables
```sql
-- Zones table
CREATE TABLE zones (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    tenant_id INTEGER NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Floors table
CREATE TABLE floors (
    id SERIAL PRIMARY KEY,
    zone_id INTEGER REFERENCES zones(id),
    name VARCHAR(255) NOT NULL,
    level INTEGER NOT NULL,
    tenant_id INTEGER NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Aisles table
CREATE TABLE aisles (
    id SERIAL PRIMARY KEY,
    floor_id INTEGER REFERENCES floors(id),
    name VARCHAR(255) NOT NULL,
    tenant_id INTEGER NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Racks table
CREATE TABLE racks (
    id SERIAL PRIMARY KEY,
    aisle_id INTEGER REFERENCES aisles(id),
    name VARCHAR(255) NOT NULL,
    capacity INTEGER,
    tenant_id INTEGER NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Shelves table
CREATE TABLE shelves (
    id SERIAL PRIMARY KEY,
    rack_id INTEGER REFERENCES racks(id),
    name VARCHAR(255) NOT NULL,
    capacity INTEGER,
    tenant_id INTEGER NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Bins table
CREATE TABLE bins (
    id SERIAL PRIMARY KEY,
    shelf_id INTEGER REFERENCES shelves(id),
    name VARCHAR(255) NOT NULL,
    capacity INTEGER,
    tenant_id INTEGER NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### Inventory Tables
```sql
-- Location inventory table
CREATE TABLE location_inventory (
    id SERIAL PRIMARY KEY,
    location_id INTEGER NOT NULL,
    sku_id INTEGER NOT NULL,
    quantity INTEGER NOT NULL,
    batch_id INTEGER,
    tenant_id INTEGER NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Hub inventory table
CREATE TABLE hub_inventory (
    id SERIAL PRIMARY KEY,
    hub_id INTEGER NOT NULL,
    sku_id INTEGER NOT NULL,
    quantity INTEGER NOT NULL,
    safety_stock INTEGER DEFAULT 0,
    tenant_id INTEGER NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Location SKU mapping table
CREATE TABLE location_sku (
    id SERIAL PRIMARY KEY,
    location_id INTEGER NOT NULL,
    sku_id INTEGER NOT NULL,
    tenant_id INTEGER NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### Order Fulfillment Tables
```sql
-- Picklists table
CREATE TABLE picklists (
    id SERIAL PRIMARY KEY,
    order_id INTEGER NOT NULL,
    status VARCHAR(50) DEFAULT 'pending',
    tenant_id INTEGER NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Waves table
CREATE TABLE waves (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    status VARCHAR(50) DEFAULT 'pending',
    tenant_id INTEGER NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Warehouse carts table
CREATE TABLE warehouse_carts (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    status VARCHAR(50) DEFAULT 'available',
    tenant_id INTEGER NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

## Worker System

### Background Job Processing
The service runs multiple worker processes for asynchronous operations:

#### Core Workers
1. **Order Management Workers**
   - `NewCreateOrderListener`: Process order creation requests
   - `NewPackOrderListener`: Handle order packing operations
   - `NewCreateDeliveredOrderListener`: Process delivered orders

2. **Inventory Workers**
   - `NewWarehouseSkuListener`: Process SKU updates
   - `NewUpdateHubInventoryListener`: Handle hub inventory updates
   - `NewCreateLocationSkusMappingListener`: Process location-SKU mappings
   - `NewInventoryLogsListener`: Handle inventory log updates

3. **Warehouse Operations Workers**
   - `NewPicklistListener`: Process picklist operations
   - `NewLayoutListener`: Handle layout updates
   - `NewLocationsListener`: Process location operations
   - `NewBinListener`: Handle bin operations

4. **Quality Control Workers**
   - `NewCycleCountUpdateInventoryListener`: Process cycle count updates
   - `NewCreateCycleCountListener`: Handle cycle count creation
   - `NewUpdateNearExpiryThresholdListener`: Process expiry threshold updates

5. **Goods Receipt Workers**
   - `NewCreatePurchaseOrderListener`: Process purchase order creation
   - `NewCreateGrnSerialisationListener`: Handle GRN serialization
   - `NewAutoGRNCompletionListener`: Auto-complete GRN operations

6. **Reporting Workers**
   - `NewReportListener`: Process report generation requests
   - `NewExportCycleCountListener`: Handle cycle count exports
   - `NewExportPurchaseOrderGS1BarcodesListener`: Process GS1 barcode exports

7. **Batch Management Workers**
   - `NewBatchListener`: Process batch operations
   - `NewUpdateSafetyStockListener`: Handle safety stock updates
   - `NewChangeSkuBatchStatusListener`: Process batch status changes

8. **Specialized Workers**
   - `NewAssemblyListener`: Handle assembly operations
   - `NewGenerateWaveListener`: Process wave generation
   - `NewStoListener`: Handle stock transfer orders
   - `NewStockOwnershipTransferListener`: Process ownership transfers

---

## Integration Patterns

### 1. Service-to-Service Communication
- **HTTP API Calls**: Direct service-to-service communication
- **Event-Driven Communication**: Kafka-based event publishing and consumption

### 2. External Service Integration
- **Tenant Service**: User authentication and tenant management
- **OMS Service**: Order management and fulfillment coordination
- **Product Service**: SKU information and product details
- **Shipping Service**: Shipment creation and management
- **Billing Service**: Billing and invoicing operations
- **Rule Service**: Business rule processing
- **Hub Service**: Hub and warehouse management

### 3. Event Streaming Integration
- **Kafka Topics**: Real-time event processing
- **Order Events**: Order lifecycle updates
- **Inventory Events**: Stock level changes
- **Picklist Events**: Picking operation updates
- **Hub Events**: Warehouse layout changes

### 4. Message Queue Integration
- **AWS SQS**: Asynchronous job processing
- **Worker Queues**: Background task execution
- **FIFO Queues**: Ordered processing for critical operations

---

## Configuration

### Environment Configuration
```yaml
server:
  port: ":8083"

postgresql:
  database: "crud"
  master:
    host: "0.0.0.0"
    port: "5433"
    username: "admin"
    password: "admin"

redis:
  hosts: "stagredis.omnifulinfra.com:6379"
  db: 0

# External service configurations
tenantService:
  baseUrl: "http://tenantapi.omnifulinfra.com"
  timeout: 5

omsService:
  baseUrl: "http://omsapi.omnifulinfra.com"
  timeout: 5

shippingService:
  baseUrl: "http://shippingapi.omnifulinfra.com"
  timeout: 5

# Worker configurations
worker:
  purchaseOrder:
    name: "staging-wms-service-failed-purchase-orders"
    workerCount: 1
    concurrencyPerWorker: 4

  hubInventory:
    name: "staging-wms-service-hub-inventory-upload"
    workerCount: 1
    concurrencyPerWorker: 4

# Kafka consumer configurations
consumers:
  order:
    topic: "omniful.wms-aggregator-service.order-events"
    groupId: "omniful.wms-aggregator-service.order-events.wms-service.order.cg"
    enabled: true

  picklist:
    topic: "omniful.wms-service.picklist-update-events"
    groupId: "omniful.wms-service.picklist-update-events.wms-service.picklist.cg"
    enabled: true
```

---

## Development & Deployment

### Local Development Setup
```bash
# Start dependencies
docker-compose up -d

# Run the service
go run main.go

# Run in worker mode
go run main.go -mode=worker

# Run migrations
go run main.go -mode=migration -migrationType=up
```

### Database Migrations
```bash
# Run migrations up
go run main.go -mode=migration -migrationType=up

# Run migrations down
go run main.go -mode=migration -migrationType=down

# Force migration to specific version
go run main.go -mode=migration -migrationType=force -migrationNumber=5
```

### Docker Deployment
```bash
# Build image
docker build -t wms-service .

# Run container
docker run -p 8083:8083 wms-service
```

---

## Common Use Cases

### 1. Warehouse Setup Workflow
```
1. Create zones → 2. Add floors → 3. Create aisles → 4. Setup racks → 5. Configure shelves → 6. Add bins
```

### 2. Inventory Receipt Workflow
```
1. Receive purchase order → 2. Create GRN → 3. Process inwarding → 4. Update inventory → 5. Assign locations
```

### 3. Order Fulfillment Workflow
```
1. Receive order → 2. Generate picklist → 3. Create wave → 4. Pick items → 5. Pack order → 6. Create shipment
```

### 4. Cycle Count Workflow
```
1. Schedule cycle count → 2. Generate count sheets → 3. Perform count → 4. Update inventory → 5. Generate report
```

### 5. Stock Transfer Workflow
```
1. Create transfer order → 2. Pick from source → 3. Transfer items → 4. Update destination → 5. Complete transfer
```

---

## Troubleshooting

### Common Issues

#### 1. Worker Processing Failures
- **Check SQS queue**: Verify messages are being received
- **Review worker logs**: Check for error messages
- **Verify dependencies**: Ensure external services are accessible
- **Check database connections**: Verify PostgreSQL connectivity

#### 2. Inventory Synchronization Issues
- **Verify Kafka topics**: Check event publishing/consumption
- **Review inventory logs**: Check for data inconsistencies
- **Validate business rules**: Ensure proper inventory logic
- **Check cache invalidation**: Verify Redis cache updates

#### 3. API Endpoint Failures
- **Check authentication**: Verify JWT token validity
- **Review request logs**: Check for validation errors
- **Verify database**: Check PostgreSQL connectivity
- **Check middleware**: Ensure proper request processing

#### 4. Performance Issues
- **Database optimization**: Review query performance and indexing
- **Worker scaling**: Adjust worker concurrency settings
- **Cache utilization**: Optimize Redis usage
- **Resource monitoring**: Check CPU, memory, and I/O usage

### Debugging Tools
- **New Relic**: Application performance monitoring
- **Structured logging**: JSON-formatted log output
- **Health checks**: Service health monitoring endpoints
- **Metrics**: Performance and operational metrics

---

**WMS Service** is the backbone of Omniful's warehouse operations, providing comprehensive warehouse management capabilities from layout design to order fulfillment, ensuring efficient and accurate warehouse operations across all tenants.
