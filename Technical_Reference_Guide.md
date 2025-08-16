# Omniful Technical Reference Guide

## Table of Contents
1. [File Structure Patterns](#file-structure-patterns)
2. [Common File Types](#common-file-types)
3. [API Endpoint Patterns](#api-endpoint-patterns)
4. [Database Models](#database-models)
5. [Configuration Management](#configuration-management)
6. [Error Handling](#error-handling)
7. [Testing Patterns](#testing-patterns)
8. [Deployment & Infrastructure](#deployment--infrastructure)

---

## File Structure Patterns

### Sales Channel Service Structure

Every sales channel follows this consistent structure:

```
sales-channel-service/internal/sales_channels/{platform_name}/
├── configurations/                    # Platform-specific configurations
│   ├── web_interact_config.go         # API endpoint URL generators
│   ├── default_configurations.go      # Platform settings and defaults
│   ├── oauth_config.go               # OAuth authentication (if applicable)
│   ├── default_order_status.go       # Status mapping between platform and Omniful
│   └── default_sync_events.go        # Webhook event configurations
├── models/                           # Data structures
│   ├── requests/                     # API request models
│   ├── responses/                    # API response models
│   └── webhook/                      # Webhook payload models
├── service.go                        # Main service implementation
├── controller.go                     # HTTP request handlers
├── webhook_controller.go             # Webhook endpoint handlers
├── web_interact_service.go           # External API interactions
├── update_inventory.go               # Inventory synchronization logic
├── update_order.go                   # Order update logic
├── products.go                       # Product synchronization logic
├── order.go                          # Order processing logic
├── integration.go                    # Platform integration setup
├── provider.go                       # Dependency injection setup
├── wire.go                           # Wire dependency injection
└── wire_gen.go                       # Generated wire code
```

### Common File Patterns Across Services

#### 1. **Main Entry Points**
```go
// main.go pattern
package main

import (
    "context"
    "flag"
    "time"
    "github.com/omniful/go_commons/config"
    "github.com/omniful/go_commons/http"
    "github.com/omniful/go_commons/log"
    "github.com/omniful/go_commons/shutdown"
)

const (
    modeWorker = "worker"
    modeHttp   = "http"
)

func main() {
    // Initialize configuration
    err := config.Init(time.Second * 10)
    if err != nil {
        log.Panicf("Error while initialising config, err: %v", err)
        panic(err)
    }

    ctx, err := config.TODOContext()
    if err != nil {
        log.Panicf("Error while getting context from config, err: %v", err)
        panic(err)
    }

    // Initialize application
    appinit.Initialize(ctx)

    // Parse command line flags
    var mode string
    flag.StringVar(&mode, "mode", modeHttp, "Service mode")
    flag.Parse()

    // Start service based on mode
    switch strings.ToLower(mode) {
    case modeHttp:
        runHttpServer(ctx)
    case modeWorker:
        runWorker(ctx)
    default:
        runHttpServer(ctx)
    }
}
```

#### 2. **Router Patterns**
```go
// router/router.go pattern
package router

import (
    "context"
    "github.com/gin-gonic/gin"
    "github.com/omniful/go_commons/http"
)

func Initialize(ctx context.Context, server *http.Server) error {
    // Setup middleware
    server.Use(middlewares.AuthenticationMiddleware())
    server.Use(middlewares.RateLimitingMiddleware())
    server.Use(middlewares.CORSMiddleware())

    // Setup routes
    setupRoutes(server)
    
    return nil
}

func setupRoutes(server *http.Server) {
    // API v1 routes
    v1 := server.Group("/api/v1")
    {
        // Sales channel routes
        salesChannelRoutes := v1.Group("/sales_channels")
        {
            salesChannelRoutes.GET("/sellers/:seller_id/sales_channels/:ssc_id/sync_products", 
                controllers.SyncEntities)
            salesChannelRoutes.POST("/webhooks/:platform", 
                controllers.HandleWebhook)
        }

        // Order routes
        orderRoutes := v1.Group("/orders")
        {
            orderRoutes.GET("/:order_id", controllers.GetOrder)
            orderRoutes.PUT("/:order_id", controllers.UpdateOrder)
        }
    }
}
```

#### 3. **Controller Patterns**
```go
// controllers/controller.go pattern
package controllers

import (
    "github.com/gin-gonic/gin"
    "github.com/omniful/go_commons/log"
    commonError "github.com/omniful/go_commons/error"
)

type Controller struct {
    service interfaces.Service
}

func (c *Controller) SyncEntities(ctx *gin.Context) {
    // Extract parameters
    sellerID := ctx.Param("seller_id")
    sscID := ctx.Param("ssc_id")
    syncEntity := ctx.Query("sync_entity")
    
    // Validate parameters
    if err := validateSyncParams(sellerID, sscID, syncEntity); err != nil {
        ctx.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    
    // Call service layer
    result, err := c.service.SyncEntities(ctx, sscID, syncEntity)
    if err != nil {
        log.Errorf("Failed to sync entities: %v", err)
        ctx.JSON(http.StatusInternalServerError, gin.H{"error": "Sync failed"})
        return
    }
    
    ctx.JSON(http.StatusOK, result)
}
```

---

## Common File Types

### 1. **Configuration Files**

#### `web_interact_config.go`
```go
package configurations

import (
    "context"
    "fmt"
    "net/url"
    "github.com/omniful/go_commons/config"
    "github.com/omniful/sales-channel-service/constants"
)

// Base URL configuration
func getPlatformHost(ctx context.Context) string {
    return config.GetString(ctx, "salesChannels.platform.host")
}

// API endpoint generators
func GetFetchProductsURL(ctx context.Context, storeIdentifier string, queryParams url.Values) string {
    return getPlatformHost(ctx) + storeIdentifier + constants.PlatformFetchProductsURL + "?" + queryParams.Encode()
}

func GetFetchOrdersURL(ctx context.Context, storeIdentifier string, queryParams url.Values) string {
    return getPlatformHost(ctx) + storeIdentifier + constants.PlatformFetchOrdersURL + "?" + queryParams.Encode()
}

func GetUpdateInventoryURL(ctx context.Context, storeIdentifier string, productID string, queryParams url.Values) string {
    return getPlatformHost(ctx) + storeIdentifier + fmt.Sprintf(constants.PlatformUpdateInventoryURL, productID) + "?" + queryParams.Encode()
}

func GetCreateWebhookURL(ctx context.Context, storeIdentifier string, queryParams url.Values) string {
    return getPlatformHost(ctx) + storeIdentifier + constants.PlatformCreateWebhookURL + "?" + queryParams.Encode()
}
```

#### `default_configurations.go`
```go
package configurations

import (
    "context"
    "github.com/omniful/go_commons/config"
)

type PlatformConfig struct {
    StockReleaseState       bool
    ExternalWebhookToken    string
    APIKey                  string
    APISecret              string
    WebhookSecret          string
    MaxRetries             int
    TimeoutSeconds         int
}

func GetPlatformConfig(ctx context.Context) PlatformConfig {
    return PlatformConfig{
        StockReleaseState:    config.GetBool(ctx, "salesChannels.platform.stockReleaseState"),
        ExternalWebhookToken: config.GetString(ctx, "salesChannels.platform.externalWebhookToken"),
        APIKey:               config.GetString(ctx, "salesChannels.platform.apiKey"),
        APISecret:            config.GetString(ctx, "salesChannels.platform.apiSecret"),
        WebhookSecret:        config.GetString(ctx, "salesChannels.platform.webhookSecret"),
        MaxRetries:           config.GetInt(ctx, "salesChannels.platform.maxRetries"),
        TimeoutSeconds:       config.GetInt(ctx, "salesChannels.platform.timeoutSeconds"),
    }
}
```

### 2. **Service Implementation Files**

#### `service.go`
```go
package platform_name

import (
    "context"
    "github.com/omniful/sales-channel-service/internal/domain/interfaces"
    "github.com/omniful/sales-channel-service/internal/sales_channels/platform_name/configurations"
    "github.com/omniful/sales-channel-service/internal/sales_channels/platform_name/models"
    commonError "github.com/omniful/go_commons/error"
)

type Service struct {
    repository    interfaces.Repository
    httpClient    interfaces.HTTPClient
    config        configurations.PlatformConfig
}

// Implement interfaces.SalesChannel
func (s *Service) GetSalesChannelID() int {
    return enums.PlatformSalesChannelID
}

func (s *Service) GetSalesChannelTag() string {
    return enums.PlatformSalesChannelTag
}

// Core functionality
func (s *Service) FetchProducts(ctx context.Context, eventAttributes models.FetchProductEventAttributes) commonError.CustomError {
    // Implementation for fetching products from platform
    url := configurations.GetFetchProductsURL(ctx, eventAttributes.StoreIdentifier, eventAttributes.QueryParams)
    
    response, err := s.httpClient.Get(ctx, url, s.getAuthHeaders())
    if err != nil {
        return commonError.NewCustomError(
            scsError.InternalServerError,
            fmt.Sprintf("Failed to fetch products: %v", err),
            commonError.WithShouldNotify(true),
        )
    }
    
    // Parse response and process products
    return s.processProductsResponse(ctx, response, eventAttributes)
}

func (s *Service) FetchOrders(ctx context.Context, eventAttributes models.FetchOrderEventAttributes) commonError.CustomError {
    // Implementation for fetching orders from platform
    return commonError.CustomError{}
}

func (s *Service) UpdateInventory(ctx context.Context, eventAttributes models.UpdateInventoryEventAttributes) commonError.CustomError {
    // Implementation for updating inventory on platform
    return commonError.CustomError{}
}

// Helper methods
func (s *Service) getAuthHeaders() map[string]string {
    return map[string]string{
        "Authorization": "Bearer " + s.config.APIKey,
        "Content-Type":  "application/json",
    }
}
```

#### `web_interact_service.go`
```go
package platform_name

import (
    "context"
    "encoding/json"
    "fmt"
    "net/http"
    "time"
    "github.com/omniful/sales-channel-service/internal/sales_channels/platform_name/configurations"
    "github.com/omniful/sales-channel-service/internal/sales_channels/platform_name/models"
)

type WebInteractService struct {
    config configurations.PlatformConfig
}

func (s *WebInteractService) makeAPICall(ctx context.Context, url string, method string, body interface{}) (*http.Response, error) {
    client := &http.Client{
        Timeout: time.Duration(s.config.TimeoutSeconds) * time.Second,
    }
    
    var req *http.Request
    var err error
    
    if body != nil {
        jsonBody, err := json.Marshal(body)
        if err != nil {
            return nil, fmt.Errorf("failed to marshal request body: %v", err)
        }
        req, err = http.NewRequestWithContext(ctx, method, url, bytes.NewBuffer(jsonBody))
    } else {
        req, err = http.NewRequestWithContext(ctx, method, url, nil)
    }
    
    if err != nil {
        return nil, fmt.Errorf("failed to create request: %v", err)
    }
    
    // Add headers
    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("Authorization", "Bearer "+s.config.APIKey)
    
    // Add retry logic
    return s.makeRequestWithRetry(client, req)
}

func (s *WebInteractService) makeRequestWithRetry(client *http.Client, req *http.Request) (*http.Response, error) {
    var lastErr error
    
    for attempt := 0; attempt <= s.config.MaxRetries; attempt++ {
        response, err := client.Do(req)
        if err == nil && response.StatusCode < 500 {
            return response, nil
        }
        
        lastErr = err
        if attempt < s.config.MaxRetries {
            time.Sleep(time.Duration(attempt+1) * time.Second)
        }
    }
    
    return nil, fmt.Errorf("request failed after %d attempts: %v", s.config.MaxRetries+1, lastErr)
}
```

### 3. **Webhook Processing Files**

#### `webhook_controller.go`
```go
package platform_name

import (
    "crypto/hmac"
    "crypto/sha256"
    "encoding/hex"
    "io"
    "net/http"
    "github.com/gin-gonic/gin"
    "github.com/omniful/sales-channel-service/internal/sales_channels/platform_name/configurations"
    "github.com/omniful/sales-channel-service/internal/sales_channels/platform_name/models"
    "github.com/omniful/sales-channel-service/utils/scs_sqs_events"
    "github.com/omniful/sales-channel-service/external_service/controllers/push_to_sqs"
)

type WebhookController struct {
    config configurations.PlatformConfig
}

func (c *WebhookController) PublicWebhookHandler(ctx *gin.Context) {
    // Read request body
    body, err := io.ReadAll(ctx.Request.Body)
    if err != nil {
        ctx.JSON(http.StatusBadRequest, gin.H{"error": "Failed to read request body"})
        return
    }
    
    // Verify webhook signature
    signature := ctx.GetHeader("X-Platform-Signature")
    if !c.verifyWebhookSignature(body, signature) {
        ctx.JSON(http.StatusUnauthorized, gin.H{"error": "Invalid webhook signature"})
        return
    }
    
    // Parse webhook payload
    var webhookPayload models.WebhookPayload
    if err := json.Unmarshal(body, &webhookPayload); err != nil {
        ctx.JSON(http.StatusBadRequest, gin.H{"error": "Invalid webhook payload"})
        return
    }
    
    // Create SQS event
    event := sqs_events.CreateWebhookEvent(webhookPayload)
    
    // Send to SQS queue
    err = pushToSqs.SendMessageToWebhookQueue(ctx, event)
    if err != nil {
        ctx.JSON(http.StatusInternalServerError, gin.H{"error": "Failed to process webhook"})
        return
    }
    
    ctx.JSON(http.StatusOK, gin.H{"status": "success"})
}

func (c *WebhookController) verifyWebhookSignature(body []byte, signature string) bool {
    expectedSignature := hmac.New(sha256.New, []byte(c.config.WebhookSecret))
    expectedSignature.Write(body)
    expectedHex := hex.EncodeToString(expectedSignature.Sum(nil))
    
    return hmac.Equal([]byte(expectedHex), []byte(signature))
}
```

### 4. **Model Files**

#### `models/requests/`
```go
package requests

type FetchProductsRequest struct {
    StoreIdentifier string            `json:"store_identifier"`
    QueryParams     map[string]string `json:"query_params"`
    Page            int               `json:"page"`
    Limit           int               `json:"limit"`
}

type UpdateInventoryRequest struct {
    ProductID string `json:"product_id"`
    SKU       string `json:"sku"`
    Quantity  int    `json:"quantity"`
    Location  string `json:"location"`
}

type CreateWebhookRequest struct {
    Topic   string `json:"topic"`
    Address string `json:"address"`
    Format  string `json:"format"`
}
```

#### `models/responses/`
```go
package responses

type ProductResponse struct {
    ID          string  `json:"id"`
    Title       string  `json:"title"`
    SKU         string  `json:"sku"`
    Price       float64 `json:"price"`
    Inventory   int     `json:"inventory"`
    Status      string  `json:"status"`
    CreatedAt   string  `json:"created_at"`
    UpdatedAt   string  `json:"updated_at"`
}

type OrderResponse struct {
    ID            string           `json:"id"`
    OrderNumber   string           `json:"order_number"`
    Status        string           `json:"status"`
    Total         float64          `json:"total"`
    Currency      string           `json:"currency"`
    Customer      CustomerResponse `json:"customer"`
    Items         []OrderItem      `json:"items"`
    ShippingAddress AddressResponse `json:"shipping_address"`
    CreatedAt     string           `json:"created_at"`
}

type WebhookResponse struct {
    ID      string `json:"id"`
    Topic   string `json:"topic"`
    Address string `json:"address"`
    Status  string `json:"status"`
}
```

---

## API Endpoint Patterns

### 1. **Sync Endpoints**
```go
// Pattern: GET /api/v1/sales_channels/sellers/{seller_id}/sales_channels/{ssc_id}/sync_products
func (c *Controller) SyncEntities(ctx *gin.Context) {
    sellerID := ctx.Param("seller_id")
    sscID := ctx.Param("ssc_id")
    syncEntity := ctx.Query("sync_entity") // products, orders, inventory
    syncFromTime := ctx.Query("sync_from_time")
    syncFromType := ctx.Query("sync_from_type") // day, hour, minute
    
    // Validate parameters
    if err := validateSyncParams(syncEntity, syncFromTime, syncFromType); err != nil {
        ctx.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    
    // Call service
    result, err := c.service.SyncEntities(ctx, sscID, syncEntity, syncFromTime, syncFromType)
    if err != nil {
        ctx.JSON(http.StatusInternalServerError, gin.H{"error": "Sync failed"})
        return
    }
    
    ctx.JSON(http.StatusOK, result)
}
```

### 2. **Webhook Endpoints**
```go
// Pattern: POST /api/v1/sales_channels/webhooks/{platform}
func (c *Controller) HandleWebhook(ctx *gin.Context) {
    platform := ctx.Param("platform")
    
    // Route to platform-specific handler
    switch platform {
    case "shopify":
        c.shopifyController.PublicWebhookHandler(ctx)
    case "woocommerce":
        c.woocommerceController.PublicWebhookHandler(ctx)
    default:
        ctx.JSON(http.StatusNotFound, gin.H{"error": "Platform not supported"})
    }
}
```

### 3. **Order Management Endpoints**
```go
// Pattern: GET /api/v1/orders/{order_id}
func (c *Controller) GetOrder(ctx *gin.Context) {
    orderID := ctx.Param("order_id")
    
    order, err := c.service.GetOrder(ctx, orderID)
    if err != nil {
        ctx.JSON(http.StatusNotFound, gin.H{"error": "Order not found"})
        return
    }
    
    ctx.JSON(http.StatusOK, order)
}

// Pattern: PUT /api/v1/orders/{order_id}
func (c *Controller) UpdateOrder(ctx *gin.Context) {
    orderID := ctx.Param("order_id")
    
    var updateRequest models.UpdateOrderRequest
    if err := ctx.ShouldBindJSON(&updateRequest); err != nil {
        ctx.JSON(http.StatusBadRequest, gin.H{"error": "Invalid request body"})
        return
    }
    
    result, err := c.service.UpdateOrder(ctx, orderID, updateRequest)
    if err != nil {
        ctx.JSON(http.StatusInternalServerError, gin.H{"error": "Update failed"})
        return
    }
    
    ctx.JSON(http.StatusOK, result)
}
```

---

## Database Models

### Common Database Patterns

#### 1. **Sales Channel Models**
```go
type SellerSalesChannels struct {
    ID                    int       `json:"id" db:"id"`
    SellerID              string    `json:"seller_id" db:"seller_id"`
    SalesChannelID        int       `json:"sales_channel_id" db:"sales_channel_id"`
    Name                  string    `json:"name" db:"name"`
    ExternalMerchantID    string    `json:"external_merchant_id" db:"external_merchant_id"`
    ExternalWebhookSecret string    `json:"external_webhook_secret" db:"external_webhook_secret"`
    Configs               string    `json:"configs" db:"configs"`
    CreatedAt             time.Time `json:"created_at" db:"created_at"`
    UpdatedAt             time.Time `json:"updated_at" db:"updated_at"`
}

type SellerSalesChannelAuthentication struct {
    ID                    int       `json:"id" db:"id"`
    SellerSalesChannelID  int       `json:"seller_sales_channel_id" db:"seller_sales_channel_id"`
    AppID                 string    `json:"app_id" db:"app_id"`
    AccessToken           string    `json:"access_token" db:"access_token"`
    RefreshToken          string    `json:"refresh_token" db:"refresh_token"`
    ExpiresAt             time.Time `json:"expires_at" db:"expires_at"`
    CreatedAt             time.Time `json:"created_at" db:"created_at"`
    UpdatedAt             time.Time `json:"updated_at" db:"updated_at"`
}
```

#### 2. **Order Models**
```go
type Orders struct {
    ID                    int       `json:"id" db:"id"`
    SellerSalesChannelID  int       `json:"seller_sales_channel_id" db:"seller_sales_channel_id"`
    ExternalOrderID       string    `json:"external_order_id" db:"external_order_id"`
    OrderNumber           string    `json:"order_number" db:"order_number"`
    Status                string    `json:"status" db:"status"`
    Total                 float64   `json:"total" db:"total"`
    Currency              string    `json:"currency" db:"currency"`
    CustomerData          string    `json:"customer_data" db:"customer_data"`
    ShippingAddress       string    `json:"shipping_address" db:"shipping_address"`
    BillingAddress        string    `json:"billing_address" db:"billing_address"`
    CreatedAt             time.Time `json:"created_at" db:"created_at"`
    UpdatedAt             time.Time `json:"updated_at" db:"updated_at"`
}
```

#### 3. **Product Models**
```go
type Products struct {
    ID                    int       `json:"id" db:"id"`
    SellerSalesChannelID  int       `json:"seller_sales_channel_id" db:"seller_sales_channel_id"`
    ExternalProductID     string    `json:"external_product_id" db:"external_product_id"`
    SKU                   string    `json:"sku" db:"sku"`
    Title                 string    `json:"title" db:"title"`
    Description           string    `json:"description" db:"description"`
    Price                 float64   `json:"price" db:"price"`
    Inventory             int       `json:"inventory" db:"inventory"`
    Status                string    `json:"status" db:"status"`
    ProductData           string    `json:"product_data" db:"product_data"`
    CreatedAt             time.Time `json:"created_at" db:"created_at"`
    UpdatedAt             time.Time `json:"updated_at" db:"updated_at"`
}
```

---

## Configuration Management

### Environment Configuration

#### 1. **Service Configuration**
```yaml
# config.yaml
server:
  port: ":8080"
  readTimeout: 30
  writeTimeout: 30
  idleTimeout: 60

database:
  host: "localhost"
  port: 5432
  name: "omniful"
  user: "postgres"
  password: "password"
  sslmode: "disable"

redis:
  host: "localhost"
  port: 6379
  password: ""
  db: 0

aws:
  region: "us-east-1"
  sqs:
    queueUrl: "https://sqs.us-east-1.amazonaws.com/123456789012/omniful-queue"
  sns:
    topicArn: "arn:aws:sns:us-east-1:123456789012:omniful-topic"

salesChannels:
  shopify:
    host: "https://api.shopify.com"
    stockReleaseState: true
    maxRetries: 3
    timeoutSeconds: 30
  woocommerce:
    host: "https://api.woocommerce.com"
    stockReleaseState: false
    maxRetries: 3
    timeoutSeconds: 30
```

#### 2. **Feature Flags**
```go
// configs/environment.go
func IsNewFeatureActive(ctx context.Context) bool {
    return config.GetBool(ctx, "features.newFeature")
}

func IsSyncInventoryV2Active(ctx context.Context) bool {
    return config.GetBool(ctx, "features.syncInventoryV2")
}
```

---

## Error Handling

### Custom Error Types

#### 1. **Error Definition**
```go
// internal/errors/errors.go
type ErrorCode int

const (
    BadRequest ErrorCode = iota + 1
    Unauthorized
    Forbidden
    NotFound
    InternalServerError
    ServiceUnavailable
)

type CustomError struct {
    Code           ErrorCode
    Message        string
    ShouldNotify   bool
    Retryable      bool
    OriginalError  error
}

func NewCustomError(code ErrorCode, message string, opts ...ErrorOption) CustomError {
    err := CustomError{
        Code:         code,
        Message:      message,
        ShouldNotify: false,
        Retryable:    false,
    }
    
    for _, opt := range opts {
        opt(&err)
    }
    
    return err
}

type ErrorOption func(*CustomError)

func WithShouldNotify(shouldNotify bool) ErrorOption {
    return func(err *CustomError) {
        err.ShouldNotify = shouldNotify
    }
}

func WithRetryable(retryable bool) ErrorOption {
    return func(err *CustomError) {
        err.Retryable = retryable
    }
}
```

#### 2. **Error Usage**
```go
func (s *Service) fetchProducts(ctx context.Context, url string) commonError.CustomError {
    response, err := s.httpClient.Get(ctx, url)
    if err != nil {
        return commonError.NewCustomError(
            scsError.InternalServerError,
            fmt.Sprintf("Failed to fetch products: %v", err),
            commonError.WithShouldNotify(true),
            commonError.WithRetryable(true),
        )
    }
    
    if response.StatusCode >= 400 {
        return commonError.NewCustomError(
            scsError.BadRequest,
            fmt.Sprintf("API returned status %d", response.StatusCode),
            commonError.WithShouldNotify(false),
        )
    }
    
    return commonError.CustomError{}
}
```

---

## Testing Patterns

### 1. **Unit Tests**
```go
// service_test.go
func TestService_FetchProducts(t *testing.T) {
    // Setup
    mockRepo := &mocks.MockRepository{}
    mockHTTPClient := &mocks.MockHTTPClient{}
    service := &Service{
        repository: mockRepo,
        httpClient: mockHTTPClient,
    }
    
    // Test data
    ctx := context.Background()
    eventAttributes := models.FetchProductEventAttributes{
        StoreIdentifier: "test-store",
        QueryParams:     map[string]string{"limit": "10"},
    }
    
    // Mock expectations
    mockHTTPClient.On("Get", ctx, mock.AnythingOfType("string"), mock.AnythingOfType("map[string]string")).
        Return(&http.Response{StatusCode: 200}, nil)
    
    // Execute
    err := service.FetchProducts(ctx, eventAttributes)
    
    // Assert
    assert.NoError(t, err)
    mockHTTPClient.AssertExpectations(t)
}
```

### 2. **Integration Tests**
```go
// integration_test.go
func TestIntegration_OrderSync(t *testing.T) {
    // Setup test environment
    testDB := setupTestDatabase(t)
    testRedis := setupTestRedis(t)
    
    // Create test data
    sellerSalesChannel := createTestSellerSalesChannel(t, testDB)
    
    // Execute integration test
    result := syncOrders(t, sellerSalesChannel.ID)
    
    // Assert results
    assert.NotNil(t, result)
    assert.Len(t, result.Orders, 1)
}
```

### 3. **API Tests**
```go
// api_test.go
func TestAPI_SyncProducts(t *testing.T) {
    // Setup test server
    router := setupTestRouter()
    
    // Create test request
    req := httptest.NewRequest("GET", "/api/v1/sales_channels/sellers/123/sales_channels/456/sync_products?sync_entity=products", nil)
    req.Header.Set("Authorization", "Bearer test-token")
    
    // Execute request
    w := httptest.NewRecorder()
    router.ServeHTTP(w, req)
    
    // Assert response
    assert.Equal(t, http.StatusOK, w.Code)
    
    var response map[string]interface{}
    err := json.Unmarshal(w.Body.Bytes(), &response)
    assert.NoError(t, err)
    assert.Equal(t, "success", response["status"])
}
```

---

## Deployment & Infrastructure

### 1. **Docker Configuration**
```dockerfile
# Dockerfile
FROM golang:1.21-alpine AS builder

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN go build -o main ./cmd/service

FROM alpine:latest
RUN apk --no-cache add ca-certificates
WORKDIR /root/

COPY --from=builder /app/main .
COPY --from=builder /app/configs ./configs

EXPOSE 8080
CMD ["./main"]
```

### 2. **Docker Compose**
```yaml
# docker-compose.yml
version: '3.8'

services:
  api-gateway:
    build: ./api-gateway
    ports:
      - "8080:8080"
    environment:
      - DB_HOST=postgres
      - REDIS_HOST=redis
    depends_on:
      - postgres
      - redis

  sales-channel-service:
    build: ./sales-channel-service
    ports:
      - "8081:8081"
    environment:
      - DB_HOST=postgres
      - REDIS_HOST=redis
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgres:13
    environment:
      POSTGRES_DB: omniful
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:6-alpine
    ports:
      - "6379:6379"

volumes:
  postgres_data:
```

### 3. **Kubernetes Deployment**
```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sales-channel-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: sales-channel-service
  template:
    metadata:
      labels:
        app: sales-channel-service
    spec:
      containers:
      - name: sales-channel-service
        image: omniful/sales-channel-service:latest
        ports:
        - containerPort: 8081
        env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: omniful-config
              key: db_host
        - name: REDIS_HOST
          valueFrom:
            configMapKeyRef:
              name: omniful-config
              key: redis_host
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

---

## Monitoring & Observability

### 1. **Logging Patterns**
```go
// Structured logging
log.WithFields(log.Fields{
    "service": "sales-channel-service",
    "platform": "shopify",
    "seller_id": sellerID,
    "ssc_id": sscID,
    "operation": "sync_products",
}).Info("Starting product sync")

// Error logging with context
log.WithFields(log.Fields{
    "error": err.Error(),
    "url": url,
    "status_code": response.StatusCode,
}).Error("API call failed")
```

### 2. **Metrics Collection**
```go
// Prometheus metrics
var (
    apiRequestsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "api_requests_total",
            Help: "Total number of API requests",
        },
        []string{"method", "endpoint", "status"},
    )
    
    syncDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name: "sync_duration_seconds",
            Help: "Duration of sync operations",
        },
        []string{"platform", "entity_type"},
    )
)
```

### 3. **Health Checks**
```go
// Health check endpoint
func (c *Controller) HealthCheck(ctx *gin.Context) {
    health := map[string]interface{}{
        "status": "healthy",
        "timestamp": time.Now().Unix(),
        "version": "1.0.0",
        "services": map[string]string{
            "database": "healthy",
            "redis": "healthy",
            "aws": "healthy",
        },
    }
    
    ctx.JSON(http.StatusOK, health)
}
```

---

## Development Tools

### 1. **Code Generation**
```bash
# Generate wire code
go generate ./...

# Generate mocks
mockgen -source=interfaces.go -destination=mocks/mocks.go

# Generate API documentation
swag init -g main.go
```

### 2. **Wire Dependency Injection**
```bash
# Install wire
go install github.com/google/wire/cmd/wire@latest

# Generate wire code in specific package
cd internal/sales_channels/acme
wire

# Generate wire code for all packages
find . -name "wire.go" -execdir wire \;
```

### 3. **Sales Channel Integration Commands**
```bash
# Scaffold new sales channel
mkdir -p internal/sales_channels/new_platform/{configurations,models,services,controller}

# Generate wire code for new platform
cd internal/sales_channels/new_platform
wire

# Test integration
go test ./...
go build ./...
```

### 2. **Code Quality**
```bash
# Run linter
golangci-lint run

# Run tests
go test ./...

# Run tests with coverage
go test -cover ./...

# Run benchmarks
go test -bench=. ./...
```

### 3. **Database Migrations**
```bash
# Run migrations
migrate -path db/migrations -database "postgres://user:pass@localhost/dbname?sslmode=disable" up

# Rollback migrations
migrate -path db/migrations -database "postgres://user:pass@localhost/dbname?sslmode=disable" down
```

---

## Wire Dependency Injection Patterns

### 1. **Provider Sets**

**File**: `internal/sales_channels/acme/provider.go`

```go
var ProviderSet = wire.NewSet(
    // Core dependencies
    sales_channel_commons.NewRepository,
    sales_channel_commons.NewService,
    base.NewSalesChannel,

    // Submodule services
    integration.NewService,
    webhooks.NewService,
    authenticator.NewAuthenticator,

    // External services
    oms.NewClient, 
    analytics.NewClient, 
    product.NewClient,
    redisCache.NewRedisCacheClient, 
    redisCache.NewMsgpackSerializer, 
    cache.NewCache,

    // Interface bindings
    wire.Bind(new(domain.SalesChannelRepository), new(*sales_channel_commons.Repository)),
    wire.Bind(new(redisCache.ICache), new(*redisCache.RedisCache)),
    wire.Bind(new(interfaces.Authenticator), new(*authenticator.Authenticator)),
    wire.Bind(new(domain.SalesChannelService), new(*sales_channel_commons.Service)),

    // Main service
    NewService,
)
```

### 2. **Wire Functions**

**File**: `internal/sales_channels/acme/wire.go`

```go
//go:build wireinject
// +build wireinject
package acme

import (
    "context"
    "github.com/google/wire"
    "github.com/omniful/go_commons/db/sql/postgres"
    oredis "github.com/omniful/go_commons/redis"
)

func ServiceWire(db *postgres.DbCluster, ctx context.Context, client *oredis.Client, nameSpace string) (*Service, error) {
    panic(wire.Build(ProviderSet))
}
```

### 3. **Factory Pattern with Wire**

**File**: `internal/sales_channels/sales_channel_factory.go`

```go
var salesChannelConstructorMap = map[int]salesChannelConstructor{
    enums.ShopifySalesChannelID:       initShopifyService,
    enums.WooCommerceV2SalesChannelID: initWooCommerceV2Service,
    enums.AcmeSalesChannelID:          initAcmeService, // NEW
}

func initAcmeService(ctx context.Context) (interfaces.SalesChannel, error) {
    acmeSvc, err := acme.ServiceWire(
        postgres.GetCluster().DbCluster,
        ctx,
        redis.GetClient().Client,
        globalUtils.GetNameSpace(ctx),
    )
    if err != nil {
        log.Errorf("Unable to init Acme service: %v", err)
        return nil, err
    }
    return acmeSvc, nil
}
```

### 4. **Common Service Wiring for Workers**

**File**: `internal/common_service/common_service.go`

```go
type SalesChannelServices struct {
    ShopifyService      domain.ShopifyService
    WooCommerceService  domain.WooCommerceService
    AcmeService         domain.AcmeService // NEW
}

func NewSalesChannelServices(...) *SalesChannelServices {
    // Create Acme service
    acmeSvc, err := acme.ServiceWire(postgres.GetCluster().DbCluster, ctx, redis.GetClient().Client, utils.GetNameSpace(ctx))
    if err != nil { log.Errorf("Acme wire failed: %v", err) }

    return &SalesChannelServices{
        ShopifyService:     shopifySvc,
        WooCommerceService: wooCommerceSvc,
        AcmeService:        acmeSvc, // NEW
    }
}
```

---

## Related Documentation

For comprehensive implementation details and step-by-step guides, refer to:

- **[Sales Channel Implementation Guide](./Sales_Channel_Implementation_Guide.md)** - Complete 14-phase implementation guide for new integrations
- **[Sales Channel Service Documentation](./Sales_Channel_Service_Documentation.md)** - High-level service overview and architecture
- **[API Gateway Documentation](./Api_Gateway_Documentation.md)** - Routing, authentication, and API patterns

---

## SQS Event Processing Patterns

### 1. **Event Model Structure**

**File**: `internal/workers/models/sqs_models/sync_integration_events.go`

```go
type SqsSyncIntegrationEvents struct {
    EventType            string
    SellerSalesChannelID int64
    SalesChannelID       int
    SellerID             string
    MerchantID           string
    CreatedAtMin         string
    CreatedAtMax         string
    // Additional fields as needed
}
```

### 2. **Worker Dispatch Logic**

**File**: `internal/workers/handlers/sqs_integration_events_handler.go`

```go
switch event.EventType {
case constants.CreateWebhook:
    switch event.SalesChannelID {
    case enums.ShopifySalesChannelID:
        return commonService.SalesChannelServices.ShopifyService.CreateWebhook(ctx, event)
    case enums.AcmeSalesChannelID:
        return commonService.SalesChannelServices.AcmeService.CreateWebhook(ctx, event)
    }

case constants.DefaultOrderStatusMapping:
    switch event.SalesChannelID {
    case enums.ShopifySalesChannelID:
        return commonService.SalesChannelServices.ShopifyService.HandleDefaultOrderStatusMappingEvent(ctx, event)
    case enums.AcmeSalesChannelID:
        return commonService.SalesChannelServices.AcmeService.HandleDefaultOrderStatusMappingEvent(ctx, event)
    }
}
```

### 3. **Event Creation Utilities**

**File**: `utils/scs_sqs_events/sqs_events.go`

```go
func CreateWebhookEvent(ctx *gin.Context, ssc *models.SellerSalesChannels) commonError.CustomError {
    event := sqsModels.SqsSyncIntegrationEvents{
        EventType:            constants.CreateWebhook,
        SellerSalesChannelID: ssc.Id,
        SalesChannelID:       ssc.SalesChannelId,
        SellerID:            ssc.SellerId,
        MerchantID:          ssc.ExternalMerchantID,
    }
    
    return pushToSqs.SendMessageToWebhookQueue(ctx, event)
}

func CreateOrderStatusMappingEvent(ctx *gin.Context, ssc *models.SellerSalesChannels) commonError.CustomError {
    event := sqsModels.SqsSyncIntegrationEvents{
        EventType:            constants.DefaultOrderStatusMapping,
        SellerSalesChannelID: ssc.Id,
        SalesChannelID:       ssc.SalesChannelId,
        SellerID:            ssc.SellerId,
        MerchantID:          ssc.ExternalMerchantID,
    }
    
    return pushToSqs.SendMessageToOrderStatusMappingQueue(ctx, event)
}
```

### 4. **Background Worker Processing**

**File**: `internal/workers/handlers/sqs_integration_events_handler.go`

```go
func (e SqsIntegrationEventsHandler) Process(ctx context.Context, sqsMessages *[]sqs.Message) error {
    for _, message := range *sqsMessages {
        var eventAttributes sqsModels.SqsSyncIntegrationEvents
        
        if err := json.Unmarshal([]byte(message.Body), &eventAttributes); err != nil {
            log.Errorf("Failed to unmarshal SQS message: %v", err)
            continue
        }
        
        // Route to appropriate handler based on event type and sales channel
        if err := e.routeEvent(ctx, eventAttributes); err != nil {
            log.Errorf("Failed to process event: %v", err)
            // Consider dead letter queue for failed events
        }
    }
    
    return nil
}
```

---
