# Omniful Tenant Service Documentation

## Table of Contents
1. [Overview](#overview)
2. [Architecture & Design](#architecture--design)
3. [Core Functionality](#core-functionality)
4. [API Endpoints](#api-endpoints)
5. [Data Models & Database](#data-models--database)
6. [Worker System](#worker-system)
7. [Integration Patterns](#integration-patterns)
8. [Configuration](#configuration)
9. [Development & Deployment](#development--deployment)
10. [Common Use Cases](#common-use-cases)

---

## Overview

The **Tenant Service** is a foundational microservice in the Omniful platform responsible for managing multi-tenancy, user authentication, authorization, and tenant-specific configurations.

### Key Responsibilities
- **Multi-tenancy Management**: Create, manage, and isolate data between different tenants
- **User Authentication & Authorization**: Handle user login, OTP verification, and role-based access control
- **Tenant Configuration**: Manage tenant-specific settings, feature flags, and integrations
- **User Management**: Create, update, and manage user accounts across the platform
- **Role & Permission Management**: Define and manage user roles and permissions
- **Address & Location Management**: Handle geographic data, city mappings, and address validation

### Service Characteristics
- **Database**: PostgreSQL (primary), Redis (caching)
- **Authentication**: JWT-based with RSA key pairs
- **Message Queue**: AWS SQS for asynchronous processing
- **Event Streaming**: Kafka for real-time updates
- **API Style**: RESTful HTTP APIs with JWT authentication
- **Multi-mode Operation**: HTTP server, worker mode, and migration mode

---

## Architecture & Design

### Service Architecture
```
┌─────────────────────────────────────────────────────────────┐
│                   Tenant Service                            │
├─────────────────────────────────────────────────────────────┤
│    ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│    │   HTTP API  │  │   Workers   │  │  Migration  │        │
│    │   Server    │  │   System    │  │   System    │        │
│    └─────────────┘  └─────────────┘  └─────────────┘        │
├─────────────────────────────────────────────────────────────┤
│    ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│    │ Controllers │  │   Services  │  │   Models    │        │
│    │             │  │             │  │             │        │
│    └─────────────┘  └─────────────┘  └─────────────┘        │
├─────────────────────────────────────────────────────────────┤
│    ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│    │ PostgreSQL  │  │    Redis    │  │    Kafka    │        │
│    │             │  │             │  │             │        │
│    └─────────────┘  └─────────────┘  └─────────────┘        │
└─────────────────────────────────────────────────────────────┘
```

### Directory Structure
```
tenant-service/
├── main.go                           # Service entry point
├── configs/                          # Configuration files
├── router/                           # API routing
├── internal/                         # Core business logic
│   ├── tenant/                      # Tenant management
│   ├── omniful_user/                # User management
│   ├── omniful_role/                # Role management
│   ├── omniful_permission/          # Permission management
│   ├── address/                     # Address management
│   ├── city/                        # City management
│   ├── workers/                     # Background job processors
│   └── utils/                       # Utility functions
├── pkg/                             # Shared packages
└── deployment/                      # Deployment configurations
```

---

## Core Functionality

### 1. Tenant Management
- **Tenant Creation**: Create new tenant accounts with basic information
- **Tenant Updates**: Modify existing tenant details and configurations
- **Tenant Isolation**: Ensure data separation between tenants
- **Tenant Validation**: Ensure tenant data integrity and business rules

### 2. User Management
- **User Creation**: Create new user accounts with roles and permissions
- **User Authentication**: Handle login, password management, and session handling
- **User Updates**: Modify existing user details and preferences
- **User Validation**: Ensure user data integrity and business rules

### 3. Role & Permission Management
- **Role Definition**: Define system-wide and tenant-specific roles
- **Permission Assignment**: Assign granular permissions to roles
- **Access Control**: Enforce role-based access control (RBAC)
- **Permission Validation**: Ensure proper permission checks

### 4. Address & Location Management
- **Address Validation**: Validate and standardize addresses
- **City Management**: Manage city data and mappings
- **Location Services**: Provide geocoding and reverse geocoding
- **Address Autofill**: Auto-complete address information

---

## API Endpoints

### Core Endpoints

#### Tenant Management
```
POST   /api/v1/lead                    # Create tenant lead
GET    /api/v1/leads                   # Get tenant leads
POST   /api/v1                        # Create tenant
GET    /api/v1/:tenant_id             # Get tenant
GET    /api/v1                        # Get tenants
PUT    /api/v1/:tenant_id             # Update tenant
```

#### User Management
```
POST   /api/v1/tenant/:tenant_id/users     # Create user
GET    /api/v1/tenant/:tenant_id/users     # Get users
GET    /api/v1/tenant/:tenant_id/users/:id # Get user
PUT    /api/v1/tenant/:tenant_id/users/:id # Update user
DELETE /api/v1/tenant/:tenant_id/users/:id # Delete user
```

#### Authentication
```
POST   /api/v1/auth/login              # User login
POST   /api/v1/auth/logout             # User logout
POST   /api/v1/auth/refresh            # Refresh token
POST   /api/v1/auth/verify-otp         # Verify OTP
```

#### Address & Location
```
POST   /api/v1/tenant/:tenant_id/addresses    # Create address
GET    /api/v1/tenant/:tenant_id/addresses    # Get addresses
GET    /api/v1/cities                          # Get cities
POST   /api/v1/cities                          # Create city
```

---

## Data Models & Database

### Core Database Tables

#### Tenants Table
```sql
CREATE TABLE tenants (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    domain VARCHAR(255) UNIQUE,
    status VARCHAR(50) DEFAULT 'active',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### Users Table
```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    tenant_id INTEGER REFERENCES tenants(id),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255),
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    status VARCHAR(50) DEFAULT 'active',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### Roles Table
```sql
CREATE TABLE roles (
    id SERIAL PRIMARY KEY,
    tenant_id INTEGER REFERENCES tenants(id),
    name VARCHAR(100) NOT NULL,
    description TEXT,
    is_system_role BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

## Worker System

### Background Job Processing
The service runs multiple worker processes for asynchronous operations:

#### Core Workers
1. **User Management Workers**
   - `NewCreateUserListener`: Process user creation requests
   - `NewEmailListener`: Handle email notifications

2. **Location Workers**
   - `NewLocationMappingListener`: Process location mapping events
   - `NewCityListener`: Handle city-related operations

3. **Cache Workers**
   - `NewInvalidateRedisListener`: Purge Redis cache

4. **Notification Workers**
   - `NewNotificationListener`: Handle notification events

---

## Integration Patterns

### 1. Service-to-Service Communication
- **HTTP API Calls**: Direct service-to-service communication
- **Event-Driven Communication**: Kafka-based event publishing

### 2. External Service Integration
- **Google Maps API**: Geocoding and address validation
- **Firebase**: Push notifications
- **Slack**: Notifications and alerts

### 3. Authentication Integration
- **JWT Token Generation**: RSA-signed tokens
- **OTP Verification**: Time-based one-time passwords

---

## Configuration

### Environment Configuration
```yaml
server:
  port: ":8081"

postgresql:
  database: "crud"
  master:
    host: "localhost"
    port: 5400
    username: "admin"
    password: "admin"

redis:
  hosts: "127.0.0.1:7005"
  db: 1

authentication:
  rsaPrivateKey: "-----BEGIN PRIVATE KEY-----..."
  rsaPublicKey: "-----BEGIN PUBLIC KEY-----..."
```

---

## Development & Deployment

### Local Development Setup
```bash
# Start dependencies
docker-compose up -d

# Run the service
go run main.go

# Run migrations
go run main.go -mode=migration -migrationType=up
```

### Database Migrations
```bash
# Run migrations up
go run main.go -mode=migration -migrationType=up

# Run migrations down
go run main.go -mode=migration -migrationType=down
```

---

## Common Use Cases

### 1. Tenant Onboarding Workflow
```
1. Create tenant lead → 2. Validate lead → 3. Create tenant → 4. Setup default roles → 5. Create admin user
```

### 2. User Authentication Flow
```
1. User login → 2. Validate credentials → 3. Generate JWT → 4. Set user session → 5. Return access token
```

### 3. Address Validation & Geocoding
```
1. Receive address → 2. Validate format → 3. Geocode with Google → 4. Store coordinates → 5. Return standardized address
```

---
