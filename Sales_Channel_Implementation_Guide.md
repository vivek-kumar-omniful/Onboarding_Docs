# Sales Channel Implementation Guide

## Table of Contents
1. [Overview](#overview)
2. [Phase 1: Register the Channel](#phase-1--register-the-channel)
3. [Phase 2: Package Scaffold](#phase-2--package-scaffold)
4. [Phase 3: Router & Controllers](#phase-3--router--controllers)
5. [Phase 4: Config & URL Builders](#phase-4--config--url-builders)
6. [Phase 5: Database Models & Transactions](#phase-5--database-models--transactions)
7. [Phase 6: SQS Events & Workers](#phase-6--sqs-events--workers)
8. [Phase 7: Authentication & Token Management](#phase-7--authentication--token-management)
9. [Phase 8: Webhook Creation & Ingestion](#phase-8--webhook-creation--ingestion)
10. [Phase 9: Adapters & UI Conversions](#phase-9--adapters--ui-conversions)
11. [Phase 10: Status Mapping Seeds](#phase-10--status-mapping-seeds)
12. [Phase 11: POS/Locations Management](#phase-11--poslocations-management)
13. [Phase 12: End-to-End Login Flow](#phase-12--end-to-end-login-flow)
14. [Phase 13: Data Transfer Objects](#phase-13--data-transfer-objects)
15. [Phase 14: Testing Checklist](#phase-14--testing-checklist)
16. [Quick Reference](#quick-reference)

---

## Overview

This guide provides a comprehensive, step-by-step approach to implementing new sales channel integrations in the Omniful platform. It covers the complete lifecycle from initial setup to production deployment, including authentication, webhook handling, background processing, and testing.

**Target Audience**: Senior developers and architects implementing new eCommerce platform integrations.

**Prerequisites**: 
- Understanding of Go, Gin framework, and dependency injection
- Familiarity with SQS, PostgreSQL, and Redis
- Knowledge of the existing Omniful architecture

**Related Documentation**:
- [Sales Channel Service Documentation](./Sales_Channel_Service_Documentation.md) - High-level service overview
- [Technical Reference Guide](./Technical_Reference_Guide.md) - Common patterns and file structures
- [API Gateway Documentation](./Api_Gateway_Documentation.md) - Routing and authentication

---

## Phase 1: Register the Channel

### 1.1 Enums (ID/Tag + Maps)

**File**: `internal/enums/sales_channel_enums.go`

```go
package enums

// 1) Tag
const (
    // existing...
    AcmeSalesChannelTag string = "acme" // NEW
)

// 2) ID
const (
    // existing...
    AcmeSalesChannelID int = 1001 // NEW (pick next free int)
)

// 3) ID -> Tag
var SalesChannelIDToTagMap = map[int]string{
    // existing...
    AcmeSalesChannelID: AcmeSalesChannelTag, // NEW
}

// 4) Tag -> ID
var SalesChannelTagToSalesChannelIDMap = map[string]int{
    // existing...
    AcmeSalesChannelTag: AcmeSalesChannelID, // NEW
}
```

**Purpose**: This ties "acme" to a numeric channel ID used everywhere (DB configs, workers, routing).

### 1.2 Domain Interfaces

**File**: `internal/domain/acme.go` (mirror `internal/domain/to_you.go`)

```go
package domain

import (
    "context"
    "github.com/gin-gonic/gin"
    "github.com/omniful/sales-channel-service/models"
    "github.com/omniful/sales-channel-service/internal/workers/models/sqs_models"
    commonError "github.com/omniful/go_commons/error"
)

type AcmeService interface {
    // used by workers:
    CreateWebhook(ctx context.Context, event sqs_models.SqsSyncIntegrationEvents) commonError.CustomError

    // optionally expose helper hooks like:
    HandleDefaultOrderStatusMappingEvent(ctx context.Context, event sqs_models.SqsSyncIntegrationEvents) commonError.CustomError
}

type AcmeController interface {
    LoginController(ctx *gin.Context)
    WebhookHandler(ctx *gin.Context) // if Acme uses callbacks
}
```

**Purpose**: Domain interfaces allow workers & controllers to depend on abstractions.

### 1.3 Factory Registration

**File**: `internal/sales_channels/sales_channel_factory.go`

```go
// a) add constructor entry by ID
var salesChannelConstructorMap = map[int]salesChannelConstructor{
    // existing...
    enums.AcmeSalesChannelID: initAcmeService, // NEW
}

// b) add constructor entry by Tag
var salesChannelConstructorMapByTag = map[string]salesChannelConstructor{
    // existing...
    enums.AcmeSalesChannelTag: initAcmeService, // NEW
}

// c) the constructor itself
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

**Purpose**: Factory pattern allows the app to instantiate your service by ID/Tag.

---

## Phase 2: Package Scaffold

### 2.1 Directory Structure

Create the following directory structure:

```
internal/sales_channels/acme/
├─ service.go
├─ provider.go
├─ wire.go                 // + generated wire_gen.go
├─ controller/
│  ├─ controller.go
│  ├─ login.go
│  ├─ webhook_handler.go   // if needed
│  ├─ provider.go
│  └─ wire.go              // + wire_gen.go
├─ configurations/
│  ├─ web_interact_config.go
│  ├─ configs.go
│  ├─ default_configurations.go
│  ├─ default_order_statuses.go   // optional (seed)
│  ├─ default_sync_events.go
│  └─ http_client.go
├─ constants/
│  └─ constants.go
├─ models/
│  ├─ request/login.go
│  ├─ response/auth_response.go
│  ├─ webhook.go
│  ├─ order_transformer.go
│  └─ pos.go                      // if channel has stores
└─ services/
   ├─ integration/
   │  ├─ interface.go
   │  ├─ service.go
   │  ├─ integration.go
   │  ├─ create_webhook.go
   │  ├─ provider.go
   │  └─ wire.go                 // + wire_gen.go
   ├─ authenticator/
   │  ├─ service.go
   │  ├─ provider.go
   │  └─ wire.go                 // + wire_gen.go
   ├─ webhooks/
   │  ├─ service.go
   │  ├─ webhook_process_service.go
   │  ├─ provider.go
   │  └─ wire.go                 // + wire_gen.go
   ├─ orders/    // optional
   └─ products/  // optional
```

### 2.2 Top-Level Service

**File**: `internal/sales_channels/acme/service.go`

```go
package acme

import (
    "context"
    "sync"

    "github.com/omniful/go_commons/log"
    "github.com/omniful/sales-channel-service/internal/domain"
    "github.com/omniful/sales-channel-service/internal/sales_channels/base"
    "github.com/omniful/sales-channel-service/internal/enums"
    "github.com/omniful/sales-channel-service/internal/sales_channels/acme/configurations"
)

type Service struct {
    base.SalesChannel
    salesChannelService domain.SalesChannelService
    once sync.Once
}

func NewService(base base.SalesChannel, scs domain.SalesChannelService) *Service {
    s := &Service{SalesChannel: base, salesChannelService: scs}
    s.once.Do(func(){ s.setHost(context.Background()) })
    return s
}

func (s *Service) setHost(ctx context.Context) {
    cfg, cusErr := s.salesChannelService.GetSalesChannelConfig(ctx, enums.AcmeSalesChannelID)
    if cusErr.Exists() { log.Errorf("Acme host fetch failed: %v", cusErr); return }
    configurations.SetAcmeHost(cfg.Host)
}
```

### 2.3 Provider & Wire Configuration

**File**: `internal/sales_channels/acme/provider.go`

```go
package acme

import (
    "github.com/google/wire"
    "github.com/omniful/sales-channel-service/internal/sales_channels/base"
    "github.com/omniful/sales-channel-service/internal/sales_channel_commons"
    "github.com/omniful/sales-channel-service/internal/sales_channel_commons/cache"
    "github.com/omniful/sales-channel-service/internal/domain"
    "github.com/omniful/sales-channel-service/internal/domain/interfaces"

    "github.com/omniful/sales-channel-service/internal/sales_channels/acme/services/integration"
    "github.com/omniful/sales-channel-service/internal/sales_channels/acme/services/webhooks"
    "github.com/omniful/sales-channel-service/internal/sales_channels/acme/services/authenticator"

    redisCache "github.com/omniful/go_commons/redis_cache"
    "github.com/omniful/sales-channel-service/external_service/service/oms"
    "github.com/omniful/sales-channel-service/external_service/service/analytics"
    "github.com/omniful/sales-channel-service/external_service/service/product"
)

var ProviderSet = wire.NewSet(
    sales_channel_commons.NewRepository,
    sales_channel_commons.NewService,  // domain.SalesChannelService
    base.NewSalesChannel,

    // submodules
    integration.NewService,
    webhooks.NewService,
    authenticator.NewAuthenticator,

    // externals/caches
    oms.NewClient, analytics.NewClient, product.NewClient,
    redisCache.NewRedisCacheClient, redisCache.NewMsgpackSerializer, cache.NewCache,

    // binds
    wire.Bind(new(domain.SalesChannelRepository), new(*sales_channel_commons.Repository)),
    wire.Bind(new(redisCache.ICache), new(*redisCache.RedisCache)),
    wire.Bind(new(redisCache.ISerializer), new(*redisCache.MsgpackSerializer)),
    wire.Bind(new(interfaces.Authenticator), new(*authenticator.Authenticator)),
    wire.Bind(new(domain.SalesChannelService), new(*sales_channel_commons.Service)),

    NewService,
)
```

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

**Next Step**: Run `wire` to generate `wire_gen.go`.

---

## Phase 3: Router & Controllers

### 3.1 Router Configuration

**File**: `router/gateway_routes.go` (under `/internal/v1`, JWT-protected)

```go
acmeRoutes := publicV1.Group("/acme")
{
    acmeController, err := acme.Wire(postgres.GetCluster().DbCluster, ctx, redis.GetClient().Client, utils.GetNameSpace(ctx))
    if err != nil { return err }

    acmeRoutes.POST("/login/:seller_id", acmeController.LoginController)
}
// If Acme webhooks must be public (no JWT), mount on s.Engine directly:
s.Engine.POST("/acme/webhooks/:merchant_id/:topic", acmeController.WebhookHandler)
```

### 3.2 Controller Implementation

**File**: `internal/sales_channels/acme/controller/controller.go`

```go
package controller

import "github.com/omniful/sales-channel-service/internal/sales_channels/acme/services/integration"

type Controller struct {
    integration integration.Service
}

func NewController(i integration.Service) *Controller { return &Controller{integration: i} }
```

**File**: `internal/sales_channels/acme/controller/login.go`

```go
package controller

import (
    "strconv"
    "github.com/gin-gonic/gin"
    oresponse "github.com/omniful/go_commons/response"
    scsError "github.com/omniful/sales-channel-service/pkg/errors"
    req "github.com/omniful/sales-channel-service/internal/sales_channels/acme/models/request"
)

func (c *Controller) LoginController(ctx *gin.Context) {
    sellerID := ctx.Param("seller_id")
    var body req.LoginRequest
    if err := ctx.ShouldBindJSON(&body); err != nil {
        scsError.NewBadRequest(ctx, "invalid body"); return
    }
    ssc, cusErr := c.integration.CreateIntegration(ctx, sellerID, body)
    if cusErr.Exists() { scsError.NewErrorResponse(ctx, cusErr); return }
    oresponse.NewSuccessResponse(ctx, gin.H{
        "status": "Integration created successfully",
        "seller_sales_channel_id": strconv.Itoa(ssc.Id),
    })
}
```

**File**: `internal/sales_channels/acme/controller/webhook_handler.go` (if needed)

```go
func (c *Controller) WebhookHandler(ctx *gin.Context) {
    merchantID := ctx.Param("merchant_id")
    topic := ctx.Param("topic")
    var body map[string]any
    if err := ctx.ShouldBindJSON(&body); err != nil { ctx.Status(400); return }
    if cusErr := c.webhooks.ProcessWebhook(ctx, body, merchantID, topic); cusErr.Exists() { ctx.Status(500); return }
    ctx.Status(200)
}
```

---

## Phase 4: Config & URL Builders

### 4.1 URL Builders / Host Configuration

**File**: `internal/sales_channels/acme/configurations/web_interact_config.go`

```go
package configurations

var acmeHost string

func SetAcmeHost(host string)        { acmeHost = host }
func GetAuthURL() string              { return acmeHost + "/oauth/token" }
func GetCreateOrderWebhookURL() string   { return acmeHost + "/webhooks/orders" }
func GetCreateProductWebhookURL() string { return acmeHost + "/webhooks/products" }
// Add: orders list, single order, inventory, pos list...
```

### 4.2 Authentication HTTP Calls

**File**: `internal/sales_channels/acme/configurations/configs.go`

```go
func GenerateAccessToken(ctx context.Context, r req.LoginRequest) (res.AuthTokenResponse, error) {
    body, _ := json.Marshal(map[string]string{"email": r.Email, "password": r.Password})
    rq, _ := http.NewRequestWithContext(ctx, http.MethodPost, GetAuthURL(), bytes.NewReader(body))
    rq.Header.Set("Content-Type", "application/json")
    resp, err := (&http.Client{}).Do(rq)
    if err != nil { return res.AuthTokenResponse{}, err }
    defer resp.Body.Close()
    var out res.AuthTokenResponse
    json.NewDecoder(resp.Body).Decode(&out)
    return out, nil
}

func GenerateNewAccessToken(ctx context.Context, refreshToken string) (res.AuthTokenResponse, error) {
    // implement refresh call for Acme
    return res.AuthTokenResponse{}, nil
}
```

---

## Phase 5: Database Models & Transactions

### 5.1 Database Tables

You reuse existing tables (same as ToYou):

- `seller_sales_channels`
- `seller_sales_channel_authentication`
- `seller_sales_channel_metadata`
- `seller_sales_channel_configurations`

### 5.2 Default Configuration Builders

**File**: `internal/sales_channels/acme/configurations/default_configurations.go`

```go
func GenerateSalesChannelEntity(attrs request.LoginRequest, sellerID string, token string) *dbm.SellerSalesChannels { ... }
func GenerateSalesChannelAuth(auth response.AuthTokenResponse, attrs request.LoginRequest, sellerID string) *dbm.SellerSalesChannelAuthentication { ... }
func GenerateSellerSalesChannelMeta(attrs request.LoginRequest, sellerID string) *dbm.SellerSalesChannelMetadata { ... }
func GenerateSellerSalesChannelConfig(sscID int, attrs request.LoginRequest, sellerID string) []*dbm.SellerSalesChannelConfigurations { ... } // Inventory/Sync/Picking/Shipment
```

### 5.3 Repository Transaction Pattern

Use the commons repository exactly as ToYou does. Pattern:

```go
// inside integration service
tx := repo.db.BeginTx(ctx) // (your repo exposes a Tx helper)
defer func() { if r := recover(); r != nil { tx.Rollback() } }()

// 1) seller_sales_channels
if err := tx.Create(ssc).Error; err != nil { tx.Rollback(); return nil, wrap(err) }

// 2) auth (set SellerSalesChannelId after ssc.ID is known)
authRow.SellerSalesChannelId = ssc.Id
if err := tx.Create(authRow).Error; err != nil { tx.Rollback(); return nil, wrap(err) }

// 3) metadata
meta.SellerSalesChannelId = ssc.Id
if err := tx.Create(meta).Error; err != nil { tx.Rollback(); return nil, wrap(err) }

// 4) configurations (bulk)
for _, c := range configs { c.SellerSalesChannelId = ssc.Id }
if err := tx.Create(&configs).Error; err != nil { tx.Rollback(); return nil, wrap(err) }

if err := tx.Commit().Error; err != nil { return nil, wrap(err) }

return ssc, ok
```

**Note**: Your repo already has helpers like `CreateSellerSalesChannel`, `CreateSellerSalesChannelAuth`, etc. — mirror ToYou's exact calls.

---

## Phase 6: SQS Events & Workers

### 6.1 Enqueue Default Events (after DB commit)

**File**: `internal/sales_channels/acme/configurations/default_sync_events.go`

```go
func CreateAcmeDefaultEvents(ctx *gin.Context, ssc *models.SellerSalesChannels) commonError.CustomError {
    if cusErr := scs_sqs_events.CreateWebhookEvent(ctx, ssc); cusErr.Exists() { return cusErr }
    if cusErr := scs_sqs_events.CreateOrderStatusMappingEvent(ctx, ssc); cusErr.Exists() { return cusErr }
    return commonError.CustomError{}
}
```

### 6.2 Event Model (already exists)

**File** (existing): `internal/workers/models/sqs_models/sync_integration_events.go`

```go
type SqsSyncIntegrationEvents struct {
    EventType            string
    SellerSalesChannelID int64
    SalesChannelID       int
    SellerID             string
    MerchantID           string
    //...
}
```

Use `CreateWebhook` / `DefaultOrderStatusMapping` constants from the shared constants.

### 6.3 Worker Dispatch

**File**: `internal/workers/handlers/sqs_integration_events_handler.go`

```go
switch event.EventType {
case constants.CreateWebhook:
    switch event.SalesChannelID {
    case enums.AcmeSalesChannelID:
        return commonService.SalesChannelServices.AcmeService.CreateWebhook(ctx, event)
    }

case constants.DefaultOrderStatusMapping:
    switch event.SalesChannelID {
    case enums.AcmeSalesChannelID:
        return commonService.SalesChannelServices.AcmeService.HandleDefaultOrderStatusMappingEvent(ctx, event)
    }
}
```

### 6.4 Common Service Wiring

**File**: `internal/common_service/common_service.go`

```go
type SalesChannelServices struct {
    // existing...
    AcmeService domain.AcmeService // NEW
}

func NewSalesChannelServices(...) *SalesChannelServices {
    // create Acme
    acmeSvc, err := acme.ServiceWire(postgres.GetCluster().DbCluster, ctx, redis.GetClient().Client, utils.GetNameSpace(ctx))
    if err != nil { log.Errorf("Acme wire failed: %v", err) }

    return &SalesChannelServices{
        // existing...
        AcmeService: acmeSvc,
    }
}
```

---

## Phase 7: Authentication & Token Management

### 7.1 Authenticator Service

**File**: `internal/sales_channels/acme/services/authenticator/service.go`

```go
type Authenticator struct { authSvc interfaces.Service }

func NewAuthenticator(s interfaces.Service) *Authenticator { return &Authenticator{authSvc: s} }

func (a *Authenticator) FetchAuthorization(ctx context.Context, model interfaces.AuthModel) (authResponses.AuthorizationRes, commonError.CustomError) {
    // if model.IsExpired() { refresh & save }
    key, val := "Authorization", "Bearer "+model.GetAccessToken()
    return authResponses.AuthorizationRes{AuthorizationKey: key, AuthorizationValue: val}, commonError.CustomError{}
}

func (a *Authenticator) GenerateRefreshAuthFunc(ctx context.Context, model interfaces.AuthModel) func(ctx context.Context, req *http.Request) error {
    return func(ctx context.Context, req *http.Request) error {
        // call configurations.GenerateNewAccessToken(model.GetRefreshToken())
        // model.RefreshAuthModel(newAccess, newRefresh, time.Now().Add(45*time.Minute))
        // a.authSvc.Save(ctx, model)
        req.Header.Set("Authorization", "Bearer "+model.GetAccessToken())
        return nil
    }
}
```

**Hook the httpclient** (same pattern as ToYou):

Build client with `ClientAttributes{ SSCID, SellerID, AccessTokenRefresherFunc: a.GenerateRefreshAuthFunc(...) }`.

The custom client's Unauthorized middleware will call your refresher and retry the original request.

---

## Phase 8: Webhook Creation & Ingestion

### 8.1 Create Webhooks (async)

**File**: `internal/sales_channels/acme/services/integration/create_webhook.go`

```go
func (s *Service) CreateWebhook(ctx context.Context, event sqs_models.SqsSyncIntegrationEvents) commonError.CustomError {
    // 1) load auth model for SSCID
    model, cusErr := s.auth.FetchAuthModel(ctx, authRequests.FetchAuthModelReq{
        SalesChannelID: enums.AcmeSalesChannelID,
        SSCID:          int(event.SellerSalesChannelID),
    })
    if cusErr.Exists() { return cusErr }

    // 2) get auth header now (client also has auto-refresh)
    authHdr, cusErr := s.auth.FetchAuthorization(ctx, model)
    if cusErr.Exists() { return cusErr }

    // 3) POST webhooks
    if err := s.postWebhook(ctx, configurations.GetCreateOrderWebhookURL(), authHdr, payloadFor("order", event)); err.Exists() { return err }
    if err := s.postWebhook(ctx, configurations.GetCreateProductWebhookURL(), authHdr, payloadFor("product", event)); err.Exists() { return err }

    return commonError.CustomError{}
}
```

`payloadFor` builds the callback URL to your handler: e.g., `fmt.Sprintf("%s/%s/%s", callbackBase, event.MerchantID, topic)`.

### 8.2 Webhook Ingestion & Dispatch

**File**: `internal/sales_channels/acme/services/webhooks/webhook_process_service.go`

```go
func (s *Service) ProcessWebhook(ctx *gin.Context, body map[string]any, merchantID, topic string) commonError.CustomError {
    switch topic {
    case enums.OmnifulOrderTopic:
        // map event -> internal minimal order
        return scs_sqs_events.PushFetchOrder(ctx, /* attrs... include SSCID by merchantID lookup */)
    case enums.OmnifulProductTopic:
        return scs_sqs_events.PushFetchProduct(ctx, /* attrs... */)
    default:
        return commonError.NewBadRequestError("unknown topic")
    }
}
```

---

## Phase 9: Adapters & UI Conversions

Usually no changes are needed. Generic adapters live at:

`internal/sales_channel_commons/adapters/adapters.go`

They convert service results to controller responses. Only touch if Acme needs special fields in generic endpoints (e.g., adding per-channel metadata fields to list views). When you do:

```go
func ConvertSvcResToGatewayCtrlResGetSellerSalesChannels(in *svcResType) (out ctrlResType) {
    // add/override fields if Acme requires, otherwise defer to common mapping
}
```

**Tip**: Keep adapter changes minimal; prefer channel-specific endpoints under `/acme` if you need special shapes.

---

## Phase 10: Status Mapping Seeds

If Acme has bespoke order statuses, add seed defaults (like ToYou):

**File**: `internal/sales_channels/acme/configurations/default_order_statuses.go`

```go
package configurations
import "github.com/omniful/sales-channel-service/internal/enums"

var DefaultAcmeToOmnifulStatus = []StatusMap{
    { External: "NEW",      Internal: enums.OmniOrderStatus_New },
    { External: "CANCELLED",Internal: enums.OmniOrderStatus_Cancelled },
    { External: "SHIPPED",  Internal: enums.OmniOrderStatus_Shipped },
    // ...
}
```

Then in your `DefaultOrderStatusMapping` event handler (Phase 6.3), write these to the mapping table for that SSC.

---

## Phase 11: POS/Locations Management

If the channel has multiple stores, expose locations (used by UI for Hub mapping):

**Endpoint**: `GET /internal/v1/seller_sales_channel/:seller_sales_channel_id/locations`

Implement by calling Acme POS API and transforming to `GlobalFetchAllLocationsSvcResponse` (same as ToYou's `pos_transformer.go`).

Map POS→Hub via:

**Endpoint**: `PUT /internal/v1/hubs`

(already implemented in commons; you don't change code, you just supply the POS key as `location_id`).

Inventory updates include the POS key to target the correct store (mirror ToYou's `/pos-product-availability` flow).

---

## Phase 12: End-to-End Login Flow

**Endpoint**: `POST /internal/v1/acme/login/:seller_id` (JWT header required)

**Flow**: Controller → `integration.CreateIntegration`:

1. `GenerateAccessToken` (Acme)
2. Build: `SellerSalesChannels`, `SellerSalesChannelAuthentication`, `SellerSalesChannelMetadata`, `SellerSalesChannelConfigurations`
3. Persist in one transaction
4. Enqueue SQS: `CreateWebhook` + `DefaultOrderStatusMapping`
5. Return `{ seller_sales_channel_id }`

**Worker consumes CreateWebhook**:

1. Load tokens (auto-refresh on 401)
2. POST order + product webhook regs to Acme

**Acme calls back** → `WebhookHandler`:

1. Parse → push `FetchOrder/Product` SQS

Orders/products/inventory flows reuse the authenticated httpclient with 401 auto-refresh via your authenticator.

---

## Phase 13: Data Transfer Objects

### Request/Response Models

**File**: `internal/sales_channels/acme/models/request/login.go`

```go
type LoginRequest struct {
    Email              string `json:"email" validate:"required"`
    Password           string `json:"password" validate:"required"`
    ExternalMerchantID string `json:"external_merchant_id" validate:"required"`
    StoreName          string `json:"store_name" validate:"required"`
}
```

**File**: `internal/sales_channels/acme/models/response/auth_response.go`

```go
type AuthTokenResponse struct {
    AccessToken  string `json:"access_token"`
    RefreshToken string `json:"refresh_token"`
    ExpiresIn    int64  `json:"expires_in"`
}
```

---

## Phase 14: Testing Checklist

### JWT Authentication
- Send header `constants.JWTHeader` with a dev (even `alg=none`) token

### Login Verification
Verify rows in DB:
- `seller_sales_channels` (`external_merchant_id` set)
- `seller_sales_channel_authentication` (`access/refresh`, `expires_in`)
- `seller_sales_channel_metadata`
- `seller_sales_channel_configurations` (`inventory/sync/picking/shipment`)

### SQS Verification
- Messages for `CreateWebhook` & `DefaultOrderStatusMapping`

### Webhook Registration
- Worker logs show POSTs to Acme endpoints
- Repeat on token expiry to see refresh

### Webhook Reception
- `POST /acme/webhooks/:merchant_id/:topic` with sample → observe SQS push

### Locations (if applicable)
- `GET /internal/v1/seller_sales_channel/:id/locations` returns POS list (IDs = posKey)

### Hub Mapping
- `PUT /internal/v1/hubs` maps POS to hub

---

## Quick Reference

### Files to Add/Change

1. **`internal/enums/sales_channel_enums.go`**: add `AcmeSalesChannelID/Tag`, maps
2. **`internal/sales_channels/sales_channel_factory.go`**: add to maps + `initAcmeService`
3. **`internal/domain/acme.go`**: interface(s)
4. **`internal/sales_channels/acme/`**: scaffold folders & files (service, controller, configurations, services/*, constants, models)
5. **`internal/common_service/common_service.go`**: create & assign `AcmeService`
6. **`internal/workers/handlers/sqs_integration_events_handler.go`**: dispatch `CreateWebhook` & `DefaultOrderStatusMapping` for `AcmeSalesChannelID`
7. **`router/gateway_routes.go`**: mount `/internal/v1/acme/login/:seller_id` (+ public webhook if needed)
8. **Run wire** in packages with `wire.go` and rebuild

### Key Commands

```bash
# Generate wire code
cd internal/sales_channels/acme
go generate

# Build and test
go build ./...
go test ./...

# Run service
go run main.go -mode=http
```

### Common Issues & Solutions

1. **Wire generation fails**: Check import paths and provider sets
2. **Service not found**: Verify factory registration and enum values
3. **Webhook not working**: Check route mounting and JWT requirements
4. **SQS events not processing**: Verify worker dispatch logic and common service wiring

---

## Related Documentation

- **[Sales Channel Service Documentation](./Sales_Channel_Service_Documentation.md)** - High-level service overview and architecture
- **[Technical Reference Guide](./Technical_Reference_Guide.md)** - Common patterns, file structures, and best practices
- **[API Gateway Documentation](./Api_Gateway_Documentation.md)** - Routing, authentication, and API patterns
- **[OMS Service Documentation](./OMS_Service_Documentation.md)** - Order management integration
- **[Product Service Documentation](./Product_Service_Documentation.md)** - Product catalog integration

---

*This guide provides a comprehensive implementation path for new sales channel integrations. For questions or clarifications, refer to the existing documentation or consult with the development team.*
