# ![[Pasted image 20250816171101.png|23]] Omniful Product Service Documentation

## Table of Contents
1. [Overview](#overview)
2. [Architecture & Design](#architecture--design)
3. [Core Functionality](#core-functionality)
4. [API Endpoints](#api-endpoints)
5. [Data Models & Collections](#data-models--collections)
6. [Worker System](#worker-system)
7. [Search & Indexing](#search--indexing)
8. [Integration Patterns](#integration-patterns)
9. [Configuration](#configuration)
10. [Development & Deployment](#development--deployment)
11. [Common Use Cases](#common-use-cases)
12. [Troubleshooting](#troubleshooting)

---

## Overview

The **Product Service** is a core microservice in the Omniful platform responsible for managing the entire product catalog, SKUs, categories, attributes, and product-related operations. It serves as the central hub for all product-related data and operations across the platform.

### Key Responsibilities
- **Product Catalog Management**: Centralized product information storage and management
- **SKU Management**: Handle product variants, barcodes, and inventory tracking
- **Category & Attribute Management**: Organize products with flexible categorization and attributes
- **Search & Discovery**: Provide fast product search and filtering capabilities
- **Multi-tenant Support**: Isolate data between different sellers/tenants
- **Bulk Operations**: Handle large-scale product imports, exports, and updates
- **Integration Hub**: Connect with other services for order processing and inventory management

### Service Characteristics
- **Database**: MongoDB (primary), Redis (caching)
- **Search Engine**: MongoDB Atlas Search
- **Message Queue**: AWS SQS for asynchronous processing
- **Event Streaming**: Kafka for real-time updates
- **API Style**: RESTful HTTP APIs with JWT authentication
- **Multi-mode Operation**: HTTP server, worker mode, and index creation mode

---

## Architecture & Design

### Service Architecture
```
┌─────────────────────────────────────────────────────────────┐
│                    Product Service                          │
├─────────────────────────────────────────────────────────────┤
│    ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│    │   HTTP API  │  │   Workers   │  │   Indexing  │        │
│    │   Server    │  │   System    │  │   System    │        │
│    └─────────────┘  └─────────────┘  └─────────────┘        │
├─────────────────────────────────────────────────────────────┤
│    ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│    │ Controllers │  │   Services  │  │   Models    │        │
│    │             │  │             │  │             │        │
│    └─────────────┘  └─────────────┘  └─────────────┘        │
├─────────────────────────────────────────────────────────────┤
│    ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│    │   MongoDB   │  │    Redis    │  │    Kafka    │        │
│    │             │  │             │  │             │        │
│    └─────────────┘  └─────────────┘  └─────────────┘        │
└─────────────────────────────────────────────────────────────┘
```

### Directory Structure
```
product-service/
├── main.go                           # Service entry point
├── configs/                          # Configuration files
│   ├── config.yaml                  # Main configuration
│   └── redis.conf                   # Redis configuration
├── router/                           # API routing
│   ├── router.go                    # Main router setup
│   ├── public_routes.go             # Public API endpoints
│   └── internal_routes.go           # Internal service endpoints
├── internal/                         # Core business logic
│   ├── controllers/                  # HTTP request handlers
│   │   ├── public/                  # Public API controllers
│   │   └── internal_controllers/    # Internal API controllers
│   ├── workers/                     # Background job processors
│   ├── domain/                      # Business domain models
│   ├── enums/                       # Enumerations and constants
│   └── utils/                       # Utility functions
├── pkg/                             # Shared packages
│   ├── db/                          # Database utilities
│   ├── redis/                       # Redis client
│   ├── middleware/                  # HTTP middleware
│   ├── kafka_producer/              # Kafka event publishing
│   └── pusher/                      # Real-time notifications
├── index/                           # Search indexing
├── constants/                       # Service constants
├── init/                            # Initialization logic
└── external_service/                # External service clients
```

### Data Flow Architecture
```
External Request → API Gateway → Product Service → Database/External Services
                     ↓
              Authentication & Authorization
                     ↓
              Route Handler → Controller → Service → Repository
                     ↓
              Response ← Cache ← Database ← External APIs
```

---

## Core Functionality

### 1. Product Management
- **Product Creation**: Create new products with basic information
- **Product Updates**: Modify existing product details
- **Product Deletion**: Soft delete products (archive)
- **Product Search**: Full-text search with filters
- **Product Validation**: Ensure data integrity and business rules

### 2. SKU Management
- **SKU Creation**: Generate unique SKUs for products
- **Variant Management**: Handle product variations (size, color, etc.)
- **Barcode Management**: Associate barcodes with SKUs
- **Inventory Tracking**: Monitor stock levels and availability
- **Bulk Operations**: Import/export large numbers of SKUs

### 3. Category Management
- **Hierarchical Categories**: Support nested category structures
- **Category CRUD**: Create, read, update, delete categories
- **Category Mapping**: Map categories across different sales channels
- **Category Validation**: Ensure category hierarchy integrity

### 4. Attribute Management
- **Dynamic Attributes**: Flexible attribute system for products
- **Attribute Groups**: Group related attributes together
- **Attribute Variations**: Handle different attribute types (text, number, select)
- **Attribute Validation**: Ensure attribute value consistency

### 5. Sales Channel Integration
- **Platform Mapping**: Map products to external platform formats
- **Inventory Sync**: Synchronize stock levels across platforms
- **Product Listing**: Manage product listings on different channels
- **Webhook Processing**: Handle real-time updates from platforms

---

## API Endpoints

### Base URL Structure
```
/api/v2/sellers/{seller_id}/[resource]
/api/v3/sellers/{seller_id}/[resource]  # Newer version
```

### Core Endpoints

#### Products
```
GET    /api/v2/sellers/:seller_id/products          # List products
POST   /api/v2/sellers/:seller_id/products          # Create product
GET    /api/v2/sellers/:seller_id/products/:id      # Get product
PUT    /api/v2/sellers/:seller_id/products/:id      # Update product
DELETE /api/v2/sellers/:seller_id/products/:id      # Delete product
```

#### SKUs
```
GET    /api/v2/sellers/:seller_id/skus              # List SKUs
POST   /api/v2/sellers/:seller_id/skus              # Create SKU
GET    /api/v2/sellers/:seller_id/skus/:id          # Get SKU
PUT    /api/v2/sellers/:seller_id/skus/:id          # Update SKU
DELETE /api/v2/sellers/:seller_id/skus/:id          # Delete SKU
```

#### Categories
```
GET    /api/v2/sellers/:seller_id/categories        # List categories
POST   /api/v2/sellers/:seller_id/categories        # Create category
GET    /api/v2/sellers/:seller_id/categories/:id    # Get category
PUT    /api/v2/sellers/:seller_id/categories/:id    # Update category
DELETE /api/v2/sellers/:seller_id/categories/:id    # Delete category
```

#### Attributes
```
GET    /api/v2/sellers/:seller_id/attributes        # List attributes
POST   /api/v2/sellers/:seller_id/attributes        # Create attribute
GET    /api/v2/sellers/:seller_id/attributes/:id    # Get attribute
PUT    /api/v2/sellers/:seller_id/attributes/:id    # Update attribute
DELETE /api/v2/sellers/:seller_id/attributes/:id    # Delete attribute
```

#### Sales Channel Integration
```
GET    /api/v2/sellers/:seller_id/sales_channel_skus           # List channel SKUs
POST   /api/v2/sellers/:seller_id/sales_channel_skus          # Create channel SKU
GET    /api/v2/sellers/:seller_id/sales_channel_categories    # List channel categories
GET    /api/v2/sellers/:seller_id/sales_channel_attributes    # List channel attributes
```

#### Configuration
```
GET    /api/v2/configurations                           # Get configurations
POST   /api/v2/configurations                           # Create configuration
PUT    /api/v2/configurations/:id                       # Update configuration
GET    /api/v2/configurations/:id                       # Get configuration
```

#### Pricing & Tax
```
GET    /api/v2/sellers/:seller_id/pricing              # Export pricing
GET    /api/v2/catalog/tax_categories                  # List tax categories
POST   /api/v2/catalog/tax_categories                  # Create tax category
PUT    /api/v2/catalog/tax_categories/:id              # Update tax category
```

### Authentication & Authorization
- **JWT Authentication**: Required for all endpoints
- **Tenant Isolation**: Data is automatically filtered by seller_id
- **Role-based Access**: Different permissions for different user roles

---

## Data Models & Collections

### Core Collections

#### Products Collection
```json
{
  "_id": "ObjectId",
  "seller_id": "number",
  "name": "string",
  "description": "string",
  "category_id": "ObjectId",
  "manufacturer_id": "ObjectId",
  "brand_id": "ObjectId",
  "status": "string",
  "created_at": "Date",
  "updated_at": "Date"
}
```

#### SKUs Collection
```json
{
  "_id": "ObjectId",
  "seller_id": "number",
  "product_id": "ObjectId",
  "sku_code": "string",
  "barcode": "string",
  "attributes": "array",
  "pricing": "object",
  "inventory": "object",
  "status": "string",
  "created_at": "Date",
  "updated_at": "Date"
}
```

#### Categories Collection
```json
{
  "_id": "ObjectId",
  "seller_id": "number",
  "name": "string",
  "code": "string",
  "parent_id": "ObjectId",
  "level": "number",
  "path": "array",
  "status": "string",
  "created_at": "Date",
  "updated_at": "Date"
}
```

#### Attributes Collection
```json
{
  "_id": "ObjectId",
  "seller_id": "number",
  "name": "string",
  "code": "string",
  "type": "string",
  "group_id": "ObjectId",
  "is_required": "boolean",
  "status": "string",
  "created_at": "Date",
  "updated_at": "Date"
}
```

### Database Indexes
The service creates optimized indexes for:
- **Seller-based queries**: `{seller_id: 1, created_at: 1}`
- **Text search**: `{seller_id: 1, name: "text"}`
- **Unique constraints**: `{seller_id: 1, code: 1}` (unique)
- **Relationship queries**: `{seller_id: 1, category_id: 1}`

---

## Worker System

### Background Job Processing
The service runs multiple worker processes for asynchronous operations:

#### Core Workers
1. **SKU Workers**
   - `CreateSkuAutoOffListener`: Process SKU creation requests
   - `ImportSkuAutoOffListener`: Handle bulk SKU imports
   - `ExportSkuAutoOffListener`: Process SKU export requests
   - `SyncSkuAutoOffListener`: Synchronize SKUs across systems

2. **Catalog Workers**
   - `CatalogComparisonAutoOffListener`: Compare product catalogs
   - `CatalogSKUBulkCreateAutoOffListener`: Bulk create catalog SKUs
   - `NewCatalogComparisonAutoOffListener`: Enhanced catalog comparison

3. **Configuration Workers**
   - `NewEvaluateConfigurationListener`: Evaluate configuration changes
   - `NewWeightedAverageCostListener`: Calculate weighted average costs

4. **Integration Workers**
   - `SalesChannelListingAutoOffListener`: Manage sales channel listings
   - `NewReturnOrderProcessingListener`: Process return orders
   - `NewUpdateSKUListener`: Handle SKU update events

#### Worker Configuration
```yaml
worker:
  sku:
    name: "local-product-service-skus"
    workerCount: 1
    concurrencyPerWorker: 2
  importSku:
    name: "local-product-service-product.fifo"
    workerCount: 1
    concurrencyPerWorker: 2
  exportSku:
    name: "local-product-service-export-skus"
    workerCount: 1
    concurrencyPerWorker: 2
```

### Message Queue Integration
- **AWS SQS**: Primary message queue for worker jobs
- **FIFO Queues**: For operations requiring strict ordering
- **Standard Queues**: For general asynchronous processing
- **Dead Letter Queues**: For failed job handling

---

## Search & Indexing

### MongoDB Atlas Search
The service uses MongoDB Atlas Search for advanced product discovery:

#### Search Features
- **Full-text Search**: Search across product names, descriptions, and attributes
- **Fuzzy Matching**: Handle typos and variations in search terms
- **Faceted Search**: Filter results by category, brand, price range, etc.
- **Geographic Search**: Location-based product discovery
- **Relevance Scoring**: Rank results by relevance

#### Index Configuration
```yaml
mongodb:
  atlas:
    search:
      enabled: true
      skusIndex:
        name: "default"
        minShouldMatch: 1
```

#### Search API
```go
// Example search query
searchQuery := bson.M{
    "$search": bson.M{
        "index": "default",
        "text": bson.M{
            "query": "search term",
            "path": []string{"name", "description"},
            "fuzzy": bson.M{"maxEdits": 2},
        },
    },
}
```

### Index Management
- **Automatic Indexing**: Indexes are created automatically on startup
- **Index Optimization**: Regular index maintenance and optimization
- **Search Performance**: Monitoring and tuning search performance

---

## Integration Patterns

### 1. Service-to-Service Communication

#### HTTP API Calls
```go
// Example: Calling OMS service
omsServiceURL := config.GetString(ctx, "omsService.baseUrl")
response, err := httpClient.Post(omsServiceURL + "/api/v1/orders", data)
```

#### Event-Driven Communication
```go
// Example: Publishing SKU update event
kafkaProducer.Publish("omniful.product-service.sku-update-events", event)
```

### 2. External Platform Integration

#### Sales Channel Mapping
```go
// Map internal product to external platform format
externalProduct := mapProductToPlatform(internalProduct, platformConfig)
response, err := platformAPI.CreateProduct(externalProduct)
```

#### Webhook Processing
```go
// Process incoming webhooks from external platforms
func (c *Controller) HandleWebhook(ctx *gin.Context) {
    webhookData := parseWebhook(ctx)
    event := createEventFromWebhook(webhookData)
    sqsClient.SendMessage(queueURL, event)
}
```

### 3. Data Synchronization

#### Inventory Sync
```go
// Sync inventory changes to external platforms
func (s *Service) SyncInventory(ctx context.Context, skuID string, quantity int) error {
    // Update internal inventory
    err := s.updateInternalInventory(ctx, skuID, quantity)
    if err != nil {
        return err
    }
    
    // Notify external platforms
    event := createInventoryUpdateEvent(skuID, quantity)
    return s.publishInventoryEvent(ctx, event)
}
```

#### Product Sync
```go
// Sync product changes across systems
func (s *Service) SyncProduct(ctx context.Context, productID string) error {
    // Get product data
    product, err := s.getProduct(ctx, productID)
    if err != nil {
        return err
    }
    
    // Update search index
    err = s.updateSearchIndex(ctx, product)
    if err != nil {
        return err
    }
    
    // Notify other services
    event := createProductUpdateEvent(product)
    return s.publishProductEvent(ctx, event)
}
```

---

## Configuration

### Environment Configuration
```yaml
server:
  port: ":8088"
  readTimeout: 35
  writeTimeout: 30
  idleTimeout: 70

service:
  name: "product_service"

mongodb:
  database: "product_service_db"
  maxPoolSize: 20
  master:
    uri: "mongodb://admin:admin@127.0.0.1:27017/product_service_db"
    host: "127.0.0.1"
    port: "27017"

redis:
  clusterMode: false
  hosts: "localhost:7006"
  db: 0

kafka:
  brokers: ["127.0.0.1:9095"]
  clientId: "product-service"
  version: 3.3.2
```

### Feature Flags
```yaml
featureFlag:
  active: true
  importSkuV2:
    enable: true
    tenants: [115]
    allTenants: false
  wac:
    enabled: true
    allTenants: false
    tenants: [1, 147]
```

### Service Dependencies
```yaml
tenantService:
  baseUrl: "http://localhost:8081"
  timeout: 60

wmsService:
  baseUrl: "http://wmsapi.omnifulinfra.com/"
  timeout: 60

salesChannelService:
  baseUrl: "http://localhost:8085"
  timeout: 60

omsService:
  baseUrl: "http://omsapi.omnifulinfra.com/"
  timeout: 60
```

---

## Development & Deployment

### Local Development Setup

#### Prerequisites
- Go 1.22+
- Docker & Docker Compose
- MongoDB
- Redis
- Kafka
- LocalStack (for SQS/S3)

#### Quick Start
```bash
# Clone and navigate to service
cd product-service

# Start dependencies
docker-compose up -d

# Install dependencies
go mod download

# Run the service
go run main.go

# Or run in different modes
go run main.go -mode=worker        # Worker mode
go run main.go -mode=create_index  # Create database indexes
```

#### Environment Variables
```bash
# Copy and configure environment
cp .env.example .env

# Key environment variables
MONGODB_URI=mongodb://admin:admin@localhost:27017/product_service_db
REDIS_HOST=localhost:7006
KAFKA_BROKERS=localhost:9095
AWS_REGION=eu-central-1
```

### Docker Deployment
```dockerfile
# Build image
docker build -t product-service .

# Run container
docker run -d \
  --name product-service \
  -p 8088:8088 \
  -e MONGODB_URI=mongodb://mongodb:27017/product_service_db \
  product-service
```

### Kubernetes Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: product-service
  template:
    metadata:
      labels:
        app: product-service
    spec:
      containers:
      - name: product-service
        image: product-service:latest
        ports:
        - containerPort: 8088
        env:
        - name: MONGODB_URI
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: uri
```

---

## Common Use Cases

### 1. Product Import Workflow
```
1. Upload CSV/Excel file → 2. Validate data → 3. Create products/SKUs → 4. Update search index → 5. Notify other services
```

#### Implementation
```go
func (s *Service) ImportProducts(ctx context.Context, file multipart.File) error {
    // 1. Parse file
    products, err := s.parseImportFile(file)
    if err != nil {
        return err
    }
    
    // 2. Validate data
    err = s.validateProducts(ctx, products)
    if err != nil {
        return err
    }
    
    // 3. Create products in batches
    err = s.createProductsBatch(ctx, products)
    if err != nil {
        return err
    }
    
    // 4. Update search index
    err = s.updateSearchIndex(ctx, products)
    if err != nil {
        return err
    }
    
    // 5. Publish events
    return s.publishProductEvents(ctx, products)
}
```

### 2. Catalog Comparison
```
1. Select source and target catalogs → 2. Compare products → 3. Generate differences → 4. Export results → 5. Sync changes
```

#### Implementation
```go
func (s *Service) CompareCatalogs(ctx context.Context, sourceID, targetID string) (*ComparisonResult, error) {
    // 1. Get source catalog
    sourceCatalog, err := s.getCatalog(ctx, sourceID)
    if err != nil {
        return nil, err
    }
    
    // 2. Get target catalog
    targetCatalog, err := s.getCatalog(ctx, targetID)
    if err != nil {
        return nil, err
    }
    
    // 3. Compare catalogs
    differences := s.findDifferences(sourceCatalog, targetCatalog)
    
    // 4. Generate comparison report
    result := s.generateComparisonReport(differences)
    
    // 5. Store comparison result
    err = s.storeComparisonResult(ctx, result)
    if err != nil {
        return nil, err
    }
    
    return result, nil
}
```

### 3. Inventory Synchronization
```
1. WMS inventory update → 2. Product service receives event → 3. Updates internal inventory → 4. Syncs to sales channels → 5. Updates search index
```

#### Implementation
```go
func (s *Service) HandleInventoryUpdate(ctx context.Context, event InventoryUpdateEvent) error {
    // 1. Update internal inventory
    err := s.updateInventory(ctx, event.SkuID, event.Quantity)
    if err != nil {
        return err
    }
    
    // 2. Get affected SKUs
    sku, err := s.getSKU(ctx, event.SkuID)
    if err != nil {
        return err
    }
    
    // 3. Update search index
    err = s.updateSearchIndex(ctx, sku)
    if err != nil {
        return err
    }
    
    // 4. Sync to sales channels
    err = s.syncInventoryToChannels(ctx, sku)
    if err != nil {
        return err
    }
    
    // 5. Publish inventory event
    return s.publishInventoryEvent(ctx, event)
}
```

---

## Troubleshooting

### Common Issues

#### 1. Database Connection Issues
```bash
# Check MongoDB connection
mongo --host localhost --port 27017 -u admin -p admin

# Check Redis connection
redis-cli -h localhost -p 7006 ping

# Check service logs
docker logs product-service
```

#### 2. Worker Processing Issues
```bash
# Check worker status
curl http://localhost:8088/health

# Check SQS queue status
aws --endpoint-url=http://localhost:4569 sqs get-queue-attributes \
  --queue-url http://localhost:4569/000000000000/local-product-service-skus

# Check worker logs
docker logs product-service | grep "worker"
```

#### 3. Search Index Issues
```bash
# Recreate search indexes
go run main.go -mode=create_index

# Check MongoDB indexes
mongo --host localhost --port 27017 -u admin -p admin product_service_db
db.skus.getIndexes()
```

#### 4. Performance Issues
```bash
# Check MongoDB query performance
db.skus.find({seller_id: 1}).explain("executionStats")

# Check Redis memory usage
redis-cli -h localhost -p 7006 info memory

# Monitor service metrics
curl http://localhost:8088/metrics
```

### Debug Mode
Enable debug logging in configuration:
```yaml
log:
  level: "debug"
  request: true
  response: true

mongodb:
  debugMode: true
  query:
    print:
      all: true
      find: true
      aggregate: true
```

### Health Checks
```bash
# Service health
curl http://localhost:8088/health

# Database health
curl http://localhost:8088/health/db

# Redis health
curl http://localhost:8088/health/redis
```

---

## Additional Resources

### Related Documentation
- [API Gateway Documentation](./API_GATEWAY_DOCUMENTATION.md)
- [Sales Channel Service Documentation](./SALES_CHANNEL_SERVICE_DOCUMENTATION.md)
- [OMS Service Documentation](./OMS_SERVICE_DOCUMENTATION.md)
- [Customer Service Documentation](./CUSTOMER_SERVICE_DOCUMENTATION.md)

### Code Examples
- [Product Import Example](./examples/product_import.md)
- [Search Implementation](./examples/search_implementation.md)
- [Worker Configuration](./examples/worker_config.md)

### API Reference
- [OpenAPI Specification](./api/openapi.yaml)
- [Postman Collection](./api/postman_collection.json)
- [API Examples](./api/examples/)

---
