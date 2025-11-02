# eShopMicroServices — High Level Architecture & Service Responsibilities

Summary
- eShopMicroServices is a containerized .NET microservices suite composed of four primary backend services (Basket, Discount, Order, Catalog).  
- Deployed on AKS using Flux GitOps and Azure infrastructure provisioned via Terraform (CICD-Templates + terraform-modules).  
- CI builds images, pushes to ACR, and updates the GitOps repo; Flux reconciles cluster state.

Quick repo pointers
- Services code: https://github.com/RijoyP/eShopMicroservices
- GitOps manifests: https://github.com/RijoyP/GitOps
- Terraform modules: https://github.com/RijoyP/terraform-modules
- CICD templates & generator: https://github.com/RijoyP/CICD-Templates 
- Helm boilerplate chart: https://github.com/RijoyP/helm-templates

# eShop Microservices - High Level Design

## 📋 Table of Contents
- [Overview](#overview)
- [Architecture Diagram](#architecture-diagram)
- [Backend Services](#backend-services)
- [Infrastructure & Deployment](#infrastructure--deployment)
- [CI/CD Pipeline](#cicd-pipeline)
- [GitOps with FluxCD](#gitops-with-fluxcd)
- [Observability Stack](#observability-stack)
- [Repository Links](#repository-links)

---

## 🎯 Overview

eShop Microservices is a production-grade distributed e-commerce system demonstrating modern cloud-native patterns and practices.

**Technology Stack:**
- **.NET 8** - Backend microservices
- **Azure Application Gateway** - API Gateway with Azure AD authentication
- **Docker & AKS** - Containerization and orchestration
- **FluxCD** - GitOps continuous delivery
- **RabbitMQ** - Event-driven messaging
- **Terraform** - Infrastructure as Code
- **Azure DevOps** - CI/CD automation

---

## 🏗️ Architecture Diagram

### High-Level System Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          CLIENT LAYER                                    │
│                     ┌──────────────────────┐                            │
│                     │   Shopping.Web       │                            │
│                     │   (ASP.NET Core)     │                            │
│                     └──────────┬───────────┘                            │
└────────────────────────────────┼────────────────────────────────────────┘
                                 │ HTTPS
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    API GATEWAY LAYER                                     │
│         ┌────────────────────────────────────────────────┐             │
│         │    Azure Application Gateway                   │             │
│         │  + Web Application Firewall (WAF)              │             │
│         │  + Azure AD Authentication                     │             │
│         │  + SSL/TLS Termination                         │             │
│         │  + URL Routing & Path-based Routing            │             │
│         │  + Load Balancing                              │             │
│         └──────┬─────────┬─────────┬─────────────────────┘             │
└────────────────┼─────────┼─────────┼─────────────────────────────────────┘
                 │         │         │
        ┌────────┼─────────┼─────────┼─────────────┐
        │        │         │         │             │
        ▼        ▼         ▼         ▼             │
┌──────────────────────────────────────────────────┼────────────────────┐
│  MICROSERVICES LAYER                             │                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────▼──────┐        │
│  │ Catalog  │  │ Basket   │  │ Discount │  │   Ordering    │        │
│  │   API    │  │   API    │  │   API    │  │     API       │        │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────────┬──────┘        │
│       │             │              │                  │                │
│       ▼             ▼              ▼                  ▼                │
│  ┌────────┐    ┌────────┐    ┌────────┐        ┌──────────┐         │
│  │Postgre │    │Postgre │    │ SQLite │        │   SQL    │         │
│  │  SQL   │    │SQL+Redis    │        │        │  Server  │         │
│  └────────┘    └────────┘    └────────┘        └──────────┘         │
└─────────────────────────────┬──────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      MESSAGING LAYER                                     │
│                  ┌────────────────────────┐                             │
│                  │      RabbitMQ          │                             │
│                  │   Event-Driven Bus     │                             │
│                  └────────────────────────┘                             │
└─────────────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    AZURE KUBERNETES SERVICE (AKS)                        │
│  ┌────────────────────────────────────────────────────────────┐        │
│  │  Ingress Controller │ Service Mesh │ Azure Key Vault       │        │
│  └────────────────────────────────────────────────────────────┘        │
└─────────────────────────┬───────────────────────────────────────────────┘
                          │
                          │ GitOps Sync
                          ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         FLUXCD GITOPS                                    │
│  ┌──────────────────────────────────────────────────────────┐          │
│  │  Source Controller → Kustomize Controller → Apply         │          │
│  └────────────────┬─────────────────────────────────────────┘          │
│                   │ Git Sync (1-5 min)                                  │
│  ┌────────────────▼─────────────────────────────────────────┐          │
│  │  GitOps Repository                                        │          │
│  │    /infrastructure  → Platform components                 │          │
│  │    /apps           → Application manifests                │          │
│  └──────────────────────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                    OBSERVABILITY STACK                                   │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌──────────┐           │
│  │  Logging  │  │  Tracing  │  │Monitoring │  │ Alerting │           │
│  │ (ELK/PLG) │  │  (Jaeger) │  │(Prometheus│  │ (Grafana)│           │
│  └───────────┘  └───────────┘  └───────────┘  └──────────┘           │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                         CI/CD PIPELINE                                   │
│  Code Push → Build & Test → Docker Build → ACR Push → Update GitOps    │
│                            → FluxCD Auto Deploy                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 🔧 Backend Services

### Service Overview

| Service | Database | Purpose | Key Features |
|---------|----------|---------|--------------|
| **Catalog API** | PostgreSQL | Product management | Product CRUD, categories, inventory, search |
| **Basket API** | PostgreSQL + Redis | Shopping cart | Cart operations, session management, checkout |
| **Discount API** | SQLite | Promotions | Coupon validation, discount calculation |
| **Ordering API** | SQL Server | Order processing | DDD implementation, order lifecycle, payments |

**Source Code**: [eShopMicroservices/Services](https://github.com/RijoyP/eShopMicroservices/tree/main/Services)

---

### 1. Catalog API

**Purpose**: Product catalog and inventory management

**Tech Stack**: ASP.NET Core (.NET 8) + PostgreSQL + Redis

**Key Responsibilities:**
- Product and category management
- Real-time inventory tracking
- Search and filtering
- Image management
- Integration with Azure Blob Storage

**Why PostgreSQL?** Complex queries, JSON support, full-text search, ACID compliance for inventory

---

### 2. Basket API

**Purpose**: Shopping cart and session management

**Tech Stack**: ASP.NET Core (.NET 8) + PostgreSQL + Redis

**Key Responsibilities:**
- Add/remove cart items
- Calculate totals
- Apply discounts
- Session persistence
- Checkout coordination

**Dual Database Strategy:**
- **PostgreSQL**: Persistent cart history and audit trails
- **Redis**: High-speed active cart operations and sessions

**Event Publishing**: Publishes `BasketCheckedOut` event to trigger order creation

---

### 3. Discount API

**Purpose**: Coupon and promotion management

**Tech Stack**: ASP.NET Core (.NET 8) + SQLite + Redis

**Key Responsibilities:**
- Coupon code validation
- Discount calculation engine
- Promotion rules management
- Time-bound offers

**Why SQLite?** Lightweight, low data volume, simplified deployment for rules-based data

---

### 4. Ordering API - Domain-Driven Design

**Purpose**: Complete order lifecycle management

**Tech Stack**: ASP.NET Core (.NET 8) + SQL Server + RabbitMQ

**Key Responsibilities:**
- Order creation and validation
- Payment coordination
- Order status workflow
- Fulfillment tracking

**Why SQL Server?** Enterprise transactions, audit capabilities, reporting features, financial data consistency

#### Domain-Driven Design Implementation

**Bounded Context: Ordering**

Core domain concepts:
- **Order** (Aggregate Root) - Enforces business rules, controls transactions
- **OrderItem** (Entity) - Line items within orders
- **Address** (Value Object) - Immutable shipping/billing address
- **Payment Info** (Value Object) - Payment details
- **OrderStatus** (Enumeration) - Order state machine

**Domain Events Published:**
- `OrderCreated` → Triggers inventory reservation
- `OrderPaid` → Initiates fulfillment
- `OrderStatusChanged` → Updates tracking systems
- `OrderShipped` → Notifies customer
- `OrderDelivered` → Closes order

#### Context Mapping

```
┌─────────────────────────────────────────────────────────┐
│                  CONTEXT MAP                             │
│                                                          │
│  Catalog Context                                         │
│       │                                                  │
│       │ (ACL)                                           │
│       ▼                                                  │
│  Ordering Context ◄──(Customer/Supplier)── Basket       │
│       │                                                  │
│       │ (Partnership)                                   │
│       ▼                                                  │
│  Payment Context                                         │
│                                                          │
│  Discount Context ──(Conformist)──► Ordering            │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

**Integration Patterns:**

**Catalog → Ordering (Anti-Corruption Layer)**
- Ordering translates Catalog models to its own domain objects
- Prevents breaking changes from cascading

**Basket → Ordering (Event-Driven)**
- Basket publishes events, Ordering subscribes
- Loose coupling through RabbitMQ

**Ordering → Payment (Synchronous Partnership)**
- Direct API calls with shared payment concepts
- Transactional consistency maintained

**Discount → Ordering (Conformist)**
- Ordering calls Discount API for price calculations
- Read-only relationship with graceful degradation

---

## ☁️ Infrastructure & Deployment

### Terraform Infrastructure

**Repository**: [terraform-modules](https://github.com/RijoyP/terraform-modules)

**Key Modules:**
- **AKS**: Kubernetes cluster with node pools, autoscaling, Azure AD integration
- **ACR**: Container registry with geo-replication and vulnerability scanning
- **Networking**: VNet, subnets, NSGs, Application Gateway
- **Databases**: PostgreSQL, SQL Server, Redis managed services
- **Monitoring**: Log Analytics, Application Insights
- **Security**: Key Vault, managed identities, RBAC

**Deployment Stages:**

```
Terraform Validate → Plan → Apply (Dev) → Apply (Staging) → Apply (Prod)
                                ↓              ↓               ↓
                          Manual Approval Required for Prod
```

### Platform Components Deployment

**Repository**: [GitOps/infrastructure](https://github.com/RijoyP/GitOps/tree/main/infrastructure)

**Deployment Order:**

```
1. FluxCD System          → GitOps operator
2. Ingress NGINX          → External traffic routing
3. Cert Manager           → TLS certificate automation
4. RabbitMQ              → Message broker
5. Logging Stack          → Fluentd + Elasticsearch + Kibana (or PLG)
6. Monitoring Stack       → Prometheus + Grafana + Alertmanager
7. Tracing               → Jaeger with OpenTelemetry
```

---

## 🚀 CI/CD Pipeline

### Pipeline Architecture

**Repository**: [CICD-Templates](https://github.com/RijoyP/CICD-Templates)

### Application Pipeline Flow

```
┌──────────────┐
│  Git Push    │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Restore &   │  → dotnet restore
│  Build       │  → dotnet build
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Unit &      │  → Run xUnit tests
│  Integration │  → Code coverage check
│  Tests       │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Security    │  → SAST scanning
│  Scanning    │  → Dependency check
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Docker      │  → Build image
│  Build       │  → Scan with Trivy
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Push to ACR │  → Tag with version & SHA
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Update      │  → Update Kustomize overlay
│  GitOps Repo │  → Commit new image tag
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  FluxCD      │  → Auto-sync to AKS
│  Deployment  │
└──────────────┘
```

**Pipeline Files**: [eShopMicroservices/.pipeline](https://github.com/RijoyP/eShopMicroservices/tree/main/.pipeline)

### Automated Profile Generation

**Script**: [generate-profiles.ps1](https://github.com/RijoyP/CICD-Templates/generate-profiles.ps1)

Automatically generates pipeline configurations for each microservice and environment:

- Scans service directory structure
- Creates dev/staging/prod pipeline files
- Injects service-specific variables
- Validates generated YAML
- Reduces setup time from hours to minutes

---

## 🔄 GitOps with FluxCD

### Repository Structure

**Repository**: [GitOps](https://github.com/RijoyP/GitOps)

```
GitOps/
├── infrastructure/              # Platform components
│   ├── base/
│   ├── overlays/
│   │   ├── dev/
│   │   ├── staging/
│   │   └── production/
│   ├── logging/                # Fluentd, Elasticsearch, Kibana
│   ├── tracing/                # Jaeger
│   ├── monitoring/             # Prometheus, Grafana
│   └── messaging/              # RabbitMQ
│
└── apps/                       # Microservice deployments
    ├── base/
    │   ├── catalog-api/
    │   ├── basket-api/
    │   ├── discount-api/
    │   └── ordering-api/
    └── overlays/
        ├── dev/
        ├── staging/
        └── production/
```

### FluxCD Workflow

```
┌──────────────────────────────────────────────────────┐
│                  FluxCD Reconciliation Loop          │
│                                                      │
│  Git Commit (New Image Tag)                         │
│         ↓                                           │
│  Source Controller (Detects Change)                 │
│         ↓                                           │
│  Kustomize Controller (Builds Manifests)            │
│         ↓                                           │
│  Apply to AKS Cluster                               │
│         ↓                                           │
│  Health Check & Validation                          │
│         ↓                                           │
│  Notification (Teams/Slack)                         │
│                                                      │
│  ◄────── Continuous Reconciliation (1-5 min) ───────┤
└──────────────────────────────────────────────────────┘
```

**Key Features:**
- **Drift Detection**: Auto-corrects manual changes back to Git state
- **Progressive Delivery**: Canary deployments with Flagger
- **Automated Rollback**: Reverts on health check failures
- **Multi-Environment**: Separate overlays for dev/staging/prod

### Helm Integration

**Repository**: [helm-templates](https://github.com/RijoyP/helm-templates)

Boilerplate Helm chart for consistent microservice deployments:
- Deployment, Service, Ingress templates
- ConfigMap and Secret management
- HPA (Horizontal Pod Autoscaling)
- ServiceMonitor for Prometheus
- PodDisruptionBudget

---

## 📊 Observability Stack

### Logging

**Stack Options:**

**ELK Stack**: Fluentd → Elasticsearch → Kibana

**PLG Stack**: Promtail → Loki → Grafana

**Capabilities:**
- Centralized log aggregation from all services
- Full-text search and filtering
- Pre-built dashboards for error tracking
- Log retention policies (dev: 7d, staging: 30d, prod: 90d)
- Correlation with trace IDs

**Deployment**: [GitOps/infrastructure/logging](https://github.com/RijoyP/GitOps/tree/main/infrastructure/logging)

---

### Tracing

**Component**: Jaeger with OpenTelemetry

**Features:**
- Distributed trace tracking across all microservices
- Automatic instrumentation for HTTP, SQL, RabbitMQ
- Latency analysis and bottleneck identification
- Service dependency mapping
- Error tracing and root cause analysis

**Deployment**: [GitOps/infrastructure/tracing](https://github.com/RijoyP/GitOps/tree/main/infrastructure/tracing)

---

### Monitoring

**Stack**: Prometheus + Grafana + Alertmanager

**Metrics Collected:**
- Application metrics (request rates, latencies, errors)
- Infrastructure metrics (CPU, memory, disk, network)
- Database metrics (connections, query performance)
- RabbitMQ metrics (queue depth, message rates)
- Custom business metrics

**Features:**
- Real-time dashboards
- Alert rules for SLOs
- Multi-environment views
- Historical trend analysis

**Deployment**: [GitOps/infrastructure/monitoring](https://github.com/RijoyP/GitOps/tree/main/infrastructure/monitoring)

---

### Messaging

**Component**: RabbitMQ

**Features:**
- Event-driven communication between services
- Durable queues for reliable message delivery
- Dead letter queues for failed messages
- Message tracing and monitoring
- High availability clustering

**Deployment**: [GitOps/infrastructure/messaging](https://github.com/RijoyP/GitOps/tree/main/infrastructure/messaging)

---

## 🔗 Repository Links

| Repository | Purpose |
|------------|---------|
| [eShopMicroservices](https://github.com/RijoyP/eShopMicroservices) | Microservices source code |
| [GitOps](https://github.com/RijoyP/GitOps) | Kubernetes manifests and FluxCD configs |
| [terraform-modules](https://github.com/RijoyP/terraform-modules) | Infrastructure as Code modules |
| [CICD-Templates](https://github.com/RijoyP/CICD-Templates) | Reusable pipeline templates |
| [helm-templates](https://github.com/RijoyP/helm-templates) | Helm chart boilerplate |

---

## 🎯 Key Architectural Patterns

- **Microservices Architecture**: Independent, loosely coupled services
- **Event-Driven Architecture**: Asynchronous communication via RabbitMQ
- **API Gateway Pattern**: Azure Application Gateway with Azure AD authentication
- **Database per Service**: Each service owns its data
- **CQRS**: Command-Query separation in Ordering service
- **Domain-Driven Design**: Rich domain model in Ordering service
- **GitOps**: Declarative infrastructure and deployments
- **Infrastructure as Code**: Terraform for all Azure resources

---

## 📈 System Characteristics

**Scalability**: Horizontal scaling through Kubernetes HPA, independent service scaling

**Resilience**: Circuit breakers, retry policies, health checks, automated recovery

**Security**: Azure AD authentication, managed identities, Key Vault integration, network policies

**Observability**: Comprehensive logging, tracing, and monitoring across all layers

**Automation**: Fully automated CI/CD pipelines, GitOps-driven deployments

**Compliance**: Audit trails, encryption at rest and in transit, RBAC

---

## 🚀 Deployment Summary

1. **Infrastructure**: Terraform provisions Azure resources (AKS, ACR, databases)
2. **Platform**: FluxCD deploys infrastructure components (logging, monitoring, messaging)
3. **Applications**: CI/CD builds images, updates GitOps repo, FluxCD auto-deploys to AKS
4. **Monitoring**: Observability stack provides full system visibility

**All deployments are automated, version-controlled, and auditable through Git.**

---

## 📸 How to Use This README

1. **Copy the entire content** (Ctrl+A, Ctrl+C)
2. **Create a new README.md file** in your repository root
3. **Paste the content** (Ctrl+V)
4. **Commit and push** to your repository
5. The ASCII diagram will render properly in GitHub, GitLab, and other markdown viewers

---

**Built with ❤️ using .NET 8, Azure, Kubernetes, and modern DevOps practices**


