# Architecture

## System Overview

Ön Muhasebe follows a **modular monolith** architecture with clear domain boundaries. The system is designed as a single deployable unit while maintaining strict module separation, making it straightforward to extract modules into microservices when scaling demands it.

## High-Level Architecture

```
                         ┌──────────────┐
                         │   Browser    │
                         │  React SPA   │
                         └──────┬───────┘
                                │ HTTPS
                         ┌──────▼───────┐
                         │    Nginx     │
                         │   (:7093)    │
                         │  Reverse     │
                         │  Proxy       │
                    ┌────┴──────┬───────┘
                    │           │
            /assets, /     /api/**
                    │           │
         ┌──────────▼──┐  ┌────▼──────────────┐
         │  Static     │  │  Spring Boot      │
         │  Files      │  │  (:8093)          │
         │  (dist/)    │  │                   │
         └─────────────┘  │  ┌─────────────┐  │
                          │  │  Security   │  │
                          │  │  Layer      │  │
                          │  └──────┬──────┘  │
                          │  ┌──────▼──────┐  │
                          │  │  Business   │  │
                          │  │  Modules    │  │
                          │  └──────┬──────┘  │
                          └─────────┼─────────┘
                       ┌────────────┼────────────┐
                       │            │            │
                ┌──────▼──┐  ┌─────▼───┐  ┌─────▼─────┐
                │PostgreSQL│  │  Redis  │  │ ActiveMQ  │
                │  (RLS)   │  │ Cache   │  │ Artemis   │
                └──────────┘  └─────────┘  └───────────┘
```

## Backend Architecture

### Module Structure

The backend is organized into domain-driven modules, each containing its own entities, repositories, controllers, and services:

```
com.financialtracker.backend
├── shared/                 # Cross-cutting concerns
│   ├── BaseEntity          # Auditable base (createdAt, updatedAt, tenantId)
│   ├── ApiResponseMapper   # Standardized API responses
│   ├── GlobalExceptionHandler
│   ├── BusinessException   # Domain-specific exceptions
│   ├── AsyncJobService     # Background task processing
│   └── ExchangeRateService # Currency rate integration
│
├── auth/                   # Authentication & Authorization
│   ├── entities/           # User, AppUser, Role, Permission
│   ├── security/           # JwtService, JwtAuthFilter, SecurityConfig
│   ├── controllers/        # AuthController, UserManagement, RoleManagement
│   └── services/           # TokenStoreService
│
├── stock/                  # Inventory Management
│   ├── entities/           # Product, StockBalance, StockMovement,
│   │                       # WarehouseLocation, ProductVariant, etc.
│   ├── repositories/       # 9 repositories
│   └── controllers/        # 4 controllers
│
├── invoice/                # Invoicing
│   ├── entities/           # Invoice, InvoiceItem, InvoiceStatus
│   ├── services/           # InvoiceService (PDF generation, calculations)
│   └── controllers/        # Invoice CRUD + PDF export
│
├── chequepromissory/       # Cheque Management
│   ├── entities/           # Check, CheckEvent, LedgerEntry
│   ├── enums/              # 6 enums (status, type, etc.)
│   ├── services/           # CheckService (lifecycle management)
│   └── controllers/        # Cheque operations
│
├── cashbank/               # Cash & Bank Operations
│   ├── entities/           # CashRegister, CashTransaction,
│   │                       # BankAccount, BankTransaction
│   └── controllers/        # Transaction management
│
├── sales/                  # Sales Module
│   ├── entities/           # SalesOrder, SalesOrderItem
│   └── controllers/        # Order processing
│
├── purchasing/             # Purchasing Module
│   ├── entities/           # PurchaseOrder, PurchaseOrderItem
│   └── controllers/        # Purchase management
│
├── currentaccount/         # Current Accounts
│   ├── entities/           # Customer, Supplier, CurrentAccountTransaction
│   └── controllers/        # Account operations
│
├── branch/                 # Multi-Branch
│   ├── entities/           # Branch, BranchUser, BranchTransfer
│   └── controllers/        # Branch & transfer operations
│
├── reporting/              # Analytics & Reports
│   └── controllers/        # Report generation endpoints
│
└── config/                 # Application Configuration
    ├── SecurityConfig      # CORS, filter chain, auth rules
    ├── WebConfig           # MVC configuration
    ├── DatabaseUrlConfig   # Dynamic datasource config
    └── RabbitMqConfig      # Message broker setup
```

### Security Architecture

```
Request Flow:

  HTTP Request
       │
       ▼
  ┌─────────────────┐
  │  Rate Limiter   │──── Too many requests ──► 429
  └────────┬────────┘
           ▼
  ┌─────────────────┐
  │  JWT Auth       │──── Invalid/Expired ────► 401
  │  Filter         │
  └────────┬────────┘
           ▼
  ┌─────────────────┐
  │  @PreAuthorize  │──── Missing Permission ─► 403
  │  (per endpoint) │
  └────────┬────────┘
           ▼
  ┌─────────────────┐
  │  Tenant Filter  │──── Row-Level Security
  │  (BaseEntity)   │
  └────────┬────────┘
           ▼
     Business Logic
```

**JWT Token Structure:**
- Access token contains: `userId`, `username`, `roles[]`, `permissions[]`, `tenantId`
- Tokens are validated on every request via `JwtAuthFilter`
- Token storage service handles refresh token rotation

**RBAC Model:**
- 5 predefined roles with granular permissions
- Permissions are stored in JWT claims for stateless authorization
- Backend: `@PreAuthorize("hasAuthority('PERMISSION_NAME')")` on all controllers
- Frontend: `useAuth()` hook with `hasRole()`, `hasPermission()` helpers

### Multi-Tenancy Strategy

Row-level security is implemented through the `BaseEntity` class:

```
Every entity extends BaseEntity
       │
       ├── tenantId (set automatically from JWT)
       ├── createdAt (audit timestamp)
       └── updatedAt (audit timestamp)

All queries are filtered by tenantId automatically,
ensuring complete data isolation between tenants.
```

## Frontend Architecture

### Component Structure

```
client/src/
├── App.tsx                 # Router with 40+ routes + ProtectedRoute guards
├── components/
│   ├── ui/                 # shadcn/ui primitives (Button, Dialog, Table, etc.)
│   ├── layout/             # Sidebar, TopBar, MainLayout
│   └── shared/             # DataTable, FormFields, Charts
├── pages/
│   ├── dashboard/          # Analytics & KPIs
│   ├── sales/              # Sales orders
│   ├── purchasing/         # Purchase orders
│   ├── stock/              # Inventory management
│   ├── invoices/           # Invoice CRUD
│   ├── cheques/            # Cheque management
│   ├── cashbank/           # Cash & bank operations
│   ├── customers/          # Customer management
│   ├── suppliers/          # Supplier management
│   ├── branches/           # Branch management
│   ├── admin/              # User & role management
│   └── settings/           # Application settings
├── hooks/
│   ├── useAuth.ts          # Auth state + RBAC helpers
│   ├── use-toast.ts        # Toast notifications
│   └── use-mobile.ts       # Responsive detection
├── lib/
│   ├── api.ts              # Axios client with JWT interceptors
│   ├── queryClient.ts      # TanStack Query configuration
│   └── utils.ts            # Shared utilities
└── types/                  # TypeScript interfaces & enums
```

### State Management

```
┌──────────────────────────────────────────┐
│              TanStack Query              │
│         (Server State Manager)           │
│                                          │
│  ┌────────────┐  ┌───────────────────┐   │
│  │ Query Cache │  │ Mutation Handlers │   │
│  │ (auto-     │  │ (optimistic       │   │
│  │  refetch)  │  │  updates)         │   │
│  └────────────┘  └───────────────────┘   │
├──────────────────────────────────────────┤
│          React Context (Auth)            │
│  JWT token, user info, RBAC helpers      │
├──────────────────────────────────────────┤
│          React Hook Form + Zod           │
│  Form state, validation, error handling  │
└──────────────────────────────────────────┘
```

### Route Protection

```
<Route element={<ProtectedRoute requiredPermission="SALES_VIEW" />}>
  <Route path="/sales" element={<SalesPage />} />
</Route>

ProtectedRoute checks:
1. Is user authenticated? (JWT valid)
2. Does user have required permission?
3. If no → redirect to /unauthorized or /login
```

## Database Architecture

### Migration Strategy

The database schema evolves through **24 numbered Flyway migrations** (V1 through V24), ensuring reproducible deployments and version-controlled schema changes.

### Core Entity Relationships

```
                    ┌──────────┐
                    │  Tenant  │
                    └────┬─────┘
                         │ owns
          ┌──────────────┼──────────────┐
          │              │              │
    ┌─────▼─────┐  ┌────▼────┐  ┌──────▼──────┐
    │  Branch   │  │  Users  │  │  Products   │
    └─────┬─────┘  └────┬────┘  └──────┬──────┘
          │              │              │
    ┌─────▼─────────┐   │    ┌─────────▼─────────┐
    │ BranchTransfer│   │    │  StockBalance     │
    └───────────────┘   │    │  StockMovement    │
                        │    │  WarehouseLocation│
                        │    └───────────────────┘
                        │
         ┌──────────────┼──────────────────┐
         │              │                  │
   ┌─────▼──────┐ ┌────▼─────┐ ┌──────────▼──────┐
   │  Customer  │ │ Supplier │ │  CashRegister   │
   └─────┬──────┘ └────┬─────┘ │  BankAccount    │
         │              │       └────────┬────────┘
   ┌─────▼──────────────▼──┐    ┌────────▼────────┐
   │  CurrentAccount       │    │  Transactions   │
   │  Transaction          │    │  (Cash/Bank)    │
   └───────────────────────┘    └─────────────────┘
         │
   ┌─────▼────────┐    ┌──────────────┐
   │  SalesOrder  │    │   Invoice    │
   │  PurchaseOrd │    │   Quote      │
   └──────────────┘    └──────┬───────┘
                              │
                        ┌─────▼──────┐
                        │   Check    │
                        │   Event    │
                        │   Ledger   │
                        └────────────┘
```

## Infrastructure

### Deployment Architecture

```
┌────────────────────────────────────────────┐
│              AWS EC2 Instance              │
│              Ubuntu 24.04 LTS              │
│                                            │
│  ┌──────────────────────────────────────┐  │
│  │  Nginx (:7093)                       │  │
│  │  ├── / → /var/www/financialtracker/  │  │
│  │  └── /api → localhost:8093           │  │
│  └──────────────────────────────────────┘  │
│                                            │
│  ┌──────────────────────────────────────┐  │
│  │  Systemd Service: backend            │  │
│  │  └── java -jar /home/ubuntu/         │  │
│  │       backend.jar (:8093)            │  │
│  └──────────────────────────────────────┘  │
│                                            │
│  ┌──────────────┐  ┌──────────────┐        │
│  │  PostgreSQL  │  │    Redis     │        │
│  └──────────────┘  └──────────────┘        │
│                                            │
│  ┌──────────────────────────────────────┐  │
│  │  ActiveMQ Artemis (Message Broker)   │  │
│  └──────────────────────────────────────┘  │
└────────────────────────────────────────────┘
```

### CI/CD Pipeline

```
Developer Push → GitHub Actions
                    │
            ┌───────┴───────┐
            │  Build & Test │
            │  (Maven +     │
            │   Vite)       │
            └───────┬───────┘
                    │
            ┌───────▼───────┐
            │  Deploy via   │
            │  SFTP/SCP     │
            └───────┬───────┘
                    │
            ┌───────▼───────┐
            │  Restart      │
            │  Services     │
            └───────────────┘
```

## Performance Considerations

- **Redis caching** for frequently accessed data (exchange rates, reference data)
- **TanStack Query** with smart cache invalidation on the frontend
- **Database indexing** on tenant_id and frequently queried columns
- **Async processing** via ActiveMQ for non-blocking operations
- **Rate limiting** on API endpoints to prevent abuse
- **Prometheus metrics** via Spring Actuator for monitoring

## Security Measures

1. **JWT-based authentication** with token rotation
2. **Row-level security** for multi-tenant data isolation
3. **RBAC** with granular permissions on every API endpoint
4. **Rate limiting** per user/IP
5. **CORS** configuration for allowed origins
6. **Input validation** with Zod (frontend) and Bean Validation (backend)
7. **SQL injection prevention** via JPA parameterized queries
8. **XSS protection** through React's built-in escaping
