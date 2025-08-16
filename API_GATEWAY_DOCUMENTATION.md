# ![[Pasted image 20250816171101.png|23]] Omniful API Gateway Documentation

## Table of Contents
1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Request Flow](#request-flow)
4. [Authentication & Authorization](#authentication--authorization)
5. [Key Components](#key-components)
6. [Order Processing Flow](#order-processing-flow)
7. [Webhook Processing Flow](#webhook-processing-flow)
8. [Error Handling](#error-handling)
9. [Monitoring & Observability](#monitoring--observability)
10. [Configuration](#configuration)

---

## Overview

The **API Gateway** serves as the single entry point for all external API requests to the Omniful platform. It acts as a reverse proxy, handling authentication, authorization, rate limiting, request routing, and response transformation.

### Key Responsibilities
- **Authentication & Authorization**: JWT validation, role-based access control
- **Request Routing**: Route requests to appropriate microservices
- **Rate Limiting**: Prevent API abuse and ensure fair usage
- **Load Balancing**: Distribute traffic across service instances
- **Request/Response Transformation**: Standardize data formats
- **Security**: CORS handling, input validation, sanitization
- **Monitoring**: Request logging, metrics collection, error tracking

---

## Architecture

### High-Level Architecture
```
┌─────────────────────────────────────────────────────────────┐
│                    External Clients                         │
│  (Web Apps, Mobile Apps, Third-party Integrations)          │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│                    API Gateway                              │
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────┐ │
│  │   Middleware    │  │   Router        │  │  Controllers │ │
│  │   Stack         │  │   Layer         │  │              │ │
│  └─────────────────┘  └─────────────────┘  └──────────────┘ │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│                Microservices                                │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐         │
│  │Sales Channel │ │   OMS        │ │  Product     │         │
│  │  Service     │ │  Service     │ │  Service     │         │
│  └──────────────┘ └──────────────┘ └──────────────┘         │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐         │
│  │  Customer    │ │   Tenant     │ │   WMS        │         │
│  │  Service     │ │  Service     │ │  Service     │         │
│  └──────────────┘ └──────────────┘ └──────────────┘         │
└─────────────────────────────────────────────────────────────┘
```

### Directory Structure
```
api-gateway/
├── main.go                    # Service entry point
├── router/
│   └── router.go             # Route definitions and middleware setup
├── controllers/              # Request handlers for different domains
│   ├── sales_channel_service/  # Sales channel related endpoints
│   ├── oms/                   # Order management endpoints
│   ├── product/               # Product catalog endpoints
│   ├── customer/              # Customer management endpoints
│   ├── tenant/                # Tenant management endpoints
│   └── webhook/               # Webhook processing endpoints
├── external_service/          # Service clients for microservices
│   ├── scs/                   # Sales Channel Service client
│   ├── oms/                   # OMS Service client
│   └── product/               # Product Service client
├── internal/
│   ├── middlewares/           # Custom middleware implementations
│   │   ├── authentication.go  # JWT authentication
│   │   ├── authorization.go   # Permission-based authorization
│   │   ├── rate_limiting.go   # Rate limiting logic
│   │   └── cors.go           # CORS handling
│   ├── access_control/        # Access control logic
│   ├── hub/                   # Hub management
│   └── seller/                # Seller management
├── configs/                   # Configuration files
├── constants/                 # Application constants
└── pkg/                       # Shared packages and utilities
```

---

## Request Flow

### Complete Request Processing Pipeline

```
1. Request Received
   ↓
2. New Relic Tracking
   ↓
3. Configuration Injection
   ↓
4. Request ID Generation
   ↓
5. Default Timeout (30s)
   ↓
6. Gzip Compression
   ↓
7. Client Details Injection
   ↓
8. Route Matching
   ↓
9. Authentication (JWT)
   ↓
10. Authorization (Permissions)
    ↓
11. Input Validation & Sanitization
    ↓
12. Rate Limiting
    ↓
13. Business Logic (Controller)
    ↓
14. External Service Call
    ↓
15. Response Transformation
    ↓
16. Response Sent
```

### Middleware Stack Order
```go
// From router/router.go
s.Engine.Use(nrgin.Middleware(newrelic.GetApplication()))     // 1. New Relic
s.Engine.Use(config.Middleware())                            // 2. Config
s.Engine.Use(newrelic.RequestIDMiddleware())                 // 3. Request ID
s.Use(middlewares.DefaultTimeout(30 * time.Second))          // 4. Timeout
s.Use(gzip.DefaultHandler().Gin)                             // 5. Compression
s.Use(middlewares.ClientMiddleware())                        // 6. Client Details
```

---

## Authentication & Authorization

### JWT Authentication Flow

#### 1. Token Extraction
```go
func extractToken(c *gin.Context) (string, error) {
    bearerToken := c.GetHeader("Authorization")
    if bearerToken == "" {
        return "", errors.New("authorization header is required")
    }
    
    if !strings.HasPrefix(bearerToken, "Bearer ") {
        return "", errors.New("invalid token format")
    }
    
    return strings.TrimPrefix(bearerToken, "Bearer "), nil
}
```

#### 2. Token Validation
```go
func AuthenticateJWT(ctx context.Context) gin.HandlerFunc {
    return func(c *gin.Context) {
        // 1. Extract token from Authorization header
        bearerToken, err := extractToken(c)
        if err != nil {
            c.AbortWithStatus(http.StatusUnauthorized)
            return
        }
        
        // 2. Parse JWT with RSA public key
        token, err := jwt.ParseWithClaims(bearerToken, &public.JWTClaim{}, func(token *jwt.Token) (interface{}, error) {
            rsaKey, err := public.GetRSAPublicKey(c, config.GetString(c, "authentication.rsaPublicKey"))
            return rsaKey, err
        })
        
        // 3. Validate token
        if !token.Valid {
            c.AbortWithStatus(http.StatusUnauthorized)
            return
        }
        
        // 4. Check user logout status
        claims := token.Claims.(*public.JWTClaim)
        isUserLogout, err := newCache.IsUserLogout(c, claims.Id)
        if isUserLogout {
            c.AbortWithStatus(http.StatusUnauthorized)
            return
        }
        
        // 5. Validate user type and domain
        if claims.UserDetails.UserType != jwt2.Tenant {
            c.AbortWithStatus(http.StatusUnauthorized)
            return
        }
        
        // 6. Add user details to context
        c.Set("user", claims.UserDetails)
        c.Next()
    }
}
```

### Authorization (Permission-Based)

#### Permission Check
```go
func (scsController *Controller) EditSalesChannelAuthorization(c *gin.Context, permissions middlewares.Permissions) bool {
    return permissions.HasPermission(permissions2.SalesChannelAppEdit)
}

func (scsController *Controller) GetSalesChannelsAuthorisation(c *gin.Context, permissions middlewares.Permissions) bool {
    return permissions.HasPermission(permissions2.SalesChannelAppView)
}
```

#### Permission Middleware
```go
func Permission(ctx context.Context) gin.HandlerFunc {
    return func(c *gin.Context) {
        user := c.MustGet("user").(public.UserDetails)
        permissions := getUserPermissions(user)
        
        // Check if user has required permission for the endpoint
        if !hasRequiredPermission(c, permissions) {
            c.AbortWithStatus(http.StatusForbidden)
            return
        }
        
        c.Set("permissions", permissions)
        c.Next()
    }
}
```

---

## Key Components

### 1. Router Layer (`router/router.go`)

#### Route Grouping
```go
// Main API routes
gateway := s.Group("/api/v1", 
    http.RequestLogMiddleware(loggingOptions),
    middlewares.AuthenticateJWT(ctx),
    middlewares.Permission(ctx),
    middlewares.SanitiseQueryParams(ctx),
    middlewares.ValidateHubAndSeller(ctx, accessControl),
)

// Sales Channel routes
setupSalesChannelServiceRoutes(ctx, hubCache, sellerCache, 
    gateway.Group("/sales_channels"))

// OMS routes
setupOMSRoutes(ctx, hubCache, sellerCache, 
    gateway.Group("/oms"))

// Product routes
setupProductRoutes(ctx, hubCache, sellerCache, 
    gateway.Group("/products"))
```

#### Sales Channel Routes
```go
func setupSalesChannelServiceRoutes(ctx context.Context, hubCache *hub.Cache, sellerCache *seller.Cache, r *gin.RouterGroup) {
    scsCtrl, err := sales_channel_service.NewController(ctx, hubCache, sellerCache)
    if err != nil {
        panic(err)
    }
    
    // GET endpoints
    r.GET("/sellers/:seller_id/sales_channels", 
        middlewares.AuthorizationMiddleware(scsCtrl.GetSalesChannelsAuthorisation), 
        scsCtrl.GetSellerSalesChannel)
    
    r.GET("/sellers/:seller_id/sales_channels/:seller_sales_channel_id/sync_products", 
        middlewares.AuthorizationMiddleware(scsCtrl.EditSalesChannelAuthorization), 
        scsCtrl.SyncEntities)
    
    // PUT endpoints
    r.PUT("/sellers/:seller_id/sales_channels/:seller_sales_channel_id", 
        middlewares.AuthorizationMiddleware(scsCtrl.AddSalesChannelAuthorization), 
        scsCtrl.UpdateSellerSalesChannel)
    
    // POST endpoints
    r.POST("/seller_sales_channels/:seller_sales_channel_id/orders/:order_id/update_payment_status", 
        middlewares.AuthorizationMiddleware(scsCtrl.AddSalesChannelAuthorization), 
        scsCtrl.UpdatePaymentStatus)
}
```

### 2. Controller Layer (`controllers/sales_channel_service/`)

#### Controller Structure
```go
type Controller struct {
    scsClient        scs.SalesChannelService
    responseHandler  *response.Handler
    hubCache         *hub.Cache
    sellerCache      *seller.Cache
}

func NewController(ctx context.Context, hubCache *hub.Cache, sellerCache *seller.Cache) (*Controller, error) {
    scsClient := scs.NewClient(ctx)
    responseHandler := response.NewHandler()
    
    return &Controller{
        scsClient:       scsClient,
        responseHandler: responseHandler,
        hubCache:        hubCache,
        sellerCache:     sellerCache,
    }, nil
}
```

#### Sync Entities Controller
```go
func (scsController *Controller) SyncEntities(ctx *gin.Context) {
    // 1. Extract parameters
    sscID := ctx.Param(constants.SellerSalesChannelID)
    queryParams := ctx.Request.URL.Query()
    
    // 2. Call Sales Channel Service
    response, interSvcErr := scsController.scsClient.SyncEntities(ctx, sscID, queryParams)
    if interSvcErr != nil {
        scsController.responseHandler.NewErrorResponseByInterServiceError(ctx, interSvcErr)
        return
    }
    
    // 3. Return success response
    scsController.responseHandler.NewAccessControlSuccessResponse(ctx, &response)
}
```

### 3. External Service Client (`external_service/scs/`)

#### Service Client
```go
type salesChannelService struct {
    Client *interserviceClient.Client
}

func NewClient(ctx context.Context) SalesChannelService {
    client := interserviceClient.NewClient(
        config.GetString(ctx, "services.salesChannelService.url"),
        config.GetString(ctx, "services.salesChannelService.timeout"),
    )
    
    return &salesChannelService{Client: client}
}
```

#### Sync Entities Client Method
```go
func (scs salesChannelService) SyncEntities(ctx *gin.Context, sscID string, queryParams map[string][]string) (res SyncEntitiesResponse, err *interserviceClient.Error) {
    headers := utils.GetDefaultHeaders(ctx)
    
    _, err = scs.Client.ExecuteWithAccessControl(ctx,
        http.APIGet,
        &http.Request{
            Url:         syncEntities,
            QueryParams: queryParams,
            Headers:     headers,
            PathParams:  map[string]string{constants.SellerSalesChannelID: sscID},
        }, &res)
    
    if err != nil {
        log.Error(err)
        return
    }
    
    return
}
```

---

## Order Processing Flow

### Complete Order Sync Flow

```
1. External Request
   POST /api/v1/sales_channels/sellers/{seller_id}/sales_channels/{ssc_id}/sync_products?sync_entity=orders
   ↓
2. API Gateway Processing
   ├── JWT Authentication
   ├── Permission Check (SalesChannelAppEdit)
   ├── Input Validation
   └── Rate Limiting
   ↓
3. Controller Handler
   scsController.SyncEntities()
   ↓
4. External Service Call
   scsClient.SyncEntities() → Sales Channel Service
   ↓
5. Sales Channel Service Processing
   ├── Validate Seller Sales Channel
   ├── Check Sync Permissions
   ├── Create SQS Event
   └── Send to SQS Queue
   ↓
6. Background Worker Processing
   ├── Fetch Orders from External Platform
   ├── Transform Data
   ├── Send to OMS Service
   └── Update Sync Status
   ↓
7. Response
   Success/Error Response to Client
```

### Detailed Flow Breakdown

#### 1. Request Reception
```http
GET /api/v1/sales_channels/sellers/261/sales_channels/4082/sync_products?sync_entity=orders&sync_from_time=1&sync_from_type=day
Authorization: Bearer <jwt_token>
Content-Type: application/json
```

#### 2. Authentication & Authorization
```go
// JWT Token Validation
claims := token.Claims.(*public.JWTClaim)
if claims.UserDetails.UserType != jwt2.Tenant {
    c.AbortWithStatus(http.StatusUnauthorized)
    return
}

// Permission Check
if !permissions.HasPermission(permissions2.SalesChannelAppEdit) {
    c.AbortWithStatus(http.StatusForbidden)
    return
}
```

#### 3. Controller Processing
```go
func (scsController *Controller) SyncEntities(ctx *gin.Context) {
    sscID := ctx.Param(constants.SellerSalesChannelID)  // "4082"
    queryParams := ctx.Request.URL.Query()              // sync_entity=orders, sync_from_time=1, sync_from_type=day
    
    // Call Sales Channel Service
    response, err := scsController.scsClient.SyncEntities(ctx, sscID, queryParams)
    if err != nil {
        scsController.responseHandler.NewErrorResponseByInterServiceError(ctx, err)
        return
    }
    
    scsController.responseHandler.NewAccessControlSuccessResponse(ctx, &response)
}
```

#### 4. External Service Call
```go
func (scs salesChannelService) SyncEntities(ctx *gin.Context, sscID string, queryParams map[string][]string) (res SyncEntitiesResponse, err *interserviceClient.Error) {
    headers := utils.GetDefaultHeaders(ctx)
    
    // HTTP GET to Sales Channel Service
    _, err = scs.Client.ExecuteWithAccessControl(ctx,
        http.APIGet,
        &http.Request{
            Url:         "internal/v1/seller_sales_channel/{seller_sales_channel_id}/sync",
            QueryParams: queryParams,  // sync_entity=orders&sync_from_time=1&sync_from_type=day
            Headers:     headers,
            PathParams:  map[string]string{constants.SellerSalesChannelID: sscID},
        }, &res)
    
    return
}
```

#### 5. Sales Channel Service Processing
```go
// Sales Channel Service receives the request
func (ac *Controller) SyncEntities(ctx *gin.Context) {
    sscID := ctx.Param("seller_sales_channel_id")  // "4082"
    syncEntity := ctx.Query("sync_entity")         // "orders"
    syncFromTime := ctx.Query("sync_from_time")    // "1"
    syncFromType := ctx.Query("sync_from_type")    // "day"
    
    // Validate parameters
    if err := s.CheckSyncEntityValidity(syncEntity); err != nil {
        return err
    }
    
    // Create SQS event
    event := sqs_events.CreateFetchOrderEvent(models.FetchOrderEventAttributes{
        SellerSalesChannelID: sscID,
        SyncFromTime:         syncFromTime,
        SyncFromType:         syncFromType,
    })
    
    // Send to SQS queue
    err := pushToSQS.SendMessageToSCSFetchOrdersFifoQueue(ctx, event)
    if err != nil {
        return err
    }
    
    return successResponse
}
```

---

## Webhook Processing Flow

### Webhook Reception Flow

```
1. External Platform Webhook
   POST /api/v1/sales_channels/webhooks/{platform}
   ↓
2. API Gateway Processing
   ├── CORS Handling
   ├── Request Logging
   └── Route to Platform Handler
   ↓
3. Platform-Specific Handler
   ├── Webhook Signature Verification
   ├── Payload Parsing
   ├── SQS Event Creation
   └── Queue Message
   ↓
4. Background Processing
   ├── SQS Message Processing
   ├── External Platform API Call
   ├── Data Transformation
   └── OMS Service Integration
   ↓
5. Response
   200 OK to External Platform
```

### Webhook Route Setup
```go
// From router/router.go
webhookGroup := gateway.Group("/webhooks")
{
    // Shopify webhooks
    webhookGroup.POST("/shopify", webhookCtrl.ShopifyWebhookHandler)
    
    // WooCommerce webhooks
    webhookGroup.POST("/woocommerce", webhookCtrl.WooCommerceWebhookHandler)
    
    // Generic webhook handler
    webhookGroup.POST("/:platform", webhookCtrl.GenericWebhookHandler)
}
```

### Webhook Controller Example
```go
func (wc *WebhookController) ShopifyWebhookHandler(ctx *gin.Context) {
    // 1. Read request body
    body, err := io.ReadAll(ctx.Request.Body)
    if err != nil {
        ctx.JSON(http.StatusBadRequest, gin.H{"error": "Failed to read body"})
        return
    }
    
    // 2. Verify webhook signature
    signature := ctx.GetHeader("X-Shopify-Hmac-Sha256")
    if !verifyShopifyWebhook(body, signature) {
        ctx.JSON(http.StatusUnauthorized, gin.H{"error": "Invalid signature"})
        return
    }
    
    // 3. Parse webhook payload
    var webhookPayload models.ShopifyWebhookPayload
    if err := json.Unmarshal(body, &webhookPayload); err != nil {
        ctx.JSON(http.StatusBadRequest, gin.H{"error": "Invalid payload"})
        return
    }
    
    // 4. Create SQS event
    event := sqs_events.CreateWebhookEvent(webhookPayload)
    
    // 5. Send to SQS queue
    err = pushToSQS.SendMessageToWebhookQueue(ctx, event)
    if err != nil {
        ctx.JSON(http.StatusInternalServerError, gin.H{"error": "Failed to process"})
        return
    }
    
    // 6. Return success
    ctx.JSON(http.StatusOK, gin.H{"status": "success"})
}
```

---

## Error Handling

### Error Response Structure
```go
type ErrorResponse struct {
    Status  string `json:"status"`
    Message string `json:"message"`
    Code    string `json:"code,omitempty"`
    Details map[string]interface{} `json:"details,omitempty"`
}
```

### Error Handling Patterns

#### 1. Authentication Errors
```go
// 401 Unauthorized
{
    "status": "error",
    "message": "Invalid or expired token",
    "code": "AUTH_001"
}
```

#### 2. Authorization Errors
```go
// 403 Forbidden
{
    "status": "error",
    "message": "Insufficient permissions",
    "code": "AUTH_002"
}
```

#### 3. Validation Errors
```go
// 400 Bad Request
{
    "status": "error",
    "message": "Invalid request parameters",
    "code": "VAL_001",
    "details": {
        "field": "sync_entity",
        "error": "must be one of: products, orders, inventory"
    }
}
```

#### 4. Service Errors
```go
// 500 Internal Server Error
{
    "status": "error",
    "message": "Internal server error",
    "code": "SVC_001"
}
```

### Error Handler Implementation
```go
func (rh *ResponseHandler) NewErrorResponseByInterServiceError(ctx *gin.Context, err *interserviceClient.Error) {
    switch err.StatusCode {
    case http.StatusUnauthorized:
        ctx.JSON(http.StatusUnauthorized, ErrorResponse{
            Status:  "error",
            Message: "Authentication failed",
            Code:    "AUTH_001",
        })
    case http.StatusForbidden:
        ctx.JSON(http.StatusForbidden, ErrorResponse{
            Status:  "error",
            Message: "Insufficient permissions",
            Code:    "AUTH_002",
        })
    case http.StatusBadRequest:
        ctx.JSON(http.StatusBadRequest, ErrorResponse{
            Status:  "error",
            Message: err.Message,
            Code:    "VAL_001",
        })
    default:
        ctx.JSON(http.StatusInternalServerError, ErrorResponse{
            Status:  "error",
            Message: "Internal server error",
            Code:    "SVC_001",
        })
    }
}
```

---

## Monitoring & Observability

### New Relic Integration
```go
// Request tracking
s.Engine.Use(nrgin.Middleware(newrelic.GetApplication()))

// Custom attributes
public.AddCommonUserDetailAttributesInNewRelic(&claims.UserDetails, c)
```

### Request Logging
```go
// Structured logging middleware
s.Use(http.RequestLogMiddleware(http.LoggingMiddlewareOptions{
    Format:      config.GetString(ctx, "log.format"),
    Level:       config.GetString(ctx, "log.level"),
    LogRequest:  config.GetBool(ctx, "log.request"),
    LogResponse: config.GetBool(ctx, "log.response"),
    LogHeader:   config.GetBool(ctx, "log.header"),
}))
```

### Health Check Endpoint
```go
// Health check route
s.GET("/health", health.HealthcheckHandler())

// Health check response
{
    "status": "healthy",
    "timestamp": "2024-01-15T10:30:00Z",
    "version": "1.0.0",
    "services": {
        "database": "healthy",
        "redis": "healthy",
        "sales_channel_service": "healthy"
    }
}
```

### Metrics Collection
```go
// Request metrics
var (
    requestDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name: "api_gateway_request_duration_seconds",
            Help: "Duration of API requests",
        },
        []string{"method", "endpoint", "status"},
    )
    
    requestCount = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "api_gateway_requests_total",
            Help: "Total number of API requests",
        },
        []string{"method", "endpoint", "status"},
    )
)
```

---

## Configuration

### Environment Configuration
```yaml
# config.yaml
server:
  port: ":8080"
  readTimeout: 30
  writeTimeout: 30
  idleTimeout: 60

authentication:
  rsaPublicKey: "path/to/public.key"
  jwtSecret: "${JWT_SECRET}"

services:
  salesChannelService:
    url: "http://sales-channel-service:8081"
    timeout: "30s"
  omsService:
    url: "http://oms-service:8082"
    timeout: "30s"
  productService:
    url: "http://product-service:8083"
    timeout: "30s"

redis:
  host: "localhost"
  port: 6379
  password: ""
  db: 0

log:
  format: "json"
  level: "info"
  request: true
  response: true
  header: false

rate_limiting:
  enabled: true
  requests_per_minute: 1000
  burst_size: 100
```

### Rate Limiting Configuration
```go
func RateLimitingMiddleware(ctx context.Context) gin.HandlerFunc {
    limiter := rate.NewLimiter(
        rate.Limit(config.GetInt(ctx, "rate_limiting.requests_per_minute")/60),
        config.GetInt(ctx, "rate_limiting.burst_size"),
    )
    
    return func(c *gin.Context) {
        if !limiter.Allow() {
            c.JSON(http.StatusTooManyRequests, ErrorResponse{
                Status:  "error",
                Message: "Rate limit exceeded",
                Code:    "RATE_001",
            })
            c.Abort()
            return
        }
        c.Next()
    }
}
```

### CORS Configuration
```go
func CORSMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Header("Access-Control-Allow-Origin", "*")
        c.Header("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
        c.Header("Access-Control-Allow-Headers", "Origin, Content-Type, Authorization")
        
        if c.Request.Method == "OPTIONS" {
            c.AbortWithStatus(http.StatusNoContent)
            return
        }
        
        c.Next()
    }
}
```

---

## Integration with Onboarding Guide

This API Gateway documentation is referenced in the main onboarding guide at:
- [Core Services Deep Dive - API Gateway](./OMNIFUL_ONBOARDING_GUIDE.md#1--api-gateway-apigateway)

### Key Integration Points

1. **Request Flow**: All external requests to Omniful go through the API Gateway
2. **Authentication**: JWT-based authentication for all protected endpoints
3. **Authorization**: Permission-based access control for different operations
4. **Service Communication**: API Gateway acts as the bridge between external clients and internal microservices
5. **Error Handling**: Centralized error handling and response formatting
6. **Monitoring**: Request tracking, logging, and metrics collection

### Related Documentation
- [Sales Channel Service Documentation](./sales-channel-service/docs/Overview.md)
- [Development Workflow](./sales-channel-service/docs/development_workflow.md)
- [Technical Reference Guide](./TECHNICAL_REFERENCE.md)

---
