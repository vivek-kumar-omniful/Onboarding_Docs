# Omniful Developer Onboarding Guide

## Table of Contents
1. [Company Overview](#company-overview)
2. [Microservices Architecture](#microservices-architecture)
3. [Core Services Deep Dive](#core-services-deep-dive)
4. [Sales Channel Development](#sales-channel-development)
5. [Development Workflow](#development-workflow)
6. [Common Patterns & Best Practices](#common-patterns--best-practices)
7. [Getting Started](#getting-started)

---

## Company Overview

**Omniful** is a comprehensive eCommerce operations platform that enables sellers to manage multiple online stores and marketplaces through a unified system. Our platform acts as a central hub for order management, inventory synchronization, and fulfillment across various sales channels.

### Core Value Proposition
- **Unified Order Management**: Centralize orders from multiple platforms (Shopify, WooCommerce, Amazon, etc.)
- **Real-time Inventory Sync**: Keep stock levels synchronized across all sales channels
- **Automated Fulfillment**: Streamline pick, pack, and ship operations
- **Multi-channel Analytics**: Comprehensive reporting across all platforms

---

## Microservices Architecture

Omniful follows a **microservices architecture** with the following core services:

### Architecture Overview
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   API Gateway   │    │  Sales Channel  │    │   OMS Service   │
│                 │    │    Service      │    │                 │
│ • Authentication│    │ • Platform      │    │ • Order         │
│ • Rate Limiting │    │   Integration   │    │   Processing    │
│ • Routing       │    │ • Webhook       │    │ • Fulfillment   │
│ • Load Balancing│    │   Handling      │    │ • Inventory     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         └───────────────────────┼───────────────────────┘
                                 │
          ┌─────────────────┐    │    ┌─────────────────┐
          │  Product Service│    │    │ Tenant Service  │
          │                 │    │    │                 │
          │ • Catalog Mgmt  │    │    │ • Multi-tenancy │
          │ • SKU Management│    │    │ • Configuration │
          │ • Search        │    │    │ • Settings      │
          └─────────────────┘    │    └─────────────────┘
                                 │
          ┌─────────────────┐    │    ┌─────────────────┐
          │ Customer Service│    │    │   WMS Service   │
          │                 │    │    │                 │
          │ • Customer Data │    │    │ • Warehouse     │
          │ • Profiles      │    │    │   Operations    │
          │ • Preferences   │    │    │ • Picking       │
          └─────────────────┘    │    └─────────────────┘
                                 │
                                 ▼
                        ┌─────────────────┐
                        │   External      │
                        │   Platforms     │
                        │                 │
                        │ • Shopify       │
                        │ • WooCommerce   │
                        │ • Amazon        │
                        │ • Zid           │
                        │ • Salla         │
                        └─────────────────┘
```

---

## Core Services Deep Dive

### 1. API Gateway (`api-gateway/`)

**Purpose**: Single entry point for all external API requests

**Key Responsibilities**:
- **Authentication & Authorization**: JWT validation, role-based access control
- **Rate Limiting**: Prevent API abuse
- **Request Routing**: Route requests to appropriate microservices
- **Load Balancing**: Distribute traffic across service instances
- **Request/Response Transformation**: Standardize data formats

**Key Files**:
```
api-gateway/
├── main.go                    # Service entry point
├── router/
│   └── router.go             # Route definitions
├── controllers/              # Request handlers
├── internal/
│   └── middlewares/          # Authentication, rate limiting
└── external_service/         # Service clients
```

**Common Endpoints**:
- `GET /api/v1/sales_channels/sellers/{seller_id}/sales_channels/{ssc_id}/sync_products`
- `POST /api/v1/orders`
- `GET /api/v1/products`

**Detailed Documentation**: [API Gateway Documentation](./API_GATEWAY_DOCUMENTATION.md)

### 2. Sales Channel Service (`sales-channel-service/`)

**Purpose**: Bridge between Omniful and external eCommerce platforms

**Key Responsibilities**:
- **Platform Integration**: Connect with Shopify, WooCommerce, Amazon, etc.
- **Webhook Processing**: Handle real-time events from external platforms
- **Data Synchronization**: Sync orders, products, and inventory
- **Order Processing**: Convert external orders to Omniful format
- **Inventory Management**: Keep stock levels synchronized

**Key Files**:
```
sales-channel-service/
├── main.go                    # Service entry point
├── internal/
│   ├── sales_channels/        # Platform-specific implementations
│   │   ├── shopify/           # Shopify integration
│   │   ├── woo_commerce_v2/   # WooCommerce integration
│   │   └── sales_channel_factory.go  # Factory pattern
│   ├── workers/               # Background job processors
│   └── sales_channel_commons/ # Shared functionality
├── router/                    # Internal routes
└── external_service/          # External API clients
```

**Common Flows**:
1. **Order Sync**: External platform → Webhook → SQS → OMS
2. **Product Sync**: Manual trigger → API → Product Service
3. **Inventory Sync**: WMS update → Kafka → External platform

** Detailed Documentation**: [Sales Channel Service Documentation](./Sales_Channel_Service_Documentation.md)

** Implementation Guide**: [Sales Channel Implementation Guide](./Sales_Channel_Implementation_Guide.md) - Complete 14-phase developer guide for building new integrations

### 3. OMS Service (`oms-service/`)

**Purpose**: Order Management System - Core business logic for order processing

**Key Responsibilities**:
- **Order Lifecycle Management**: From creation to fulfillment
- **Inventory Allocation**: Reserve stock for orders
- **Fulfillment Coordination**: Manage pick, pack, ship process
- **Order Status Tracking**: Monitor order progress
- **Returns Processing**: Handle customer returns
- **Hub Management**: Route orders to appropriate warehouses
- **Sales Channel Integration**: Platform-specific order handling
- **Background Processing**: Asynchronous order operations

**Key Files**:
```
oms-service/
├── main.go                    # Service entry point
├── core/                      # Business logic
│   ├── orders/               # Order management
│   ├── inventory/            # Inventory operations
│   ├── hub/                  # Warehouse/hub management
│   ├── shipment/             # Shipment operations
│   └── workers/              # Background processors
├── router/                   # API routes
└── data/                     # Data models
```

** Detailed Documentation**: [OMS Service Documentation](./OMS_SERVICE_DOCUMENTATION.md)

### 4. Product Service (`product-service/`)

**Purpose**: Centralized product catalog and SKU management

**Key Responsibilities**:
- **Product Catalog**: Manage product information
- **SKU Management**: Handle product variants and SKUs
- **Search & Discovery**: Product search functionality
- **Category Management**: Organize products by categories
- **Product Mapping**: Map external products to internal SKUs
- **Bulk Operations**: Handle large-scale imports/exports
- **Worker System**: Asynchronous processing for heavy operations
- **Search Indexing**: MongoDB Atlas Search integration

**Key Files**:
```
product-service/
├── main.go                    # Service entry point
├── internal/                  # Business logic
├── router/                    # API routes
├── index/                     # Search indexing
├── workers/                   # Background job processors
└── pkg/                       # Shared utilities
```

** Detailed Documentation**: [Product Service Documentation](./PRODUCT_SERVICE_DOCUMENTATION.md)

### 5. Customer Service (`customer-service/`)

**Purpose**: Customer data and profile management

**Key Responsibilities**:
- **Customer Profiles**: Store customer information
- **Preferences**: Manage customer preferences
- **Address Management**: Handle shipping/billing addresses
- **Customer Analytics**: Track customer behavior
- **Document Management**: Store and validate customer documents
- **Authentication**: Customer login and OTP verification
- **Bulk Operations**: Process large-scale customer imports

** Detailed Documentation**: [Customer Service Documentation](./CUSTOMER_SERVICE_DOCUMENTATION.md)

### 6. Tenant Service (`tenant-service/`)

**Purpose**: Multi-tenancy and configuration management

**Key Responsibilities**:
- **Tenant Isolation**: Separate data between customers
- **Configuration Management**: Store tenant-specific settings
- **Feature Flags**: Enable/disable features per tenant
- **Billing & Subscription**: Manage tenant subscriptions
- **User Authentication**: Handle user login, OTP verification, and JWT tokens
- **Role & Permission Management**: Define and manage user roles and permissions
- **Address & Location Services**: Handle geographic data and address validation
- **Integration Management**: Manage external service integrations per tenant

**Key Files**:
```
tenant-service/
├── main.go                    # Service entry point
├── internal/                  # Business logic
│   ├── tenant/               # Tenant management
│   ├── omniful_user/         # User management
│   ├── omniful_role/         # Role management
│   ├── address/              # Address management
│   └── workers/              # Background job processors
├── router/                    # API routes
└── deployment/                # Database migrations
```

**Detailed Documentation**: [Tenant Service Documentation](./TENANT_SERVICE_DOCUMENTATION.md)

### 7. WMS Service (`wms-service/`)

**Purpose**: Warehouse Management System - Comprehensive warehouse operations and inventory management

**Key Responsibilities**:
- **Warehouse Layout**: Manage zones, floors, aisles, racks, shelves, and bins
- **Inventory Management**: Track inventory levels, locations, batches, and stock movements
- **Order Fulfillment**: Handle picking, packing, and shipping operations
- **Goods Receipt**: Process incoming goods, GRN management, and supplier operations
- **Quality Control**: Manage cycle counting, QC operations, and inventory audits
- **Batch Management**: Handle batch tracking, expiry dates, and serialization
- **Location Management**: Manage SKU-to-location mappings and storage optimization
- **Reporting**: Generate warehouse reports and analytics
- **Barcode Management**: Handle barcode generation and scanning operations

**Key Files**:
```
wms-service/
├── main.go                    # Service entry point
├── internal/                  # Business logic
│   ├── zone/                 # Zone management
│   ├── location/             # Location management
│   ├── inventory/            # Inventory operations
│   ├── picklist/             # Picking operations
│   ├── wave/                 # Wave management
│   ├── grn/                  # Goods Receipt Note
│   ├── cycle_count/          # Cycle counting
│   ├── batch/                # Batch management
│   └── workers/              # Background job processors
├── router/                    # API routes
└── deployment/                # Database migrations
```

**Detailed Documentation**: [WMS Service Documentation](./WMS_SERVICE_DOCUMENTATION.md)

---

## Sales Channel Development

> **Implementation Guide**: For comprehensive step-by-step implementation details, refer to the **[Sales Channel Implementation Guide](./Sales_Channel_Implementation_Guide.md)** which covers all 14 phases of building new integrations.

### Understanding Sales Channels

**Sales Channel**: A platform where sellers can create their own branded online store
- **Examples**: Shopify, WooCommerce, Zid, Salla
- **Characteristics**: Seller-owned, full branding control, independent storefront

**Marketplace**: A centralized platform where multiple sellers list products
- **Examples**: Amazon, Noon, Landmark, Trendyol
- **Characteristics**: Platform-owned, shared interface, marketplace controls

### Sales Channel Structure

Every sales channel follows a consistent structure:

```
sales-channel-service/internal/sales_channels/{platform_name}/
├── configurations/           # Platform-specific configurations
│   ├── web_interact_config.go    # API endpoint URLs
│   ├── default_configurations.go # Default settings
│   ├── oauth_config.go           # Authentication config
│   └── default_order_status.go   # Status mappings
├── models/                   # Data models
├── requests/                 # API request structures
├── responses/                # API response structures
├── service.go               # Main service implementation
├── controller.go            # HTTP handlers
├── webhook_controller.go    # Webhook handlers
├── web_interact_service.go  # External API interactions
├── update_inventory.go      # Inventory sync logic
├── update_order.go          # Order update logic
├── products.go              # Product sync logic
├── order.go                 # Order processing logic
├── integration.go           # Platform integration setup
├── provider.go              # Dependency injection
├── wire.go                  # Wire dependency injection
└── wire_gen.go              # Generated wire code
```

### Required Files for New Sales Channel

#### 1. **Essential Files** (Must Have)

**`configurations/web_interact_config.go`**
```go
package configurations

import (
    "context"
    "fmt"
    "net/url"
    "github.com/omniful/go_commons/config"
    "github.com/omniful/sales-channel-service/constants"
)

// API endpoint URL generators
func GetFetchProductsURL(shopDomain string, queryParams url.Values) string {
    return "https://" + shopDomain + constants.PlatformFetchProductsURL + "?" + queryParams.Encode()
}

func GetFetchOrdersURL(shopDomain string, queryParams url.Values) string {
    return "https://" + shopDomain + constants.PlatformFetchOrdersURL + "?" + queryParams.Encode()
}

func GetUpdateInventoryURL(shopDomain string, productID string, queryParams url.Values) string {
    return "https://" + shopDomain + fmt.Sprintf(constants.PlatformUpdateInventoryURL, productID) + "?" + queryParams.Encode()
}

func GetCreateWebhookURL(shopDomain string, queryParams url.Values) string {
    return "https://" + shopDomain + constants.PlatformCreateWebhookURL + "?" + queryParams.Encode()
}
```

**`configurations/default_configurations.go`**
```go
package configurations

import (
    "context"
    "github.com/omniful/go_commons/config"
)

type PlatformConfig struct {
    StockReleaseState bool
    ExternalWebhookToken string
    // Platform-specific configurations
}

func GetPlatformConfig(ctx context.Context) PlatformConfig {
    return PlatformConfig{
        StockReleaseState: config.GetBool(ctx, "salesChannels.platform.stockReleaseState"),
        ExternalWebhookToken: config.GetString(ctx, "salesChannels.platform.externalWebhookToken"),
    }
}
```

**`service.go`**
```go
package platform_name

import (
    "context"
    "github.com/omniful/sales-channel-service/internal/domain/interfaces"
)

type Service struct {
    // Dependencies
}

func (s *Service) FetchProducts(ctx context.Context, eventAttributes models.FetchProductEventAttributes) commonError.CustomError {
    // Implementation for fetching products from platform
}

func (s *Service) FetchOrders(ctx context.Context, eventAttributes models.FetchOrderEventAttributes) commonError.CustomError {
    // Implementation for fetching orders from platform
}

func (s *Service) UpdateInventory(ctx context.Context, eventAttributes models.UpdateInventoryEventAttributes) commonError.CustomError {
    // Implementation for updating inventory on platform
}

// Implement interfaces.SalesChannel
func (s *Service) GetSalesChannelID() int {
    return enums.PlatformSalesChannelID
}

func (s *Service) GetSalesChannelTag() string {
    return enums.PlatformSalesChannelTag
}
```

**`webhook_controller.go`**
```go
package platform_name

import (
    "github.com/gin-gonic/gin"
)

func (controller *Controller) PublicWebhookHandler(ctx *gin.Context) {
    // Handle incoming webhooks from platform
    // 1. Verify webhook authenticity
    // 2. Parse webhook payload
    // 3. Route to appropriate handler
    // 4. Send to SQS for processing
}
```

**`web_interact_service.go`**
```go
package platform_name

import (
    "context"
    "github.com/omniful/sales-channel-service/internal/sales_channels/platform_name/configurations"
)

func (s *Service) makeAPICall(ctx context.Context, url string, method string, body interface{}) (*http.Response, error) {
    // Generic API call implementation
}

func (s *Service) fetchProductsFromPlatform(ctx context.Context, shopDomain string, queryParams url.Values) ([]models.Product, error) {
    url := configurations.GetFetchProductsURL(shopDomain, queryParams)
    // Implementation
}
```

#### 2. **Optional Files** (Platform-Specific)

- **`oauth_config.go`**: OAuth authentication flow
- **`default_order_status.go`**: Status mapping between platform and Omniful
- **`default_sync_events.go`**: Webhook event configurations
- **`update_price.go`**: Price synchronization logic
- **`return_order.go`**: Return order processing

### Integration Steps

#### Step 1: Add to Factory
Update `sales_channel_factory.go`:
```go
var salesChannelConstructorMap = map[int]salesChannelConstructor{
    // ... existing mappings
    enums.NewPlatformSalesChannelID: initNewPlatformService,
}

var salesChannelConstructorMapByTag = map[string]salesChannelConstructor{
    // ... existing mappings
    enums.NewPlatformSalesChannelTag: initNewPlatformService,
}

func initNewPlatformService(ctx context.Context) (interfaces.SalesChannel, error) {
    // Initialize service with dependencies
}
```

#### Step 2: Add Enums
Update `internal/enums/sales_channel.go`:
```go
const (
    NewPlatformSalesChannelID = 999 // Unique ID
    NewPlatformSalesChannelTag = "new_platform"
)
```

#### Step 3: Add Constants
Update `constants/sales_channel_constants.go`:
```go
const (
    NewPlatformFetchProductsURL = "/api/products"
    NewPlatformFetchOrdersURL = "/api/orders"
    NewPlatformUpdateInventoryURL = "/api/products/%s/inventory"
    NewPlatformCreateWebhookURL = "/api/webhooks"
)
```

#### Step 4: Add Wire Configuration
Create `wire.go`:
```go
//go:build wireinject
// +build wireinject

package platform_name

import (
    "github.com/google/wire"
)

func InitializeService(ctx context.Context) (*Service, error) {
    wire.Build(
        // Dependencies
        NewService,
        // ... other dependencies
    )
    return &Service{}, nil
}
```

#### Step 5: Generate Wire Code
```bash
cd sales-channel-service/internal/sales_channels/new_platform
go generate
```

### Common Patterns

#### 1. **Webhook Processing Pattern**
```go
func (controller *Controller) PublicWebhookHandler(ctx *gin.Context) {
    // 1. Extract webhook signature
    signature := ctx.GetHeader("X-Platform-Signature")
    
    // 2. Verify webhook authenticity
    if !verifyWebhook(ctx, body, signature) {
        ctx.JSON(http.StatusUnauthorized, gin.H{"error": "Invalid signature"})
        return
    }
    
    // 3. Parse webhook payload
    var webhookPayload models.WebhookPayload
    if err := ctx.ShouldBindJSON(&webhookPayload); err != nil {
        ctx.JSON(http.StatusBadRequest, gin.H{"error": "Invalid payload"})
        return
    }
    
    // 4. Create SQS event
    event := sqs_events.CreateWebhookEvent(webhookPayload)
    
    // 5. Send to SQS queue
    err := pushToSQS.SendMessageToWebhookQueue(ctx, event)
    if err != nil {
        ctx.JSON(http.StatusInternalServerError, gin.H{"error": "Failed to process"})
        return
    }
    
    ctx.JSON(http.StatusOK, gin.H{"status": "success"})
}
```

#### 2. **API Call Pattern**
```go
func (s *Service) makeAPICall(ctx context.Context, url string, method string, body interface{}) (*http.Response, error) {
    client := &http.Client{Timeout: 30 * time.Second}
    
    var req *http.Request
    var err error
    
    if body != nil {
        jsonBody, err := json.Marshal(body)
        if err != nil {
            return nil, err
        }
        req, err = http.NewRequestWithContext(ctx, method, url, bytes.NewBuffer(jsonBody))
    } else {
        req, err = http.NewRequestWithContext(ctx, method, url, nil)
    }
    
    if err != nil {
        return nil, err
    }
    
    // Add headers
    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("Authorization", "Bearer "+s.accessToken)
    
    return client.Do(req)
}
```

#### 3. **Error Handling Pattern**
```go
func (s *Service) handleAPIError(response *http.Response, body []byte) commonError.CustomError {
    if response.StatusCode >= 400 {
        return commonError.NewCustomError(
            scsError.BadRequest,
            fmt.Sprintf("API call failed: %s", string(body)),
            commonError.WithShouldNotify(true),
        )
    }
    return commonError.CustomError{}
}
```

---

## Development Workflow

### Branching Strategy
```
master (production)
    ↑
staging (pre-production)
    ↑
dev (development)
    ↑
feature branches
```

### Development Process

1. **Create Feature Branch**
   ```bash
   git checkout staging
   git pull origin staging
   git checkout -b feat/new-platform-integration
   ```

2. **Develop Feature**
   - Implement required files
   - Add tests
   - Update documentation

3. **Create Development Branch**
   ```bash
   git checkout -b feat/new-platform-integration-dev
   git pull origin dev
   ```

4. **Push Both Branches**
   ```bash
   git push origin feat/new-platform-integration
   git push origin feat/new-platform-integration-dev
   ```

5. **Create Pull Requests**
   - `feat/new-platform-integration-dev` → `dev`
   - `feat/new-platform-integration` → `staging`

6. **Review & Merge**
   - Merge dev PR first
   - Test in dev environment
   - Merge staging PR

### Testing Strategy

1. **Unit Tests**: Test individual functions
2. **Integration Tests**: Test API endpoints
3. **End-to-End Tests**: Test complete workflows
4. **Platform-Specific Tests**: Test with actual platform APIs

---

## Common Patterns & Best Practices

### 1. **Configuration Management**
- Use environment variables for sensitive data
- Store platform-specific configs in `configurations/` directory
- Use feature flags for gradual rollouts

### 2. **Error Handling**
- Use custom error types for different scenarios
- Log errors with context
- Implement retry mechanisms for transient failures

### 3. **Logging**
- Use structured logging
- Include request IDs for traceability
- Log at appropriate levels (DEBUG, INFO, WARN, ERROR)

### 4. **API Design**
- Follow RESTful conventions
- Use consistent response formats
- Implement proper HTTP status codes

### 5. **Database Patterns**
- Use transactions for data consistency
- Implement proper indexing
- Handle database migrations carefully

### 6. **Caching Strategy**
- Cache frequently accessed data
- Use Redis for distributed caching
- Implement cache invalidation strategies

---

## Getting Started

### Prerequisites
- Go 1.21+
- Docker & Docker Compose
- PostgreSQL
- Redis
- AWS CLI (for SQS/SNS)

### Local Development Setup

1. **Clone Repository**
   ```bash
   git clone <repository-url>
   cd omniful
   ```

2. **Start Dependencies**
   ```bash
   docker-compose up -d postgres redis
   ```

3. **Configure Environment**
   ```bash
   cp .env.example .env
   # Edit .env with your local configuration
   ```

4. **Run Services**
   ```bash
   # Start API Gateway
   cd api-gateway && go run main.go
   
   # Start Sales Channel Service
   cd sales-channel-service && go run main.go
   
   # Start OMS Service
   cd oms-service && go run main.go
   
   # Start Product Service
   cd product-service && go run main.go
   ```

### First Task: Create a Simple Sales Channel

> **Complete Guide**: For detailed implementation steps, refer to the **[Sales Channel Implementation Guide](./Sales_Channel_Implementation_Guide.md)** which provides a comprehensive 14-phase approach.

1. **Choose a Platform**: Start with a simple platform (e.g., a REST API-based platform)

2. **Create Directory Structure**:
   ```bash
   cd sales-channel-service/internal/sales_channels
   mkdir new_platform
   cd new_platform
   mkdir configurations models requests responses
   ```

3. **Implement Required Files**: Follow the patterns described above and detailed in the implementation guide

4. **Test Integration**: Use the platform's test environment and follow the testing checklist

5. **Document**: Update documentation with platform-specific details

### Common Issues & Solutions

1. **Webhook Verification Fails**
   - Check signature algorithm
   - Verify secret key configuration
   - Test with platform's webhook testing tool

2. **API Rate Limiting**
   - Implement exponential backoff
   - Use rate limiting headers
   - Consider caching responses

3. **Data Mapping Issues**
   - Create comprehensive mapping documentation
   - Test with various data formats
   - Handle edge cases gracefully

---

## Additional Resources

### Documentation
- [Sales Channel Service Overview](./sales-channel-service/docs/Overview.md)
- [Development Workflow](./sales-channel-service/docs/development_workflow.md)
- [API Documentation](./api-docs/)

### Implementation Guides
- **[Sales Channel Implementation Guide](./Sales_Channel_Implementation_Guide.md)** - Complete 14-phase step-by-step guide for building new integrations
- **[Technical Reference Guide](./Technical_Reference_Guide.md)** - Common patterns, file structures, and best practices
- **[Sales Channel Service Documentation](./Sales_Channel_Service_Documentation.md)** - High-level service overview and architecture

### Code Examples
- [Shopify Integration](./sales-channel-service/internal/sales_channels/shopify/)
- [WooCommerce Integration](./sales-channel-service/internal/sales_channels/woo_commerce_v2/)
- [Zid Integration](./sales-channel-service/internal/sales_channels/zid/)

### Tools & Utilities
- [Common Libraries](./pkg/)
- [Configuration Management](./configs/)
- [Database Utilities](./utils/)

**Welcome to Omniful! **
