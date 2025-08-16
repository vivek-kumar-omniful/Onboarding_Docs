# Omniful OMS Service Documentation

## Table of Contents
1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Core Functionality](#core-functionality)
4. [Order Processing Flow](#order-processing-flow)
5. [Sales Channel Specific Logic](#sales-channel-specific-logic)
6. [Background Workers](#background-workers)
7. [API Endpoints](#api-endpoints)
8. [Configuration](#configuration)

---

## Overview

The **OMS (Order Management System) Service** is the backbone of the Omniful platform, serving as the central hub for all order-related operations. It manages the complete order lifecycle from creation to fulfillment, handles inventory allocation, coordinates with warehouses, and ensures seamless integration across all sales channels.

### Key Responsibilities
- **Order Lifecycle Management**: Complete order processing from creation to delivery
- **Inventory Allocation**: Reserve and manage stock across multiple warehouses
- **Hub/Warehouse Coordination**: Route orders to appropriate fulfillment centers
- **Sales Channel Integration**: Handle platform-specific order logic and requirements
- **Fulfillment Management**: Coordinate pick, pack, and ship operations
- **Order Status Tracking**: Monitor and update order progress in real-time
- **Returns Processing**: Handle customer returns and refunds
- **Invoice Generation**: Create and manage order invoices
- **Rule Engine**: Apply business rules and automation workflows

### Business Value
- **Centralized Order Management**: Single source of truth for all order operations
- **Multi-channel Support**: Unified processing for 30+ sales channels
- **Scalable Architecture**: Handle millions of orders efficiently
- **Real-time Processing**: Immediate order updates and status changes
- **Automated Workflows**: Reduce manual intervention and errors
- **Comprehensive Analytics**: Detailed order insights and reporting

---

## Architecture

### High-Level Architecture
```
┌─────────────────────────────────────────────────────────────┐
│                External Sources                             │
│   ┌──────────────┐ ┌──────────────┐ ┌──────────────┐        │
│   │ Sales Channel│ │   POS        │ │   Manual     │        │
│   │   Service    │ │  Orders      │ │   Orders     │        │
│   └──────────────┘ └──────────────┘ └──────────────┘        │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│                    OMS Service                              │
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────┐ │
│  │   HTTP Server   │  │  Background     │  │   MongoDB    │ │
│  │   (API Layer)   │  │   Workers       │  │   Database   │ │
│  └─────────────────┘  └─────────────────┘  └──────────────┘ │
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────┐ │
│  │   Controllers   │  │   Services      │  │   SQS/Kafka  │ │
│  │                 │  │                 │  │   Events     │ │
│  └─────────────────┘  └─────────────────┘  └──────────────┘ │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│                External Services                            │
│   ┌──────────────┐ ┌──────────────┐ ┌──────────────┐        │
│   │   WMS        │ │   Customer   │ │   Product    │        │
│   │  Service     │ │  Service     │ │  Service     │        │
│   └──────────────┘ └──────────────┘ └──────────────┘        │
│   ┌──────────────┐ ┌──────────────┐ ┌──────────────┐        │
│   │   Sales      │ │   Invoice    │ │   Shipping   │        │
│   │  Channel     │ │  Service     │ │  Service     │        │
│   └──────────────┘ └──────────────┘ └──────────────┘        │
└─────────────────────────────────────────────────────────────┘
```

### Directory Structure
```
oms-service/
├── main.go                           # Service entry point with mode selection
├── router/
│   ├── router.go                    # Main router initialization
│   ├── public_routes.go             # Public API endpoints
│   ├── internal_routes.go           # Internal service endpoints
│   └── aggregator_routes.go         # Aggregator endpoints
├── core/
│   ├── order/                       # Order management
│   │   ├── controller/              # HTTP request handlers
│   │   ├── service/                 # Business logic
│   │   ├── repository/              # Data access layer
│   │   └── commons/                 # Shared order utilities
│   ├── inventory/                   # Inventory management
│   ├── shipment/                    # Shipment management
│   ├── fulfillment/                 # Fulfillment operations
│   ├── hub/                         # Warehouse/hub management
│   ├── hub_routing/                 # Order routing logic
│   ├── return_order/                # Return processing
│   ├── invoice/                     # Invoice management
│   ├── workers/                     # Background job processors
│   │   ├── listeners/               # SQS message listeners
│   │   └── handlers/                # Message handlers
│   └── domain/                      # Domain models and interfaces
├── external_service/                # External service clients
└── configs/                         # Configuration management
```

---

## Core Functionality

### 1. Order Management

#### Order Creation Flow
```
1. Order Reception
   Receive order from Sales Channel Service or manual entry
   ↓
2. Validation
   Validate order data, customer, items, and inventory
   ↓
3. Hub Assignment
   Route order to appropriate warehouse/hub
   ↓
4. Inventory Allocation
   Reserve stock for order items
   ↓
5. Order Creation
   Create order record in database
   ↓
6. Status Update
   Set initial order status (Pending, On Hold, etc.)
   ↓
7. Event Publishing
   Publish order creation events to Kafka
   ↓
8. Notification
   Send notifications to relevant parties
```

#### Order Processing States
```
Pending → On Hold → Approved → Picking → Picked → Packing → Packed → Shipped → Delivered
    ↓         ↓         ↓         ↓        ↓         ↓        ↓         ↓         ↓
  Manual   Manual   Auto/Manual  WMS     WMS      WMS      WMS      WMS      WMS
  Review   Review    Approval   Task    Task     Task     Task     Task     Task
```

### 2. Inventory Management

#### Inventory Allocation
- **Stock Reservation**: Reserve inventory for orders
- **Hub Inventory**: Manage stock across multiple warehouses
- **External Hub Integration**: Coordinate with external fulfillment centers
- **Stock Transfer**: Handle inter-warehouse transfers

#### Inventory Operations
```go
// Create order in WMS
func (svc *InventoryService) CreateOrder(ctx context.Context, order *models.Order) (invRes.CreateOrderRes, commonError.CustomError) {
    // Check hub type (internal vs external)
    hub, interSvcErr := svc.wmsClient.GetHubWithAddress(ctx, order.TenantID, order.HubID)
    
    switch hub.MetaData.IsExternalHub {
    case true:
        return svc.externalHubCreateOrder(ctx, order)
    case false:
        return svc.internalHubCreateOrder(ctx, order)
    }
}
```

### 3. Hub Management

#### Hub Routing
- **Geographic Routing**: Route orders based on delivery location
- **Capacity Management**: Consider warehouse capacity and workload
- **Performance Optimization**: Route to best-performing hubs
- **Fallback Logic**: Handle hub unavailability

---

## Order Processing Flow

### Complete Order Processing Flow

```
1. Order Reception (Sales Channel Service)
   SQS Message → CreateOrderListener → CreateOrderHandler
   ↓
2. Order Validation
   Validate customer, items, inventory availability
   ↓
3. Hub Assignment
   Route order to appropriate warehouse
   ↓
4. Inventory Allocation
   Reserve stock in WMS
   ↓
5. Order Creation
   Create order record in MongoDB
   ↓
6. Status Management
   Set initial status and trigger workflows
   ↓
7. Approval Process
   Manual or automatic order approval
   ↓
8. Fulfillment
   Pick → Pack → Ship operations
   ↓
9. Delivery Tracking
   Monitor delivery status
   ↓
10. Completion
    Mark order as delivered
```

### Key Service Methods

#### Order Creation Handler
```go
// From core/workers/handlers/order/order_handler/sales_channel_order.go
func (or *CreateOrderHandler) Process(ctx context.Context, sqsMessages *[]commonSQS.Message) error {
    var request handlerRequest.SalesChannelOrderReq
    if err := json.Unmarshal(ctx, messages[0].Value, &request); err != nil {
        return nil
    }
    
    // Validate request
    err := validator.Get().Struct(request)
    if err != nil {
        return err
    }
    
    // Check for missing sales channel item IDs
    skuCodesWithNoSSCItemID := make([]string, 0)
    for _, orderItem := range request.Order.OrderItems {
        if len(orderItem.SalesChannelOrderItemID) == 0 {
            skuCodesWithNoSSCItemID = append(skuCodesWithNoSSCItemID, orderItem.SellerSkuCode)
        }
    }
    
    // Process order based on event type
    switch request.Event {
    case constants.CreateOrder:
        orderRequest := request.Order.ToSvcCreateOrderRequest()
        _, cusErr := or.salesChannelOrderSvc.CreateOrder(ctx, &orderRequest, &models.UserDetail{})
        if cusErr.Exists() {
            return cusErr.ToError()
        }
    }
    
    return nil
}
```

#### Order Edit Service
```go
// From core/order/service/order_create.go
func (svc *OrderService) EditOrder(ctx context.Context, attrs svcRequest.EditOrderSvcAttrs) (orderRes svcResponse.OrderRes, cusErr commonError.CustomError) {
    // Acquire order lock
    lockKey := getOrderEditLockKey(attrs.OMSOrderID)
    acquired, lockerErr := svc.locker.Lock(ctx, lockKey, constants.OrderDetailsEditLockDuration)
    if !acquired {
        return orderRes, omsError.InvalidRequest(ctx, constants.OrderInProgress)
    }
    
    // Get existing order
    orderModel, cusErr := svc.orderRepo.GetOrder(ctx, attrs.UserDetails.TenantID, attrs.OMSOrderID)
    if cusErr.Exists() {
        return orderRes, cusErr
    }
    
    // Update order based on edit type
    switch attrs.EditEntityType {
    case enums.CustomerDetailsEntityType:
        if !orderModel.CanEditOrderDetails() {
            return orderRes, omsError.InvalidRequest(ctx, constants.UpdateNotAllowed)
        }
        attrs.Customer.UpdateCustomerDetailsInOrder(&updatedOrderModel)
        
    case enums.ShippingDetailsEntityType:
        if attrs.ShippingAddress != nil {
            attrs.ShippingAddress.UpdateShippingDetailsInOrder(&updatedOrderModel)
            
            // Sales channel specific logic
            if attrs.Source.Is(enums.SalesChannelSource) {
                updatedOrderModel.ShippingAddress.SalesChannelCity = attrs.ShippingAddress.City
                updatedOrderModel.ShippingAddress.SalesChannelState = attrs.ShippingAddress.State
                updatedOrderModel.ShippingAddress.SalesChannelCountry = attrs.ShippingAddress.Country
            }
        }
    }
    
    return orderRes, cusErr
}
```

---

## Sales Channel Specific Logic

### Platform-Specific Processing

The OMS Service contains significant sales channel-specific logic to handle the unique requirements of different platforms:

#### 1. Sales Channel Order Processing
```go
// From core/workers/handlers/order/order_handler/sales_channel_order.go
func (or *CreateOrderHandler) Process(ctx context.Context, sqsMessages *[]commonSQS.Message) error {
    // Sales channel specific validation
    skuCodesWithNoSSCItemID := make([]string, 0)
    for _, orderItem := range request.Order.OrderItems {
        if len(orderItem.SalesChannelOrderItemID) == 0 {
            skuCodesWithNoSSCItemID = append(skuCodesWithNoSSCItemID, orderItem.SellerSkuCode)
        }
    }
    
    // Alert for missing sales channel item IDs
    if len(skuCodesWithNoSSCItemID) > 0 {
        newrelic.NoticeError(ctx,
            fmt.Errorf("RequestID: %s, Received Order %s for sales channel %s without sales channel item id for skus %+v",
                env.GetRequestID(ctx), request.Order.SellerSalesChannelOrderID, request.Order.SalesChannel.Name, skuCodesWithNoSSCItemID))
    }
}
```

#### 2. Platform-Specific Address Handling
```go
// From core/order/service/order_create.go
case enums.ShippingDetailsEntityType:
    if attrs.ShippingAddress != nil {
        attrs.ShippingAddress.UpdateShippingDetailsInOrder(&updatedOrderModel)
        
        // Sales channel specific address fields
        if attrs.Source.Is(enums.SalesChannelSource) {
            updatedOrderModel.ShippingAddress.SalesChannelCity = attrs.ShippingAddress.City
            updatedOrderModel.ShippingAddress.SalesChannelState = attrs.ShippingAddress.State
            updatedOrderModel.ShippingAddress.SalesChannelCountry = attrs.ShippingAddress.Country
        }
    }
```

#### 3. Platform-Specific Status Mapping
```go
// Different platforms have different status requirements
// Shopify: pending, paid, fulfilled, cancelled
// WooCommerce: processing, completed, cancelled
// Amazon: Pending, Shipped, Delivered
```

### Key Sales Channel Considerations

1. **Order ID Mapping**: Each platform has different order ID formats
2. **Item ID Mapping**: Platform-specific product/item identifiers
3. **Status Synchronization**: Real-time status updates back to platforms
4. **Fulfillment Requirements**: Platform-specific fulfillment workflows
5. **Tax Handling**: Different tax calculation and reporting requirements
6. **Shipping Integration**: Platform-specific shipping carrier integration

---

## Background Workers

### Worker Types

The OMS Service includes multiple types of background workers:

#### 1. Order Processing Workers
- **CreateOrderListener**: Processes new order creation
- **OrderUpdatesEventsListener**: Handles order status updates
- **AssignHubListener**: Routes orders to appropriate hubs
- **HubRoutingListener**: Manages hub routing logic
- **DashboardUpdatesListener**: Updates dashboard metrics
- **BulkActionOrderListener**: Processes bulk order operations

#### 2. Shipment Workers
- **ShipmentListener**: Handles shipment creation and updates
- **ShipmentCreationEventsListener**: Processes shipment events
- **BulkShipmentListener**: Processes bulk shipment operations

#### 3. Inventory Workers
- **Inventory updates**: Real-time inventory synchronization
- **Stock transfer operations**: Inter-warehouse transfers

### Worker Configuration

```go
// From core/workers/worker.go
func InitWorkers1(ctx context.Context) {
    listenerServers := []listeners.ListenerServer{
        orderListener.NewCreateOrderListener(ctx),
        orderListener.NewOrderUpdatesEventsListener(ctx),
        orderListener.NewAssignHubListener(ctx),
        order_listener.NewHubRoutingListener(ctx),
        orderListener.NewDashboardUpdatesListener(ctx),
        orderListener.NewDarkStoreTimelineEventListener(ctx),
        orderListener.ApproveOnHoldOrderListener(ctx),
        orderListener.NewApproveOnHoldOrderListenerV2(ctx),
        orderListener.NewShippingUpdateShipmentListener(ctx),
        orderListener.NewBulkActionOrderListener(ctx),
        orderListener.NewUpdateOrderDetailsListener(ctx),
        orderListener.NewOrderInvoiceListener(ctx),
        notificationListener.NewOrderNotificationListener(ctx),
        // ... more listeners
    }
    
    server := NewServer(listenerServers)
    server.Run(ctx)
    shutdown.RegisterShutdownCallback("oms service worker 1 shutdown callback", server)
}
```

### SQS Message Processing

```go
// From core/workers/listeners/order/order_listener/sales_channel_order.go
func (l *CreateOrderListener) Start(ctx context.Context) {
    sqsQueueConfig := sqsConfig.GetCreateOrderQueueConfig(ctx)
    
    queue, err := sqs.NewStandardQueue(ctx,
        sqsQueueConfig.QueueName,
        sqsQueueConfig.ToGoCommonsSQSConfig(),
    )
    
    consumer, err := sqs.NewConsumer(
        queue,
        uint64(sqsQueueConfig.WorkerCount),
        sqsQueueConfig.ConcurrencyPerWorker,
        l.Handler,
        10,
        30,
        true,
        false,
    )
    
    consumer.Start(ctx)
}
```

---

## API Endpoints

### Public API Endpoints

#### Order Management
```http
# Create order
POST /api/v1/sellers/{seller_id}/orders

# Get order
GET /api/v1/sellers/{seller_id}/orders/{order_id}

# Update order
PUT /api/v1/sellers/{seller_id}/orders/{order_id}

# Get orders by filter
GET /api/v1/sellers/{seller_id}/orders

# Bulk order operations
POST /api/v1/sellers/{seller_id}/orders/bulk

# Approve order
POST /api/v1/sellers/{seller_id}/orders/{order_id}/approve

# Cancel order
POST /api/v1/sellers/{seller_id}/orders/{order_id}/cancel
```

#### Shipment Management
```http
# Create shipment
POST /api/v1/sellers/{seller_id}/shipments

# Get shipment
GET /api/v1/sellers/{seller_id}/shipments/{shipment_id}

# Update shipment
PUT /api/v1/sellers/{seller_id}/shipments/{shipment_id}

# Bulk shipment operations
POST /api/v1/sellers/{seller_id}/shipments/bulk
```

#### Inventory Management
```http
# Get inventory
GET /api/v1/sellers/{seller_id}/inventory

# Update inventory
PUT /api/v1/sellers/{seller_id}/inventory

# Stock transfer
POST /api/v1/sellers/{seller_id}/stock-transfers
```

#### Hub Management
```http
# Get hubs
GET /api/v1/sellers/{seller_id}/hubs

# Assign hub to order
POST /api/v1/sellers/{seller_id}/orders/{order_id}/assign-hub

# Reassign hub
POST /api/v1/sellers/{seller_id}/orders/{order_id}/reassign-hub
```

### Internal API Endpoints

#### Internal Order Management
```http
# Create internal order
POST /internal/api/v1/sellers/{seller_id}/orders

# Get internal order
GET /internal/api/v1/sellers/{seller_id}/orders/{order_id}

# Update internal order
PUT /internal/api/v1/sellers/{seller_id}/orders/{order_id}

# Bulk internal operations
POST /internal/api/v1/sellers/{seller_id}/orders/bulk
```

---

## Configuration

### Environment Configuration

```yaml
# config.yaml
server:
  port: ":8082"
  readTimeout: 35
  writeTimeout: 35
  idleTimeout: 70

mongodb:
  uri: "mongodb://localhost:27017"
  database: "oms_service"
  maxPoolSize: 100
  minPoolSize: 10

redis:
  host: "localhost"
  port: 6379
  password: ""
  db: 0

aws:
  sqs:
    region: "us-east-1"
    accessKey: "${AWS_ACCESS_KEY}"
    secretKey: "${AWS_SECRET_KEY}"
    queues:
      createOrderQueue: "oms-create-order-queue"
      orderUpdatesQueue: "oms-order-updates-queue"
      shipmentQueue: "oms-shipment-queue"
      inventoryQueue: "oms-inventory-queue"

kafka:
  brokers:
    - "localhost:9092"
  topics:
    orderCreated: "order.created"
    orderUpdated: "order.updated"
    orderCancelled: "order.cancelled"
    shipmentCreated: "shipment.created"
    inventoryUpdated: "inventory.updated"

wms:
  baseURL: "http://wms-service:8083"
  timeout: 30

salesChannel:
  baseURL: "http://sales-channel-service:8081"
  timeout: 30

pusher:
  appId: "${PUSHER_APP_ID}"
  key: "${PUSHER_KEY}"
  secret: "${PUSHER_SECRET}"
  cluster: "us2"

order:
  defaultStatus: "pending"
  autoApproval: true
  bulkBatchSize: 100
  maxRetryAttempts: 3

inventory:
  reservationTimeout: "15m"
  autoRelease: true
  lowStockThreshold: 10

hub:
  routingEnabled: true
  capacityCheck: true
  performanceWeight: 0.7
  proximityWeight: 0.3

monitoring:
  newRelic:
    enabled: true
    appName: "oms-service"
    licenseKey: "${NEW_RELIC_LICENSE_KEY}"
  
  prometheus:
    enabled: true
    port: ":9090"
```

### Feature Flags

```go
// From configs/environment.go
func IsAutoApprovalEnabled(ctx context.Context) bool {
    return config.GetBool(ctx, "order.autoApproval")
}

func IsHubRoutingEnabled(ctx context.Context) bool {
    return config.GetBool(ctx, "hub.routingEnabled")
}

func GetBulkBatchSize(ctx context.Context) int {
    return config.GetInt(ctx, "order.bulkBatchSize")
}

func GetMaxRetryAttempts(ctx context.Context) int {
    return config.GetInt(ctx, "order.maxRetryAttempts")
}
```

---

## Integration with Onboarding Guide

This OMS Service documentation is referenced in the main onboarding guide at:
- [Core Services Deep Dive - OMS Service](./OMNIFUL_ONBOARDING_GUIDE.md#3--oms-service-oms-service)

### Key Integration Points

1. **Order Lifecycle Management**: Complete order processing from creation to delivery
2. **Sales Channel Integration**: Platform-specific order handling and requirements
3. **Inventory Management**: Stock allocation and warehouse coordination
4. **Hub Management**: Order routing and warehouse assignment
5. **Fulfillment Coordination**: Pick, pack, and ship operations
6. **Event Publishing**: Real-time order updates via Kafka
7. **Background Processing**: Asynchronous order processing via SQS
8. **Multi-tenant Support**: Isolated order data per seller/tenant

### Service Dependencies

The OMS Service integrates with:
- **API Gateway**: Receives order management requests
- **Sales Channel Service**: Receives orders from external platforms
- **WMS Service**: Warehouse management and inventory operations
- **Customer Service**: Customer data for order processing
- **Product Service**: Product information and SKU management
- **Tenant Service**: Multi-tenant isolation and configuration

### Related Documentation
- [API Gateway Documentation](./API_GATEWAY_DOCUMENTATION.md)
- [Sales Channel Service Documentation](./SALES_CHANNEL_SERVICE_DOCUMENTATION.md)
- [Customer Service Documentation](./CUSTOMER_SERVICE_DOCUMENTATION.md)
- [Technical Reference Guide](./TECHNICAL_REFERENCE.md)

---
