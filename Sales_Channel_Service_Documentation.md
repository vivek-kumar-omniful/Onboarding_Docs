# Sales Channel Service Documentation

## Table of Contents
1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Service Modes](#service-modes)
4. [Order Processing Flow](#order-processing-flow)
5. [Integration Patterns](#integration-patterns)
6. [Authentication Flows](#authentication-flows)
7. [Webhook Processing](#webhook-processing)
8. [Sync Mechanisms](#sync-mechanisms)
9. [Creating New Integrations](#creating-new-integrations)
10. [Background Workers](#background-workers)
11. [Error Handling](#error-handling)
12. [Configuration](#configuration)
13. [Monitoring & Observability](#monitoring--observability)

---

## Overview

The **Sales Channel Service** is the core integration layer of the Omniful platform, responsible for connecting with external eCommerce platforms and managing data synchronization. It acts as a bridge between external platforms (Shopify, WooCommerce, Amazon, etc.) and Omniful's internal services.

### Key Responsibilities
- **Platform Integration**: Connect with 30+ eCommerce platforms
- **Data Synchronization**: Sync orders, products, inventory, and returns
- **Webhook Processing**: Handle real-time events from external platforms
- **Order Processing**: Convert external orders to Omniful format
- **Authentication Management**: Handle platform-specific authentication
- **Rate Limiting**: Prevent API abuse and ensure fair usage
- **Background Processing**: Asynchronous data processing via SQS

---

## Architecture

### High-Level Architecture
```
┌─────────────────────────────────────────────────────────────┐
│                External Platforms                           │
│   ┌──────────────┐ ┌──────────────┐ ┌──────────────┐        │
│   │   Shopify    │ │ WooCommerce  │ │   Amazon     │        │
│   │              │ │              │ │              │        │
│   └──────────────┘ └──────────────┘ └──────────────┘        │
│   ┌──────────────┐ ┌──────────────┐ ┌──────────────┐        │
│   │    Salla     │ │     Zid      │ │   Noon       │        │
│   │              │ │              │ │              │        │
│   └──────────────┘ └──────────────┘ └──────────────┘        │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│                Sales Channel Service                        │
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────┐ │
│  │   HTTP Server   │  │  Background     │  │   Factory    │ │
│  │   (API Layer)   │  │   Workers       │  │   Pattern    │ │
│  └─────────────────┘  └─────────────────┘  └──────────────┘ │
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────┐ │
│  │   Controllers   │  │   Services      │  │   Models     │ │
│  │                 │  │                 │  │              │ │
│  └─────────────────┘  └─────────────────┘  └──────────────┘ │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│                Internal Services                            │
│   ┌──────────────┐ ┌──────────────┐ ┌──────────────┐        │
│   │   OMS        │ │   Product    │ │   Customer   │        │
│   │  Service     │ │  Service     │ │  Service     │        │
│   └──────────────┘ └──────────────┘ └──────────────┘        │
│   ┌──────────────┐ ┌──────────────┐ ┌──────────────┐        │
│   │   Tenant     │ │     WMS      │ │   Analytics  │        │
│   │  Service     │ │  Service     │ │  Service     │        │
│   └──────────────┘ └──────────────┘ └──────────────┘        │
└─────────────────────────────────────────────────────────────┘
```

### Directory Structure
```
sales-channel-service/
├── main.go                           # Service entry point with mode selection
├── router/
│   └── gateway_routes.go            # Internal API routes
├── internal/
│   ├── sales_channels/              # Platform-specific implementations
│   │   ├── shopify/                 # Shopify integration
│   │   ├── woo_commerce_v2/         # WooCommerce integration
│   │   ├── salla_v2/                # Salla integration
│   │   ├── zid/                     # Zid integration
│   │   ├── amazon/                  # Amazon integration
│   │   ├── noon/                    # Noon integration
│   │   └── sales_channel_factory.go # Factory pattern implementation
│   ├── sales_channel_commons/       # Shared functionality
│   │   ├── controller.go            # Common controller logic
│   │   ├── service.go               # Common service logic
│   │   └── webhook_commons.go       # Webhook processing utilities
│   ├── workers/                     # Background job processors
│   │   ├── handlers/
│   │   │   ├── orders_handlers/     # Order processing workers
│   │   │   ├── products_handlers/   # Product processing workers
│   │   │   └── inventory_handlers/  # Inventory processing workers
│   │   └── workers.go               # Worker initialization
│   └── enums/                       # Sales channel enums and constants
├── external_service/                # External API clients
│   ├── controllers/push_to_sqs/     # SQS message sending utilities
│   └── service/                     # External service clients
├── utils/
│   └── scs_sqs_events/              # SQS event creation utilities
├── configs/                         # Configuration management
├── constants/                       # Application constants
└── docs/                           # Service documentation
```

---

## Service Modes

The Sales Channel Service can run in three different modes:

### 1. HTTP Server Mode (Default)
```bash
go run main.go -mode=http
```
- **Purpose**: Serves internal API endpoints for other microservices
- **Port**: Configured via `server.port` in config
- **Endpoints**: Sync triggers, webhook handlers, configuration management

### 2. Worker Mode
```bash
go run main.go -mode=worker
```
- **Purpose**: Processes background jobs from SQS queues
- **Workers**: Order sync, product sync, inventory sync, webhook processing
- **Scaling**: Can run multiple worker instances

### 3. Migration Mode
```bash
go run main.go -mode=migration -migrationType=up
```
- **Purpose**: Database schema migrations
- **Types**: `up`, `down`, `force`
- **Location**: `deployment/migration/`

---

## Order Processing Flow

### Complete Order Sync Flow

```
1. External Request (API Gateway)
   GET /api/v1/sales_channels/sellers/{seller_id}/sales_channels/{ssc_id}/sync_products?sync_entity=orders
   ↓
2. Sales Channel Service Controller
   SyncEntities() → Validate → Rate Limit Check
   ↓
3. SQS Event Creation
   CreateFetchOrderEvent() → Send to FIFO Queue
   ↓
4. Background Worker Processing
   FetchOrdersFifo.Process() → Platform-Specific Handler
   ↓
5. External Platform API Call
   Fetch orders from Shopify/WooCommerce/etc.
   ↓
6. Data Transformation
   Convert external format to Omniful format
   ↓
7. OMS Service Integration
   Send processed orders to Order Management System
   ↓
8. Status Update
   Update sync status in Redis
```

### Detailed Flow Breakdown

#### 1. Request Reception & Validation
```go
// From internal/sales_channel_commons/controller.go
func (ac *Controller) SyncEntities(ctx *gin.Context) {
    // Extract parameters
    sscID := ctx.Param(constants.SellerSalesChannelID)
    syncEntity := ctx.Query(constants.SyncEntity)
    syncFromTime := ctx.Query(constants.SyncFromTime)
    syncFromType := ctx.Query(constants.SyncFromType)
    
    // Validate sync entity type
    if syncEntity != constants.SyncEntityTypeOrder {
        // Return error response
        return
    }
    
    // Check rate limiting
    isValid, _, _, _, _ := ac.salesChannelService.CheckSyncEntityValidity(ctx, sscIntID, syncEntity, freezeTime)
    if !isValid {
        // Return current sync status
        return
    }
    
    // Execute sync
    cusErr := ac.salesChannelService.SyncEntities(ctx, sscID, syncEntity, syncFromTime, syncFromType, freezeTime)
}
```

#### 2. SQS Event Creation
```go
// From utils/scs_sqs_events/sqs_events.go
func CreateFetchOrderEvent(ctx *gin.Context, sellerSalesChannel *scsModels.SellerSalesChannels, syncFromTime string, syncFromType string, freezeTime time.Duration) commonError.CustomError {
    // Create SQS event
    sqsSyncEvenSyncOrdersEvent := sqsModels.SqsSyncIntegrationEvents{
        SellerSalesChannelID: sellerSalesChannel.Id,
        EventType:            constants.SyncOrders,
        SellerID:            sellerSalesChannel.SellerId,
        SalesChannelID:      sellerSalesChannel.SalesChannelId,
        CreatedAtMin:        time.Now().Add(-syncFromDuration).Format("2006-01-02 15:04:05.999999"),
        CreatedAtMax:        time.Now().Format("2006-01-02 15:04:05.999999"),
    }
    
    // Send to SQS FIFO queue
    cusErr = pushToSqs.SendMessageToSCSFetchOrdersFifoQueue(ctx, sqsSyncEvenSyncOrdersEvent)
    
    // Set sync status in Redis
    cusErr = redis.SetSyncEntityInProgress(ctx, sellerSalesChannel.Id, constants.SyncEntityTypeOrder)
}
```

#### 3. Background Worker Processing
```go
// From internal/workers/handlers/orders_handlers/sqs_fetch_order_fifo_handler.go
func (e FetchOrdersFifo) Process(ctx context.Context, sqsMessages *[]sqs.Message) error {
    // Unmarshal SQS message
    if err := json.Unmarshal(ctx, messages[0].Value, &eventAttributes); err != nil {
        return nil // Acknowledge message to prevent infinite retries
    }
    
    // Route based on event type
    switch eventAttributes.EventType {
    case constants.SyncOrders:
        cusErr := e.handleSyncOrders(ctx, eventAttributes)
        if cusErr.Exists() {
            e.purgeOrderSyncEntity(ctx, eventAttributes.SellerSalesChannelID)
            return cusErr.ToError()
        }
    }
}
```

#### 4. Platform-Specific Order Fetching
```go
// From internal/sales_channels/shopify/order.go (example)
func (s *Service) FetchOrders(ctx context.Context, eventAttributes models.FetchOrderEventAttributes) commonError.CustomError {
    // Get authentication details
    auth, cusErr := s.getAuthentication(ctx, eventAttributes.SellerSalesChannelID)
    if cusErr.Exists() {
        return cusErr
    }
    
    // Fetch orders from Shopify API
    orders, cusErr := s.fetchOrdersFromShopify(ctx, auth, eventAttributes)
    if cusErr.Exists() {
        return cusErr
    }
    
    // Transform to Omniful format
    omnifulOrders, cusErr := s.transformOrdersToOmnifulFormat(ctx, orders)
    if cusErr.Exists() {
        return cusErr
    }
    
    // Send to OMS Service
    cusErr = pushToSQS.SendMessageToOrderServiceOrderQueue(ctx, omnifulOrders)
    return cusErr
}
```

---

## Integration Patterns

### Factory Pattern Implementation

The Sales Channel Service uses a factory pattern to manage multiple platform integrations:

```go
// From internal/sales_channels/sales_channel_factory.go
type SalesChannelFactory struct {
    salesChannels      map[int]interfaces.SalesChannel
    salesChannelsByTag map[string]interfaces.SalesChannel
}

var salesChannelConstructorMap = map[int]salesChannelConstructor{
    enums.ShopifySalesChannelID:       initShopifyService,
    enums.WooCommerceV2SalesChannelID: initWooCommerceV2Service,
    enums.SallaV2SalesChannelID:       initSallaV2Service,
    enums.ZidSalesChannelID:           initZidService,
    // ... more platforms
}

func NewSalesChannelFactory(ctx context.Context) (*SalesChannelFactory, error) {
    factory := &SalesChannelFactory{
        salesChannels:      make(map[int]interfaces.SalesChannel),
        salesChannelsByTag: make(map[string]interfaces.SalesChannel),
    }
    
    // Initialize all sales channel services
    for scID, scConstructor := range salesChannelConstructorMap {
        service, err := scConstructor(ctx)
        if err != nil {
            return nil, err
        }
        factory.salesChannels[scID] = service
    }
    
    return factory, nil
}
```

### Platform Interface Contract

All sales channel implementations must implement the `SalesChannel` interface:

```go
// From internal/domain/interfaces/sales_channel.go
type SalesChannel interface {
    // Core identification
    GetSalesChannelID() int
    GetSalesChannelTag() string
    
    // Data synchronization
    FetchProducts(ctx context.Context, eventAttributes models.FetchProductEventAttributes) commonError.CustomError
    FetchOrders(ctx context.Context, eventAttributes models.FetchOrderEventAttributes) commonError.CustomError
    UpdateInventory(ctx context.Context, eventAttributes models.UpdateInventoryEventAttributes) commonError.CustomError
    
    // Webhook processing
    ProcessWebhook(ctx *gin.Context, body []byte) commonError.CustomError
    
    // Order management
    UpdateOrderStatus(ctx context.Context, orderID string, status string) commonError.CustomError
    CreateFulfillment(ctx context.Context, orderID string, trackingInfo models.TrackingInfo) commonError.CustomError
}
```

---

## Authentication Flows

### Platform-Specific Authentication

Different platforms use various authentication mechanisms:

#### 1. OAuth 2.0 (Shopify, WooCommerce)
```go
// From internal/sales_channels/shopify/oauth.go
func (s *Service) HandleOAuthCallback(ctx *gin.Context) commonError.CustomError {
    // Extract authorization code
    code := ctx.Query("code")
    shop := ctx.Query("shop")
    
    // Exchange code for access token
    tokenResponse, err := s.exchangeCodeForToken(ctx, code, shop)
    if err != nil {
        return commonError.NewCustomError(scsError.OAuthError, err.Error())
    }
    
    // Store authentication details
    auth := models.SellerSalesChannelAuthentication{
        SellerSalesChannelID: sscID,
        AccessToken:          tokenResponse.AccessToken,
        RefreshToken:         tokenResponse.RefreshToken,
        ExpiresAt:           time.Now().Add(time.Duration(tokenResponse.ExpiresIn) * time.Second),
    }
    
    return s.repository.UpsertAuthentication(ctx, &auth)
}
```

#### 2. API Key Authentication (Amazon, Noon)
```go
// From internal/sales_channels/amazon/authentication.go
func (s *Service) AuthenticateWithAPIKey(ctx context.Context, apiKey string, secretKey string) commonError.CustomError {
    // Validate API credentials
    isValid, err := s.validateAPICredentials(ctx, apiKey, secretKey)
    if err != nil {
        return commonError.NewCustomError(scsError.AuthenticationError, err.Error())
    }
    
    if !isValid {
        return commonError.NewCustomError(scsError.InvalidCredentials, "Invalid API credentials")
    }
    
    // Store encrypted credentials
    encryptedSecret := s.encryptSecret(secretKey)
    auth := models.SellerSalesChannelAuthentication{
        SellerSalesChannelID: sscID,
        APIKey:              apiKey,
        SecretKey:           encryptedSecret,
    }
    
    return s.repository.UpsertAuthentication(ctx, &auth)
}
```

#### 3. Webhook Token Authentication
```go
// From internal/sales_channel_commons/webhook_commons.go
func (s *Service) FetchSSCAndSSCAuthentication(ctx context.Context, storeUniqueIdentifierKey string, storeUniqueIdentifierValue string, appID string) (sellerSalesChannel models.SellerSalesChannels, sellerSalesChannelAuthentication models.SellerSalesChannelAuthentication, salesChannels *entities.SalesChannels, cusErr commonError.CustomError) {
    // Get authentication details
    sellerSalesChannelAuthentication, cusErr = s.getAuthentication(ctx, storeUniqueIdentifierKey, storeUniqueIdentifierValue, appID)
    if cusErr.Exists() {
        return
    }
    
    // Get seller sales channel details
    sellerSalesChannel, cusErr = s.getSellerSalesChannel(ctx, sellerSalesChannelAuthentication.SellerSalesChannelID)
    if cusErr.Exists() {
        return
    }
    
    // Get sales channel details
    salesChannels, cusErr = s.getSalesChannel(ctx, sellerSalesChannel.SalesChannelID)
    return
}
```

---

## Webhook Processing

### Webhook Reception Flow

```
1. External Platform Webhook
   POST /internal/v1/sales_channels/webhooks/{platform}
   ↓
2. Platform-Specific Handler
   Verify signature → Parse payload → Create SQS event
   ↓
3. SQS Queue Processing
   Background worker processes webhook event
   ↓
4. Data Processing
   Fetch additional data → Transform → Send to OMS
   ↓
5. Response
   200 OK to external platform
```

### Webhook Signature Verification

#### Shopify Webhook Verification
```go
// From internal/sales_channels/shopify/webhook_controller.go
func verifyWebhook(ctx *gin.Context, body []byte, hmacHeader string, shopDomain string) bool {
    // Get webhook secret from configuration
    secret := config.GetString(ctx, "salesChannels.shopify.webhookSecret")
    
    // Calculate HMAC SHA256
    expectedSignature := hmac.New(sha256.New, []byte(secret))
    expectedSignature.Write(body)
    expectedHmac := base64.StdEncoding.EncodeToString(expectedSignature.Sum(nil))
    
    // Compare signatures
    return hmac.Equal([]byte(expectedHmac), []byte(hmacHeader))
}

func (shopifyController *Controller) PublicWebhookHandler(ctx *gin.Context) {
    // Read request body
    body, err := io.ReadAll(ctx.Request.Body)
    if err != nil {
        ctx.JSON(http.StatusBadRequest, gin.H{"error": "Failed to read body"})
        return
    }
    
    // Verify webhook signature
    hmacHeader := ctx.GetHeader("X-Shopify-Hmac-Sha256")
    shopDomain := ctx.GetHeader("X-Shopify-Shop-Domain")
    
    if !verifyWebhook(ctx, body, hmacHeader, shopDomain) {
        ctx.JSON(http.StatusUnauthorized, gin.H{"error": "Invalid signature"})
        return
    }
    
    // Create SQS event
    event := sqs_events.CreateWebhookEvent(body)
    
    // Send to SQS queue
    err = pushToSQS.SendMessageToWebhookQueue(ctx, event)
    if err != nil {
        ctx.JSON(http.StatusInternalServerError, gin.H{"error": "Failed to process"})
        return
    }
    
    ctx.JSON(http.StatusOK, gin.H{"status": "success"})
}
```

#### WooCommerce Webhook Verification
```go
// From internal/sales_channels/woo_commerce_v2/webhook_process_service.go
func (s *Service) ProcessWebhook(ctx *gin.Context, body []byte) commonError.CustomError {
    // Extract webhook signature
    signature := ctx.GetHeader("X-WC-Webhook-Signature")
    
    // Verify signature using HMAC SHA256
    if !s.verifyWebhookSignature(body, signature) {
        return commonError.NewCustomError(scsError.InvalidWebhookSignature, "Invalid webhook signature")
    }
    
    // Parse webhook payload
    var webhookPayload models.WooCommerceWebhookPayload
    if err := json.Unmarshal(body, &webhookPayload); err != nil {
        return commonError.NewCustomError(scsError.InvalidPayload, "Invalid webhook payload")
    }
    
    // Process based on webhook topic
    switch webhookPayload.Topic {
    case "order.created":
        return s.handleOrderCreated(ctx, webhookPayload)
    case "order.updated":
        return s.handleOrderUpdated(ctx, webhookPayload)
    case "product.updated":
        return s.handleProductUpdated(ctx, webhookPayload)
    }
    
    return commonError.CustomError{}
}
```

---

## Sync Mechanisms

### Rate Limiting & Freeze Time

The service implements rate limiting to prevent API abuse:

```go
// From internal/sales_channel_commons/service.go
func (s *Service) CheckSyncEntityValidity(ctx context.Context, sscID int, syncEntity string, freezeTime time.Duration) (isValid bool, lastSync string, retryAfter string, enableButton bool, currentStatus string) {
    // Check if sync is in progress
    isInProgress, cusErr := redis.IsSyncEntityInProgress(ctx, sscID, syncEntity)
    if cusErr.Exists() {
        return false, "", "", false, constants.RedisSyncDisabled
    }
    
    if isInProgress {
        return false, "", "", false, constants.RedisSyncInProgress
    }
    
    // Check last sync time
    lastSyncTime, cusErr := redis.GetLastSyncTime(ctx, sscID, syncEntity)
    if cusErr.Exists() {
        return true, "", "", true, constants.RedisSyncSynced
    }
    
    // Check if enough time has passed
    timeSinceLastSync := time.Now().UTC().Sub(lastSyncTime)
    if timeSinceLastSync < freezeTime {
        retryAfter := freezeTime - timeSinceLastSync
        return false, lastSyncTime.Format("2006-01-02 15:04:05"), retryAfter.String(), false, constants.RedisSyncDisabled
    }
    
    return true, lastSyncTime.Format("2006-01-02 15:04:05"), "", true, constants.RedisSyncSynced
}
```

### Sync Entity Types

The service supports four main sync entity types:

1. **Products**: Catalog synchronization
2. **Orders**: Order data synchronization
3. **Inventory**: Stock level synchronization
4. **Returns**: Return request synchronization

### Batch Processing

For large datasets, the service implements batch processing:

```go
// From internal/workers/handlers/orders_handlers/sqs_fetch_order_fifo_handler.go
func (e FetchOrdersFifo) handleSyncNewOrdersInBatches(ctx context.Context, eventAttributes sqsModels.SqsSyncIntegrationEvents) commonError.CustomError {
    // Get sales channel service
    salesChannelService, cusErr := e.salesChannelFactory.GetSalesChannelByID(eventAttributes.SalesChannelID)
    if cusErr.Exists() {
        return cusErr
    }
    
    // Process orders in batches
    batchSize := 50
    offset := 0
    
    for {
        // Fetch batch of orders
        orders, hasMore, cusErr := salesChannelService.FetchOrdersBatch(ctx, eventAttributes, offset, batchSize)
        if cusErr.Exists() {
            return cusErr
        }
        
        // Process batch
        cusErr = e.processOrderBatch(ctx, orders)
        if cusErr.Exists() {
            return cusErr
        }
        
        if !hasMore {
            break
        }
        
        offset += batchSize
    }
    
    return commonError.CustomError{}
}
```

---

## Creating New Integrations

### Step-by-Step Integration Guide

#### 1. Create Directory Structure
```bash
mkdir -p sales-channel-service/internal/sales_channels/new_platform/{configurations,models,requests,responses}
```

#### 2. Add Platform Enums
```go
// internal/enums/sales_channel.go
const (
    NewPlatformSalesChannelID = 999
    NewPlatformSalesChannelTag = "new_platform"
)
```

#### 3. Add Platform Constants
```go
// constants/sales_channel_constants.go
const (
    NewPlatformFetchProductsURL = "/api/products"
    NewPlatformFetchOrdersURL = "/api/orders"
    NewPlatformUpdateInventoryURL = "/api/products/%s/inventory"
    NewPlatformCreateWebhookURL = "/api/webhooks"
)
```

#### 4. Create Configuration Files

**`configurations/web_interact_config.go`**
```go
package configurations

import (
    "context"
    "fmt"
    "net/url"
    "github.com/omniful/sales-channel-service/constants"
)

func GetFetchProductsURL(shopDomain string, queryParams url.Values) string {
    return "https://" + shopDomain + constants.NewPlatformFetchProductsURL + "?" + queryParams.Encode()
}

func GetFetchOrdersURL(shopDomain string, queryParams url.Values) string {
    return "https://" + shopDomain + constants.NewPlatformFetchOrdersURL + "?" + queryParams.Encode()
}

func GetUpdateInventoryURL(shopDomain string, productID string, queryParams url.Values) string {
    return "https://" + shopDomain + fmt.Sprintf(constants.NewPlatformUpdateInventoryURL, productID) + "?" + queryParams.Encode()
}
```

**`configurations/default_configurations.go`**
```go
package configurations

import (
    "context"
    "github.com/omniful/go_commons/config"
)

type NewPlatformConfig struct {
    StockReleaseState bool
    ExternalWebhookToken string
    APIBaseURL string
}

func GetNewPlatformConfig(ctx context.Context) NewPlatformConfig {
    return NewPlatformConfig{
        StockReleaseState: config.GetBool(ctx, "salesChannels.newPlatform.stockReleaseState"),
        ExternalWebhookToken: config.GetString(ctx, "salesChannels.newPlatform.externalWebhookToken"),
        APIBaseURL: config.GetString(ctx, "salesChannels.newPlatform.apiBaseURL"),
    }
}
```

#### 5. Create Service Implementation

**`service.go`**
```go
package new_platform

import (
    "context"
    "github.com/omniful/sales-channel-service/internal/domain/interfaces"
    "github.com/omniful/sales-channel-service/internal/enums"
    "github.com/omniful/sales-channel-service/internal/sales_channels/new_platform/configurations"
    "github.com/omniful/sales-channel-service/models"
    commonError "github.com/omniful/go_commons/error"
)

type Service struct {
    // Dependencies
    repository interfaces.Repository
    httpClient *http.Client
}

func NewService(repository interfaces.Repository) *Service {
    return &Service{
        repository: repository,
        httpClient: &http.Client{Timeout: 30 * time.Second},
    }
}

// Implement interfaces.SalesChannel
func (s *Service) GetSalesChannelID() int {
    return enums.NewPlatformSalesChannelID
}

func (s *Service) GetSalesChannelTag() string {
    return enums.NewPlatformSalesChannelTag
}

func (s *Service) FetchOrders(ctx context.Context, eventAttributes models.FetchOrderEventAttributes) commonError.CustomError {
    // Get authentication
    auth, cusErr := s.getAuthentication(ctx, eventAttributes.SellerSalesChannelID)
    if cusErr.Exists() {
        return cusErr
    }
    
    // Fetch orders from platform
    orders, cusErr := s.fetchOrdersFromPlatform(ctx, auth, eventAttributes)
    if cusErr.Exists() {
        return cusErr
    }
    
    // Transform to Omniful format
    omnifulOrders, cusErr := s.transformOrdersToOmnifulFormat(ctx, orders)
    if cusErr.Exists() {
        return cusErr
    }
    
    // Send to OMS Service
    cusErr = pushToSQS.SendMessageToOrderServiceOrderQueue(ctx, omnifulOrders)
    return cusErr
}

func (s *Service) FetchProducts(ctx context.Context, eventAttributes models.FetchProductEventAttributes) commonError.CustomError {
    // Implementation for product fetching
    return commonError.CustomError{}
}

func (s *Service) UpdateInventory(ctx context.Context, eventAttributes models.UpdateInventoryEventAttributes) commonError.CustomError {
    // Implementation for inventory updates
    return commonError.CustomError{}
}

func (s *Service) ProcessWebhook(ctx *gin.Context, body []byte) commonError.CustomError {
    // Implementation for webhook processing
    return commonError.CustomError{}
}
```

#### 6. Create Webhook Controller

**`webhook_controller.go`**
```go
package new_platform

import (
    "github.com/gin-gonic/gin"
    "net/http"
)

type Controller struct {
    service *Service
}

func NewController(service *Service) *Controller {
    return &Controller{service: service}
}

func (c *Controller) PublicWebhookHandler(ctx *gin.Context) {
    // Read request body
    body, err := io.ReadAll(ctx.Request.Body)
    if err != nil {
        ctx.JSON(http.StatusBadRequest, gin.H{"error": "Failed to read body"})
        return
    }
    
    // Verify webhook signature
    signature := ctx.GetHeader("X-NewPlatform-Signature")
    if !c.verifyWebhookSignature(body, signature) {
        ctx.JSON(http.StatusUnauthorized, gin.H{"error": "Invalid signature"})
        return
    }
    
    // Process webhook
    cusErr := c.service.ProcessWebhook(ctx, body)
    if cusErr.Exists() {
        ctx.JSON(http.StatusInternalServerError, gin.H{"error": "Failed to process webhook"})
        return
    }
    
    ctx.JSON(http.StatusOK, gin.H{"status": "success"})
}

func (c *Controller) verifyWebhookSignature(body []byte, signature string) bool {
    // Implement signature verification logic
    return true
}
```

#### 7. Add to Factory

**Update `sales_channel_factory.go`**
```go
// Add import
import (
    "github.com/omniful/sales-channel-service/internal/sales_channels/new_platform"
)

// Add to constructor map
var salesChannelConstructorMap = map[int]salesChannelConstructor{
    // ... existing mappings
    enums.NewPlatformSalesChannelID: initNewPlatformService,
}

var salesChannelConstructorMapByTag = map[string]salesChannelConstructor{
    // ... existing mappings
    enums.NewPlatformSalesChannelTag: initNewPlatformService,
}

// Add constructor function
func initNewPlatformService(ctx context.Context) (interfaces.SalesChannel, error) {
    repository := postgres.NewRepository(postgres.GetCluster().DbCluster)
    service := new_platform.NewService(repository)
    return service, nil
}
```

#### 8. Add Routes

**Update `router/gateway_routes.go`**
```go
// Add import
import (
    newPlatform "github.com/omniful/sales-channel-service/internal/sales_channels/new_platform"
)

// Add webhook route
webhookGroup.POST("/new_platform", newPlatformCtrl.PublicWebhookHandler)
```

#### 9. Create Wire Configuration

**`wire.go`**
```go
//go:build wireinject
// +build wireinject

package new_platform

import (
    "github.com/google/wire"
)

func InitializeService(ctx context.Context) (*Service, error) {
    wire.Build(
        NewService,
        postgres.NewRepository,
        wire.Struct(new(Service), "*"),
    )
    return &Service{}, nil
}
```

#### 10. Generate Wire Code
```bash
cd sales-channel-service/internal/sales_channels/new_platform
go generate
```

---

## Background Workers

### Worker Types

The service includes several types of background workers:

#### 1. Order Processing Workers
- **FetchOrdersFifo**: Processes order sync requests
- **WebhookOrderProcessor**: Processes order webhooks
- **OrderStatusUpdater**: Updates order statuses

#### 2. Product Processing Workers
- **FetchProductFIFO**: Processes product sync requests
- **ProductWebhookProcessor**: Processes product webhooks

#### 3. Inventory Processing Workers
- **SCSUpdateProductInventoryFIFO**: Processes inventory updates
- **InventorySyncWorker**: Handles inventory synchronization

### Worker Configuration

```go
// From internal/workers/workers.go
func Run(ctx context.Context, server *http.Server, serverConfig configs.ServerConfig) {
    // Initialize worker groups
    workerGroups := map[string]workerGroup{
        "orders": {
            handlers: []workerHandler{
                &FetchOrdersFifo{},
                &WebhookOrderProcessor{},
            },
        },
        "products": {
            handlers: []workerHandler{
                &FetchProductFIFO{},
                &ProductWebhookProcessor{},
            },
        },
        "inventory": {
            handlers: []workerHandler{
                &SCSUpdateProductInventoryFIFO{},
                &InventorySyncWorker{},
            },
        },
    }
    
    // Start workers based on configuration
    for groupName, group := range workerGroups {
        if shouldStartGroup(groupName, serverConfig) {
            for _, handler := range group.handlers {
                go startWorker(ctx, handler)
            }
        }
    }
}
```

---

## Error Handling

### Custom Error Types

The service uses custom error types for different scenarios:

```go
// From pkg/errors/scs_error.go
const (
    BadRequest              = "BAD_REQUEST"
    AuthenticationError     = "AUTHENTICATION_ERROR"
    AuthorizationError      = "AUTHORIZATION_ERROR"
    InvalidCredentials      = "INVALID_CREDENTIALS"
    RateLimitExceeded       = "RATE_LIMIT_EXCEEDED"
    WebhookSignatureError   = "WEBHOOK_SIGNATURE_ERROR"
    SyncInProgress          = "SYNC_IN_PROGRESS"
    HubMappingNotFound      = "HUB_MAPPING_NOT_FOUND"
    OAuthError              = "OAUTH_ERROR"
    InvalidPayload          = "INVALID_PAYLOAD"
    ParseIntError           = "PARSE_INT_ERROR"
    NoRowsAffectedError     = "NO_ROWS_AFFECTED"
)

func NewErrorResponse(ctx *gin.Context, cusErr commonError.CustomError) {
    ctx.JSON(http.StatusBadRequest, gin.H{
        "status":  "error",
        "message": cusErr.Error(),
        "code":    cusErr.ErrorCode(),
    })
}
```

### Error Handling Patterns

#### 1. API Error Handling
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

#### 2. Retry Logic
```go
func (s *Service) makeAPICallWithRetry(ctx context.Context, url string, method string, body interface{}, maxRetries int) (*http.Response, error) {
    for attempt := 0; attempt < maxRetries; attempt++ {
        response, err := s.makeAPICall(ctx, url, method, body)
        if err == nil {
            return response, nil
        }
        
        // Check if error is retryable
        if !isRetryableError(err) {
            return nil, err
        }
        
        // Exponential backoff
        backoff := time.Duration(attempt+1) * time.Second
        time.Sleep(backoff)
    }
    
    return nil, fmt.Errorf("max retries exceeded")
}
```

---

## Configuration

### Environment Configuration

```yaml
# config.yaml
server:
  port: ":8081"
  readTimeout: 35
  writeTimeout: 30
  idleTimeout: 70

postgresql:
  master:
    host: "localhost"
    port: "5432"
    username: "omniful"
    password: "${DB_PASSWORD}"
    database: "sales_channel_service"

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
      fetchOrdersFifo: "sales-channel-fetch-orders-fifo"
      fetchProductsFifo: "sales-channel-fetch-products-fifo"
      inventoryUpdatesFifo: "sales-channel-inventory-updates-fifo"
      webhookQueue: "sales-channel-webhook-queue"

salesChannels:
  shopify:
    webhookSecret: "${SHOPIFY_WEBHOOK_SECRET}"
    apiVersion: "2025-01"
  wooCommerce:
    webhookSecret: "${WOOCOMMERCE_WEBHOOK_SECRET}"
  salla:
    webhookToken: "${SALLA_WEBHOOK_TOKEN}"
  newPlatform:
    apiBaseURL: "https://api.newplatform.com"
    webhookSecret: "${NEW_PLATFORM_WEBHOOK_SECRET}"

sync:
  freezeTime:
    products: "5m"
    orders: "2m"
    inventory: "1m"
    returns: "10m"
  
  batchSize:
    orders: 50
    products: 100
    inventory: 200

monitoring:
  newRelic:
    enabled: true
    appName: "sales-channel-service"
    licenseKey: "${NEW_RELIC_LICENSE_KEY}"
  
  prometheus:
    enabled: true
    port: ":9090"
```

### Feature Flags

```go
// From configs/environment.go
func IsSyncInventoryV2Active(ctx context.Context) bool {
    return config.GetBool(ctx, "features.syncInventoryV2.enabled")
}

func IsNewPlatformIntegrationActive(ctx context.Context) bool {
    return config.GetBool(ctx, "features.newPlatform.enabled")
}

func GetMaxRetryAttempts(ctx context.Context) int {
    return config.GetInt(ctx, "sync.maxRetryAttempts")
}
```

---

## Monitoring & Observability

### New Relic Integration

```go
// From main.go
import (
    "github.com/newrelic/go-agent/v3/newrelic"
    "github.com/newrelic/go-agent/v3/integrations/nrgin"
)

func main() {
    // Initialize New Relic
    app, err := newrelic.NewApplication(
        newrelic.ConfigAppName("sales-channel-service"),
        newrelic.ConfigLicense(config.GetString(ctx, "monitoring.newRelic.licenseKey")),
    )
    
    // Add New Relic middleware
    server.Engine.Use(nrgin.Middleware(app))
}
```

### Custom Metrics

```go
// From internal/monitoring/metrics.go
var (
    syncRequestsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "sales_channel_sync_requests_total",
            Help: "Total number of sync requests",
        },
        []string{"platform", "entity_type", "status"},
    )
    
    syncDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name: "sales_channel_sync_duration_seconds",
            Help: "Duration of sync operations",
        },
        []string{"platform", "entity_type"},
    )
    
    webhookProcessingDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name: "sales_channel_webhook_processing_duration_seconds",
            Help: "Duration of webhook processing",
        },
        []string{"platform", "event_type"},
    )
)

func RecordSyncMetrics(platform string, entityType string, status string, duration time.Duration) {
    syncRequestsTotal.WithLabelValues(platform, entityType, status).Inc()
    syncDuration.WithLabelValues(platform, entityType).Observe(duration.Seconds())
}
```

### Health Checks

```go
// From internal/monitoring/health.go
func HealthCheckHandler() gin.HandlerFunc {
    return func(c *gin.Context) {
        health := map[string]interface{}{
            "status":    "healthy",
            "timestamp": time.Now().UTC().Format(time.RFC3339),
            "version":   "1.0.0",
            "services": map[string]string{
                "database": "healthy",
                "redis":    "healthy",
                "sqs":      "healthy",
            },
        }
        
        c.JSON(http.StatusOK, health)
    }
}
```

### Structured Logging

```go
// From utils/logging.go
func LogSyncEvent(ctx context.Context, platform string, entityType string, eventType string, details map[string]interface{}) {
    log.WithContext(ctx).WithFields(log.Fields{
        "platform":    platform,
        "entity_type": entityType,
        "event_type":  eventType,
        "details":     details,
    }).Info("Sync event processed")
}

func LogWebhookEvent(ctx context.Context, platform string, eventType string, payload map[string]interface{}) {
    log.WithContext(ctx).WithFields(log.Fields{
        "platform":   platform,
        "event_type": eventType,
        "payload":    payload,
    }).Info("Webhook event received")
}
```

---

## Integration with Onboarding Guide

This Sales Channel Service documentation is referenced in the main onboarding guide at:
- [Core Services Deep Dive - Sales Channel Service](./OMNIFUL_ONBOARDING_GUIDE.md#2--sales-channel-service-sales-channel-service)

### Key Integration Points

1. **Order Processing**: Complete end-to-end order sync flow from external platforms to OMS
2. **Platform Integration**: Factory pattern for managing multiple platform integrations
3. **Authentication**: Platform-specific authentication mechanisms (OAuth, API Keys, Webhook Tokens)
4. **Webhook Processing**: Real-time event handling from external platforms
5. **Background Processing**: Asynchronous data processing via SQS workers
6. **Rate Limiting**: Prevents API abuse and ensures fair usage
7. **Error Handling**: Comprehensive error handling and retry mechanisms

### Related Documentation
- [API Gateway Documentation](./API_GATEWAY_DOCUMENTATION.md)
- [Technical Reference Guide](./TECHNICAL_REFERENCE.md)
- [Sales Channel Service Overview](./sales-channel-service/docs/Overview.md)

---

## Implementation Resources

### For Developers Building New Integrations

- **[Sales Channel Implementation Guide](./Sales_Channel_Implementation_Guide.md)** - Complete 14-phase step-by-step guide for implementing new platform integrations
- **[Technical Reference Guide](./Technical_Reference_Guide.md)** - Common patterns, file structures, and best practices including Wire dependency injection and SQS patterns

### Implementation Phases Overview

The implementation process follows these key phases:

1. **Phase 1-3**: Channel registration, package scaffolding, and routing setup
2. **Phase 4-5**: Configuration management and database transaction patterns
3. **Phase 6-7**: SQS event processing and authentication management
4. **Phase 8-9**: Webhook handling and UI adapters
5. **Phase 10-11**: Status mapping and POS/location management
6. **Phase 12-14**: End-to-end testing and deployment

### Quick Start for New Integrations

1. **Register the channel** in enums and factory
2. **Scaffold the package** structure following the established patterns
3. **Implement core services** (integration, authenticator, webhooks)
4. **Set up routing** and controllers
5. **Configure SQS events** and worker dispatch
6. **Test the integration** using the comprehensive testing checklist

For detailed implementation steps, refer to the [Sales Channel Implementation Guide](./Sales_Channel_Implementation_Guide.md).

---
