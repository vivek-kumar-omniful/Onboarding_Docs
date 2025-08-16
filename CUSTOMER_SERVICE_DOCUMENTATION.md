# Omniful Customer Service Documentation

## Table of Contents
1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Core Functionality](#core-functionality)
4. [Customer Management Flows](#customer-management-flows)
5. [Integration Patterns](#integration-patterns)
6. [API Endpoints](#api-endpoints)
7. [Background Workers](#background-workers)
8. [Configuration](#configuration)

---

## Overview

The **Customer Service** is a critical microservice in the Omniful platform responsible for managing customer data, profiles, addresses, documents, and authentication. It serves as the central hub for all customer-related operations across the eCommerce ecosystem.

### Key Responsibilities
- **Customer Profile Management**: Create, update, and manage customer profiles
- **Address Management**: Handle customer shipping and billing addresses
- **Document Management**: Store and validate customer documents (ID, tax certificates, etc.)
- **Authentication**: Customer login, OTP verification, and session management
- **Bulk Operations**: Process large-scale customer data imports
- **Data Validation**: Ensure data integrity and uniqueness
- **Event Publishing**: Notify other services of customer changes via Kafka

### Business Value
- **Unified Customer Data**: Single source of truth for customer information
- **Multi-tenant Support**: Isolated customer data per seller/tenant
- **Compliance**: Document validation and storage for regulatory requirements
- **Scalability**: Handle millions of customer records efficiently
- **Integration**: Seamless integration with order management and fulfillment

---

## Architecture

### High-Level Architecture
```
┌─────────────────────────────────────────────────────────────┐
│                External Clients                             │
│   (Web Apps, Mobile Apps, Third-party Integrations)         │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│                    Customer Service                         │
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
│                Internal Services                            │
│   ┌──────────────┐ ┌──────────────┐ ┌──────────────┐        │
│   │   OMS        │ │   Sales      │ │   Product    │        │
│   │  Service     │ │  Channel     │ │  Service     │        │
│   │              │ │  Service     │ │              │        │
│   └──────────────┘ └──────────────┘ └──────────────┘        │
│   ┌──────────────┐ ┌──────────────┐ ┌──────────────┐        │
│   │   Tenant     │ │   Analytics  │ │   WMS        │        │
│   │  Service     │ │  Service     │ │  Service     │        │
│   └──────────────┘ └──────────────┘ └──────────────┘        │
└─────────────────────────────────────────────────────────────┘
```

### Directory Structure
```
customer-service/
├── main.go                           # Service entry point with mode selection
├── router/
│   ├── router.go                    # Main router initialization
│   ├── public_routes.go             # Public API endpoints
│   └── internal_routes.go           # Internal service endpoints
├── core/
│   ├── customer/                    # Customer management
│   │   ├── controller/              # HTTP request handlers
│   │   ├── service/                 # Business logic
│   │   ├── repository/              # Data access layer
│   │   └── adapters/                # Data transformation
│   ├── address/                     # Address management
│   ├── document/                    # Document management
│   ├── auth/                        # Authentication
│   ├── workers/                     # Background job processors
│   │   ├── listeners/               # SQS message listeners
│   │   └── handlers/                # Message handlers
│   └── domain/                      # Domain models and interfaces
├── external_service/                # External service clients
├── configs/                         # Configuration management
└── pkg/                            # Shared packages
```

---

## Core Functionality

### 1. Customer Profile Management

#### Customer Creation Flow
```
1. Request Validation
   Validate customer data, check uniqueness
   ↓
2. Document Validation
   Validate customer documents (ID, tax certificates)
   ↓
3. Customer Creation
   Create customer record in MongoDB
   ↓
4. Address Creation (if provided)
   Create associated address record
   ↓
5. Document Storage (if provided)
   Store customer documents in S3
   ↓
6. Event Publishing
   Publish customer creation event to Kafka
   ↓
7. Response
   Return customer data with addresses and documents
```

#### Customer Update Flow
```
1. Customer Validation
   Verify customer exists and can be updated
   ↓
2. Document Validation
   Validate updated documents
   ↓
3. Customer Update
   Update customer record in MongoDB
   ↓
4. Document Update (if provided)
   Update stored documents
   ↓
5. Kafka Event
   Publish customer update event
   ↓
6. Response
   Return updated customer data
```

### 2. Address Management

The service manages multiple types of addresses:
- **Shipping Addresses**: Delivery locations
- **Billing Addresses**: Payment locations
- **Default Addresses**: Primary addresses for each type

### 3. Document Management

Supports various document types:
- **Identity Documents**: Passport, National ID, Driver's License
- **Business Documents**: Tax certificates, business licenses
- **Address Proof**: Utility bills, bank statements

### 4. Authentication

Provides customer authentication mechanisms:
- **OTP-based Login**: Send and verify OTP codes
- **Access Token Management**: JWT token generation and validation
- **Session Management**: Track active sessions

---

## Customer Management Flows

### Complete Customer Creation Flow

```
1. External Request (API Gateway)
   POST /api/v1/sellers/{seller_id}/customers
   ↓
2. Customer Service Controller
   CreateCustomer() → Validate → Process
   ↓
3. Customer Existence Check
   Check if customer already exists (email, phone, documents)
   ↓
4. Document Validation
   Validate document uniqueness and format
   ↓
5. Customer Creation
   Create customer record in MongoDB
   ↓
6. Address Creation (Optional)
   Create associated address if provided
   ↓
7. Document Storage (Optional)
   Upload documents to S3 if provided
   ↓
8. Kafka Event Publishing
   Publish customer.created event
   ↓
9. Response
   Return complete customer data
```

### Key Service Methods

#### Customer Creation
```go
func (svc *CustomerService) CreateCustomer(ctx context.Context, customer svcRequest.Customer) (customerRes svcRes.Customer, cusErr commonError.CustomError) {
    // Customer existence validation
    cusErr = svc.CustomerExistValidation(ctx, customer)
    if cusErr.Exists() {
        return customerRes, cusErr
    }
    
    // Document validation
    if !customer.Documents.IsEmpty() {
        cusErr = svc.documentSvc.ValidateUniqueDocuments(ctx, customer.ToDocumentUserDetails(), customer.Documents)
        if cusErr.Exists() {
            return customerRes, cusErr
        }
    }
    
    // Create customer
    customerModel, cusErr := svc.customerRepo.CreateCustomer(ctx, customer.ToModelCustomer())
    if cusErr.Exists() {
        return customerRes, cusErr
    }
    
    // Handle documents and address
    // ... implementation details
    
    return customerRes, commonError.CustomError{}
}
```

#### Customer Update
```go
func (svc *CustomerService) UpdateCustomer(ctx context.Context, customer svcRequest.Customer) (customerRes svcRes.Customer, cusErr commonError.CustomError) {
    // Customer existence validation
    cusErr = svc.CustomerExistValidation(ctx, customer)
    if cusErr.Exists() {
        return customerRes, cusErr
    }
    
    // Get existing customer and update
    // ... implementation details
    
    // Publish Kafka event
    attributes := utils.KafkaAttributes{
        Topic: config.GetString(ctx, "onlineKafka.topics.customerUpdate"),
        Data:  customerModel,
        Headers: map[string]string{
            "event": "customer.update",
        },
    }
    cusErr = utils.KafkaPublish(ctx, attributes)
    
    return customerRes, cusErr
}
```

---

## Integration Patterns

### 1. Event-Driven Architecture

The Customer Service publishes events to Kafka for other services:

```go
// Customer creation event
attributes := utils.KafkaAttributes{
    Topic: config.GetString(ctx, "onlineKafka.topics.customerCreate"),
    Data:  customerModel,
    Headers: map[string]string{
        "event": "customer.created",
    },
}
cusErr = utils.KafkaPublish(ctx, attributes)

// Customer update event
attributes := utils.KafkaAttributes{
    Topic: config.GetString(ctx, "onlineKafka.topics.customerUpdate"),
    Data:  customerModel,
    Headers: map[string]string{
        "event": "customer.updated",
    },
}
cusErr = utils.KafkaPublish(ctx, attributes)
```

### 2. SQS Integration for Bulk Operations

For large-scale customer imports:

```go
func (e *CustomerHandler) ProcessBulkCustomer(ctx context.Context, sqsMessage sqs.Message) commonError.CustomError {
    var message models.BulkCustomerSQSMessage
    if err := json.Unmarshal(ctx, sqsMessage.Value, &message); err != nil {
        return commonError.CustomError{}
    }
    
    // Process CSV file from S3
    commonCSV, err := csv.NewCommonCSV(
        csv.WithBatchSize(configs.GetBulkCreateCustomerBatchSize(ctx)),
        csv.WithSource(csv.S3),
        csv.WithFileInfo(message.GetFilePath(), message.GetBucketName()),
    )
    
    // Process customers in batches
    for batch := range commonCSV.Process() {
        cusErr := e.processCustomerBatch(ctx, batch)
        if cusErr.Exists() {
            return cusErr
        }
    }
    
    return commonError.CustomError{}
}
```

### 3. MongoDB Integration

Uses MongoDB for customer data storage:

```go
type CustomerRepository interface {
    CreateCustomer(ctx context.Context, customer models.Customer) (models.Customer, commonError.CustomError)
    UpdateCustomer(ctx context.Context, filter map[string]interface{}, customer models.Customer) (models.Customer, commonError.CustomError)
    GetCustomerByFilter(ctx context.Context, attrs models.GetCustomerAttrs) ([]models.Customer, int64, commonError.CustomError)
    DeleteCustomer(ctx context.Context, filter map[string]interface{}) commonError.CustomError
}
```

---

## API Endpoints

### Public API Endpoints

#### Customer Management
```http
# Create customer
POST /api/v1/sellers/{seller_id}/customers

# Create bulk customers
POST /api/v1/sellers/{seller_id}/customers/bulk

# Update customer
PUT /api/v1/sellers/{seller_id}/customers/{customer_id}

# Get customer
GET /api/v1/sellers/{seller_id}/customers/{customer_id}

# Get customers by filter
GET /api/v1/sellers/{seller_id}/customers

# Delete customer
DELETE /api/v1/sellers/{seller_id}/customers/{customer_id}

# Get customer by ID (global)
GET /api/v1/customers/{customer_id}
```

#### Address Management
```http
# Create address
POST /api/v1/addresses

# Update address
PUT /api/v1/addresses/{address_id}

# Get address
GET /api/v1/addresses/{address_id}

# Get addresses by filter
GET /api/v1/addresses
```

#### Authentication
```http
# Login via OTP
POST /api/v1/auth/login

# Logout
POST /api/v1/auth/logout

# Get access token
POST /api/v1/auth/access_token

# Send OTP
POST /api/v1/auth/otp
```

### Internal API Endpoints

#### Internal Customer Management
```http
# Create internal customer
POST /internal/api/v1/sellers/{seller_id}/customers

# Upsert internal customer
POST /internal/api/v1/sellers/{seller_id}/customers/upsert

# Update internal customer
PUT /internal/api/v1/sellers/{seller_id}/customers/{customer_id}

# Get internal customer
GET /internal/api/v1/sellers/{seller_id}/customers/{customer_id}

# Get internal customers by filter
GET /internal/api/v1/sellers/{seller_id}/customers

# Delete internal customer
DELETE /internal/api/v1/sellers/{seller_id}/customers/{customer_id}
```

---

## Background Workers

### Worker Types

The service includes background workers for:

#### 1. Bulk Customer Processing
- **CustomerListener**: Listens to SQS messages for bulk operations
- **CustomerHandler**: Processes bulk customer creation from CSV files

### Worker Configuration

```go
func InitWorkers(ctx context.Context) {
    listenerServers := []listeners.ListenerServer{
        customerListener.NewCustomerListener(ctx),
    }
    
    server := NewServer(listenerServers)
    server.Run(ctx)
    
    shutdown.RegisterShutdownCallback("customer service workers shutdown callback", server)
}
```

### SQS Message Processing

```go
func (l *CustomerListener) Start(ctx context.Context) {
    sqsQueueConfig := sqsConfig.GetCustomerQueueConfig(ctx)
    
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

## Configuration

### Environment Configuration

```yaml
# config.yaml
server:
  port: ":8084"
  readTimeout: 35
  writeTimeout: 30
  idleTimeout: 70

mongodb:
  uri: "mongodb://localhost:27017"
  database: "customer_service"
  maxPoolSize: 100
  minPoolSize: 10

aws:
  s3:
    region: "us-east-1"
    bucket: "customer-documents"
    accessKey: "${AWS_ACCESS_KEY}"
    secretKey: "${AWS_SECRET_KEY}"
  
  sqs:
    region: "us-east-1"
    accessKey: "${AWS_ACCESS_KEY}"
    secretKey: "${AWS_SECRET_KEY}"
    queues:
      customerQueue: "customer-service-queue"

kafka:
  brokers:
    - "localhost:9092"
  topics:
    customerCreate: "customer.created"
    customerUpdate: "customer.updated"
    customerDelete: "customer.deleted"

pusher:
  appId: "${PUSHER_APP_ID}"
  key: "${PUSHER_KEY}"
  secret: "${PUSHER_SECRET}"
  cluster: "us2"

bulk:
  customer:
    batchSize: 100
    maxWorkers: 5

validation:
  email: true
  phone: true
  documents: true

monitoring:
  newRelic:
    enabled: true
    appName: "customer-service"
    licenseKey: "${NEW_RELIC_LICENSE_KEY}"
  
  prometheus:
    enabled: true
    port: ":9090"
```

### Feature Flags

```go
func IsDocumentValidationEnabled(ctx context.Context) bool {
    return config.GetBool(ctx, "validation.documents")
}

func IsEmailValidationEnabled(ctx context.Context) bool {
    return config.GetBool(ctx, "validation.email")
}

func GetBulkCustomerBatchSize(ctx context.Context) int {
    return config.GetInt(ctx, "bulk.customer.batchSize")
}

func GetMaxBulkWorkers(ctx context.Context) int {
    return config.GetInt(ctx, "bulk.customer.maxWorkers")
}
```

---

## Integration with Onboarding Guide

This Customer Service documentation is referenced in the main onboarding guide at:
- [Core Services Deep Dive - Customer Service](./OMNIFUL_ONBOARDING_GUIDE.md#5--customer-service-customer-service)

### Key Integration Points

1. **Customer Data Management**: Centralized customer profile management for the entire platform
2. **Multi-tenant Support**: Isolated customer data per seller/tenant
3. **Document Management**: Secure storage and validation of customer documents
4. **Address Management**: Comprehensive address handling for shipping and billing
5. **Authentication**: Customer login and session management
6. **Event Publishing**: Real-time customer change notifications via Kafka
7. **Bulk Operations**: Large-scale customer data processing via SQS
8. **Data Validation**: Ensuring data integrity and uniqueness across the platform

### Service Dependencies

The Customer Service integrates with:
- **API Gateway**: Receives customer management requests
- **OMS Service**: Provides customer data for order processing
- **Sales Channel Service**: Customer information for platform integrations
- **Product Service**: Customer preferences and history
- **Tenant Service**: Multi-tenant isolation and configuration
- **WMS Service**: Customer addresses for fulfillment

### Related Documentation
- [API Gateway Documentation](./API_GATEWAY_DOCUMENTATION.md)
- [Sales Channel Service Documentation](./SALES_CHANNEL_SERVICE_DOCUMENTATION.md)
- [Technical Reference Guide](./TECHNICAL_REFERENCE.md)

---

