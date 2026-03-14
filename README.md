<p align="center">
  <h1 align="center">Ön Muhasebe</h1>
  <p align="center">
    <strong>Enterprise-Grade Multi-Tenant SaaS Accounting Platform</strong>
  </p>
  <p align="center">
    A comprehensive financial management system built for small and medium-sized businesses in Turkey, featuring multi-branch operations, inventory tracking, invoicing, check management, and role-based access control.
  </p>
</p>

---

> **Note:** This is a showcase repository containing documentation and architecture details only. The source code is maintained in a private repository.

## Overview

Ön Muhasebe is a full-stack, multi-tenant SaaS accounting platform designed to handle the complete financial operations of Turkish SMBs. The system supports multi-branch operations with row-level security, ensuring complete data isolation between tenants.

### Key Highlights

- **40+ pages** covering all core accounting workflows
- **25+ REST API controllers** with granular RBAC authorization
- **24 database migrations** for a robust, versioned schema
- **Row-level security** for complete multi-tenant data isolation
- **Real-time** exchange rate integration and async job processing

## Features

### Financial Management
| Module | Description |
|--------|-------------|
| **Dashboard & Analytics** | Real-time financial summaries, charts, and KPI tracking |
| **Cash & Bank Management** | Cash registers, bank accounts, and transaction tracking |
| **Invoicing & Quotes** | Invoice creation, PDF export, and quote management |
| **Cheque & Promissory Notes** | Full cheque lifecycle tracking with event history and ledger entries |
| **Current Accounts** | Customer and supplier account balances with transaction history |
| **Recurring Transactions** | Automated recurring payment and income scheduling |
| **Budgets** | Budget planning and tracking against actuals |

### Commercial Operations
| Module | Description |
|--------|-------------|
| **Sales Orders** | Multi-product sales order processing |
| **Purchase Orders** | Supplier purchase order management |
| **Products & Services** | Product catalog with variants, attributes, and categories |
| **Stock Management** | Warehouse locations, stock movements, lot tracking, and alerts |
| **Customers & Suppliers** | Contact management with detailed profiles |

### Platform Features
| Module | Description |
|--------|-------------|
| **Multi-Branch** | Branch management with inter-branch stock transfers |
| **RBAC Authorization** | 5 roles (Admin, Manager, Cashier, Sales, Readonly) with 25+ granular permissions |
| **Multi-Tenant** | Row-level security ensuring complete data isolation |
| **Exchange Rates** | Real-time currency exchange rate integration |
| **Tags & Categories** | Flexible tagging and categorization system |
| **PDF Export** | Document generation for invoices and reports |

## Tech Stack

### Backend
| Technology | Purpose |
|-----------|---------|
| **Java 21** | Runtime |
| **Spring Boot 3.5** | Application framework |
| **Spring Security + JWT** | Authentication & authorization |
| **Spring Data JPA + Hibernate** | ORM & data access |
| **PostgreSQL** | Primary database |
| **Flyway** | Database migration management |
| **Redis** | Caching & session management |
| **ActiveMQ Artemis** | Async message processing |
| **Spring Actuator + Prometheus** | Monitoring & metrics |

### Frontend
| Technology | Purpose |
|-----------|---------|
| **React 18** | UI framework |
| **TypeScript** | Type safety |
| **Vite** | Build tool & dev server |
| **TanStack Query v5** | Server state management |
| **Tailwind CSS** | Utility-first styling |
| **shadcn/ui (Radix UI)** | Component library |
| **React Hook Form + Zod** | Form handling & validation |
| **Chart.js + Recharts** | Data visualization |
| **Framer Motion** | Animations |

### Mobile (Planned)
| Technology | Purpose |
|-----------|---------|
| **React Native** | Cross-platform mobile |
| **Expo** | Development & deployment |

### Infrastructure
| Technology | Purpose |
|-----------|---------|
| **AWS EC2 (Ubuntu 24.04)** | Cloud hosting |
| **Nginx** | Reverse proxy & static file serving |
| **Systemd** | Process management |
| **GitHub Actions** | CI/CD pipeline |

## Architecture

The application follows a modular monolith architecture with clear domain boundaries, designed for eventual migration to microservices if needed.

```
┌─────────────────────────────────────────────────────┐
│                    Nginx (:7093)                     │
│              Reverse Proxy + Static Files            │
├────────────────────────┬────────────────────────────┤
│    React SPA (Vite)    │   Spring Boot API (:8093)  │
│   /assets, /index.html │   /api/**                  │
├────────────────────────┴────────────────────────────┤
│                  Spring Security                     │
│            JWT + RBAC + Rate Limiting                │
├─────────────────────────────────────────────────────┤
│                 Business Modules                     │
│  ┌─────────┐ ┌──────┐ ┌───────┐ ┌──────────────┐   │
│  │  Auth   │ │Sales │ │Stock  │ │ Cash & Bank  │   │
│  ├─────────┤ ├──────┤ ├───────┤ ├──────────────┤   │
│  │Invoice  │ │Check │ │Branch │ │Current Acct  │   │
│  ├─────────┤ ├──────┤ ├───────┤ ├──────────────┤   │
│  │Purchase │ │Report│ │Config │ │  Shared      │   │
│  └─────────┘ └──────┘ └───────┘ └──────────────┘   │
├─────────────────────────────────────────────────────┤
│            PostgreSQL + Redis + ActiveMQ             │
│         Row-Level Security (Tenant Isolation)        │
└─────────────────────────────────────────────────────┘
```

For a detailed architecture breakdown, see [ARCHITECTURE.md](./ARCHITECTURE.md).

## Screenshots

> Screenshots will be added to the `/docs` directory.

## RBAC Model

The platform implements a comprehensive role-based access control system:

| Role | Scope |
|------|-------|
| **ADMIN** | Full system access including user management |
| **MANAGER** | All business operations except user management |
| **CASHIER** | Cash registers, bank transactions, cheque operations |
| **SALES** | Sales orders, customer management, invoicing |
| **READONLY** | View-only access to all modules |

Permissions are enforced at both the API level (`@PreAuthorize`) and the UI level (`ProtectedRoute` + `useAuth()` hooks).

## Database Schema

The database schema is managed through **24 Flyway migrations**, covering:

- User authentication & RBAC (users, roles, permissions)
- Product catalog (products, variants, attributes, categories)
- Stock management (balances, movements, warehouse locations, lots)
- Financial operations (invoices, quotes, cheques, ledger entries)
- Commercial operations (sales orders, purchase orders)
- Current accounts (customers, suppliers, transactions)
- Cash & bank management (cash registers, bank accounts)
- Multi-branch support (branches, branch users, transfers)
- Exchange rates, tags, and system configuration

## Project Structure

```
on-muhasebe/
├── backend-ekrem/              # Spring Boot backend
│   ├── src/main/java/.../
│   │   ├── auth/               # JWT, Security, User management
│   │   ├── branch/             # Branch operations
│   │   ├── cashbank/           # Cash & bank transactions
│   │   ├── chequepromissory/   # Cheque lifecycle management
│   │   ├── currentaccount/     # Customer/supplier accounts
│   │   ├── invoice/            # Invoicing & quotes
│   │   ├── purchasing/         # Purchase orders
│   │   ├── sales/              # Sales orders
│   │   ├── stock/              # Inventory management
│   │   ├── reporting/          # Reports & analytics
│   │   ├── shared/             # Base entities, exceptions, utilities
│   │   └── config/             # App configuration
│   └── src/main/resources/
│       └── db/migration/       # 24 Flyway migrations
│
├── client/                     # React frontend
│   └── src/
│       ├── components/         # Reusable UI components
│       ├── pages/              # 40+ page components
│       ├── hooks/              # Custom React hooks
│       ├── lib/                # Utilities & API client
│       └── types/              # TypeScript definitions
│
├── .github/workflows/          # CI/CD pipelines
└── docs/                       # Documentation & diagrams
```

## License

This project is proprietary software. All rights reserved.

## Contact

**Yavuzhan Kursun**
GitHub: [@yavuzhankursun](https://github.com/yavuzhankursun)
