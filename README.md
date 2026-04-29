# eShopMicroServices — High Level Architecture & Service Responsibilities

Summary
- eShopMicroServices is a containerized **.NET** & **Java** microservices suite composed of four primary backend services (Customer, Catalog, Basket, Discount, Order and Payment).  
- RAG + ChatGPT Integration: Leveraging **Python**, **Azure OpenAI**, and **Azure Cognitive Search**, the platform supports natural-language queries over product data and    PDF manuals.   Users can ask about features, specifications, pricing and user manual answers using semantic vector search and hybrid search.
- Deployed on **AKS** using **Flux** **GitOps** and Azure infrastructure provisioned via **Terraform** (CICD-Templates + terraform-modules).  
- CI builds images, pushes to **ACR**, and updates the GitOps repo; Flux reconciles cluster state.

## 🔗 Repository Links

| Repository | Purpose |
|------------|---------|
| [eShopMicroservices](https://github.com/RijoyP/eShopMicroservices) | Microservices source code |
| [GitOps](https://github.com/RijoyP/GitOps) | Kubernetes manifests and FluxCD configs |
| [terraform-modules](https://github.com/RijoyP/terraform-modules) | Infrastructure as Code modules |
| [CICD-Templates](https://github.com/RijoyP/CICD-Templates) | Reusable pipeline templates |
| [helm-templates](https://github.com/RijoyP/helm-templates) | Helm chart boilerplate |

## 🔗 Backend Urls Links

## Removed Azure Resources due to the high cost, below url wont work

| Backend | Urls |
|------------|---------|
| Front End React (React Typescript) | https://eshopreact.4.187.236.253.nip.io/ | 
| Catatlog API (ASP.NET Core 8.0) | https://catalog.4.187.236.253.nip.io/swagger/index.html | 
| Discount API (ASP.NET Core 8.0) | https://discount.4.187.236.253.nip.io/swagger/index.html | 
| Basket API (ASP.NET Core 8.0) | https://basket.4.187.236.253.nip.io/swagger/index.html |
| Order API (ASP.NET Core 8.0) | https://order.4.187.236.253.nip.io/swagger/index.html |
| Payment API (ASP.NET Core 9.0) | https://payment.4.187.236.253.nip.io/swagger/index.html |
| Customer API (Java : Maven) | https://customer.4.187.236.253.nip.io/swagger-ui/index.html |
| Chat API (Python) | https://chatgptrag.4.187.236.253.nip.io/docs |

## 🔗 Infra Monitoring / Logging / Messaging Urls Links

| Resource   | Url |
|------------|---------|
| Zipkin     | http://zipkin.135.235.243.70.nip.io/ | 
| Rabbit MQ  | http://rabbitmq.135.235.243.70.nip.io/ | 
| Grafana    | http://grafana.135.235.243.70.nip.io/ | 
| Prometheus | http://prometheus.135.235.243.70.nip.io/ | 
| Jaeger   | http://jaeger.135.235.243.70.nip.io/ | 
| Kibana   | http://kibana.135.235.243.70.nip.io/ | 

# eShop Microservices - High Level Design

## 🎯 Overview

eShop Microservices is a production-grade distributed e-commerce system demonstrating modern cloud-native patterns and practices.

**Technology Stack:**
- **.NET 9** - Backend microservices
- **Azure Application Gateway** - API Gateway with Azure AD authentication
- **Docker & AKS** - Containerization and orchestration
- **FluxCD** - GitOps continuous delivery
- **RabbitMQ** - Event-driven messaging
- **Terraform** - Infrastructure as Code
- **Azure DevOps** - CI/CD automation

---

## 🏗️ Architecture Diagram

### High-Level System Architecture

![Project Diagram](https://github.com/RijoyP/RijoyP/blob/main/assets/eshopmicroserviceslatest.png)

---

## 🔧 Backend Services

### Architecture Layers

Order microservices follow Clean Architecture principles with clear separation of concerns:

```
┌──────────────────────────────────────────────────────┐
│         API Layer (Carter - Minimal APIs)            │
│  • HTTP Endpoints                                    │
│  • Request/Response DTOs                             │
│  • FluentValidation for input validation             │
└────────────────────┬─────────────────────────────────┘
                     ▼
┌─────────────────────────────────────────────────────┐
│       Application Layer (MediatR - CQRS)            │
│  ┌──────────────┐        ┌──────────────┐           │
│  │  Commands    │        │   Queries    │           │
│  │(Write Model) │        │(Read Model)  │           │
│  └──────┬───────┘        └──────┬───────┘           │
│         │                       │                   │
│         ▼                       ▼                   │
│  Command Handlers        Query Handlers             │
│  + Business Logic        + Caching Logic            │
└────────────────────┬────────────────────────────────┘
                     ▼
┌─────────────────────────────────────────────────────┐
│            Domain Layer (DDD)                       │
│  • Aggregates & Entities                            │ 
│  • Value Objects                                    │
│  • Domain Events                                    │
│  • Business Rules                                   │
└────────────────────┬────────────────────────────────┘
                     ▼
┌─────────────────────────────────────────────────────┐
│         Infrastructure Layer                        │
│  • Entity Framework Core (Traditional ORM)          │
│  • Marten (Event Sourcing for Ordering)             │
│  • PostgreSQL / SQL Server / SQLite                 │
│  • Redis (Caching)                                  │
│  • RabbitMQ (Messaging)                             │
│  • gRPC (Inter-service communication)               │
└─────────────────────────────────────────────────────┘

        Cross-Cutting Concerns (Building Blocks)
┌──────────────────────────────────────────────────────┐
│  OpenTelemetry | Serilog | Polly | FluentValidation  │ 
└──────────────────────────────────────────────────────┘
```

### Service Overview

| Service | Database | Purpose | Key Features |
|---------|----------|---------|--------------|
| **Catalog API** | PostgreSQL | Product management | Product CRUD, categories, inventory, search |
| **Catalog API GRPC** | PostgreSQL | Internal pod communication | Get Product |
| **Basket API** | PostgreSQL + Redis | Shopping cart | Cart operations, session management and Checkout |
| **Discount API GRPC** | SQLite | Internal pod communication | Coupon validation, discount calculation |
| **Discount API** |    | UI Call | Create, Update and Get Coupon |
| **Ordering API** | SQL Server | Order processing | DDD implementation, order lifecycle, payments |
| **Inventory API** | PostgreSQL | Inventory Management | Reserve Stock, Confirm Stock and Release Stock |
| **Payment API** | PostgreSQL | Payment Management | Payment |

**Source Code**: [eShopMicroservices/Services](https://github.com/RijoyP/eShopMicroservices/tree/main/Services)

---

**Admin Azure AD Authentication**

```
┌────────────────────────────────────────────────────────────────────────────┐
│                         Azure AD B2C Tenant                                │
│                                                                            │
│  ┌──────────────┐  ┌───────────────┐  ┌───────────────┐ ┌──────────────┐   │
│  │ Catalog API  │  │  Discount API │  │ Inventory API │ │    React     │   │
│  │  (Backend)   │  │   (Backend)   │  │   (Backend)   │ │  (Frontend)  │   │
│  │ Roles:       │  │ Roles:        │  │ Roles:        │ │              │   │
│  │ CatalogWrite │  │ DiscountWrite │  │InventoryWrite │ │ Permissions: │   │
│  │              │  │               │  │               │ │ - Catalog    │   │
│  │ Scopes:      │  │ Scopes:       │  │ Scopes:       │ │ - Discount   │   │
│  │ Catalog.Write│  │ Discount.Write│  │Inventory.Write│ │              │   │
│  └──────────────┘  └───────────────┘  └───────────────┘ └──────────────┘   │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────┐                     │
│  │ Catalog User │  │ Discount User│  │ Inventory User│                     │
│  │              │  │              │  │               │                     │
│  │ Assigned:    │  │ Assigned:    │  │ Assigned:     │                     │
│  │ CatalogWrite │  │ DiscountWrite│  │ InventoryWrite│                     │ 
│  └──────────────┘  └──────────────┘  └───────────────┘                     │
└────────────────────────────────────────────────────────────────────────────┘

```

**Admin Authentication Flow Diagram**

```

┌──────────┐                                                    ┌──────────┐
│          │  1. User clicks "Login"                            │          │
│          │ ─────────────────────────────────────────────────> │          │
│          │                                                    │          │
│          │  2. Redirect to Azure AD B2C login page            │  Azure   │
│  React   │ <───────────────────────────────────────────────── │  AD B2C  │
│   App    │                                                    │          │
│          │  3. User enters credentials                        │          │
│          │ ─────────────────────────────────────────────────> │          │
│          │                                                    │          │
│          │  4. ID Token (user identity)                       │          │
│          │ <───────────────────────────────────────────────── │          │
└────┬─────┘                                                    └──────────┘
     │
     │ User is now logged in
     │
     │ 5. User attempts to create product
     │
     ▼
┌──────────┐
│  React   │  6. Request Access Token
│   App    │     Scope: api://catalog-api/Catalog.Write
│          │     Account: cataloguser@domain.com
└────┬─────┘
     │
     │ 7. acquireTokenSilent()
     │
     ▼
┌──────────┐
│  Azure   │  8. Validate user + scope
│  AD B2C  │     Check: Does user have CatalogWrite role?
│          │     Yes → Issue Access Token
└────┬─────┘
     │
     │ 9. Access Token
     │    {
     │      "aud": "api://catalog-api",
     │      "roles": ["CatalogWrite"],
     │      "scp": "Catalog.Write"
     │    }
     │
     ▼
┌──────────┐
│  React   │  10. POST /api/catalog/create
│   App    │      Authorization: Bearer <access-token>
│          │      Body: { name: "Product", price: 99.99 }
└────┬─────┘
     │
     │
     ▼
┌──────────┐
│ Catalog  │  11. Validate Token
│   API    │      • Verify signature
│          │      • Check expiration
│          │      • Validate audience
│          │      • Check role: CatalogWrite
│          │
│          │  12. Execute: Create Product
│          │
│          │  13. Response: 200 OK
│          │      { "id": "123", "name": "Product" }
└──────────┘

```

---

🛒 **E-commerce Microservices Architecture**

This repository demonstrates an event-driven e-commerce system using Saga orchestration and the Transactional Outbox pattern 📦🐇 for reliable messaging across microservices.

All inter-service communication is asynchronous using RabbitMQ 🐇, ensuring loose coupling and eventual consistency.

🌐 **Architecture Overview**

👨‍💻 **Basket API** 🛍️
On checkout, writes a BasketCheckoutEvent to its outbox 📦.

📦 **Order API** 🏷️

Consumes BasketCheckoutEvent from RabbitMQ 🐇

Creates the order 📝

Writes OrderCreatedEvent to its outbox 📦

📦 **Inventory API** 📦

Consumes OrderCreatedEvent from RabbitMQ 🐇

Checks and reserves stock:

❌ Stock unavailable → Publishes InventoryFailedEvent 📦 → RabbitMQ 🐇 → Order API (Order canceled 🛑)

✅ Stock available → Publishes InventoryReservedEvent 📦 → RabbitMQ 🐇 → Payment API 💳

💳 **Payment API** 💳

Consumes InventoryReservedEvent from RabbitMQ 🐇

**Processes payment**:

✅ Success → Publishes PaymentCompletedEvent 📦 → Inventory API 📦

❌ Failure → Publishes PaymentFailedEvent 📦 → Inventory API (Release stock 📦)

📦 **Inventory API (after payment)**

Consumes PaymentCompletedEvent → Confirms inventory → Publishes InventoryConfirmedEvent 📦 → Order API (Order completed ✅)

Consumes PaymentFailedEvent → Releases inventory → Publishes InventoryFailedEvent 📦 → Order API (Order canceled 🛑)

**User Checkout** 🛒

```

                                 ┌───────────────────┐
                                 │    User Checkout  │
                                 └──────────┬────────┘
                                            │
                                            ▼
                               ┌─────────────────────────┐
                               │    BasketCheckoutEvent  │
                               │   → Outbox  → RabbitMQ  │ (Async)
                               └─────────────┬───────────┘
                                             │
                                             ▼
                             ┌────────────────────────────┐
                             │     Order API Consumes     │
                             │       Creates Order        │
                             │     → OrderCreatedEvent    │
                             │     → Outbox  → RabbitMQ   │ (Async)
                             └───────────────┬────────────┘
                                             │  
                                             ▼
                               ┌────────────────────────┐
                               │ Inventory API Consumes │
                               │       Reserves Stock   │
                               └───────────┬────────────┘
                                   ┌───────┴──────────┐
                                   │                  │ 
                         ✅ Success▼                  ▼ ❌ Failure
                      ┌───────────────────────┐  ┌──────────────────────────┐
                      │ InventoryReservedEvent│  │   InventoryFailedEvent   │
                      │ → Outbox  → RabbitMQ  │  │ → Outbox  → RabbitMQ     │
                      │     → Payment API     │  │ → Order API Cancels Order│
                      └───────────┬───────────┘  └──────────────────────────┘
                                  │
                                  ▼
                     ┌────────────────────────┐
                     │ Payment API  processes │
                     └───────────┬────────────┘
                      ┌──────────┴──────────┐
            ✅ Success▼                     ▼ ❌ Failure
        ┌───────────────────────────┐  ┌────────────────────────┐
        │       CompletedEvent      │  │        FailedEvent     │
        │     → Outbox  → RabbitMQ  │  │ → Outbox  → RabbitMQ   │
        │     → Inventory API       │  │    → Inventory API     │
        │  → InventoryConfirmedEvent│  │ → InventoryFailedEvent │
        │  → Order API Completes    │  │ → Order API  cancels   │
        └───────────────────────────┘  └────────────────────────┘

```
🧩 **Key Features**

📦 **Transactional Outbox**
Ensures reliable event delivery across microservices.

🧵 **Saga Orchestration**
Coordinates complex transactions with success and failure paths (inventory release, order cancellation) 🛠️♻️.

💡 **Idempotent Handlers**
Supports retry scenarios without creating inconsistent state 🔄.

🗄️ **Database Choice**

Order API → SQL 🗄️

Inventory API → PostgreSQL 🗄️

Payment API → PostgreSQL 🗄️
Ensures each microservice persists its own data reliably.

🏗️ **Implementation Notes**

All services write events to Outbox instead of publishing directly.

A background OutboxPublisher service reads pending events and sends them to RabbitMQ 🐇.

All downstream services consume events from RabbitMQ and update their state accordingly.

Failure events (payment failed, inventory unavailable) trigger compensating actions automatically.

---

**Azure Cognitive Search + Azure OpenAI Integration**

**Overview**

This project implements a serverless ingestion platform using Azure Functions to process both:

Structured product data (event-driven ingestion)

Unstructured user manuals (document-based ingestion)

The ingested data is indexed into Azure Cognitive Search and enriched with embeddings from Azure OpenAI, enabling AI-powered search and Q&A using a Retrieval-Augmented Generation (RAG) pattern.

Ingestion Types
Ingestion Type	Trigger	Data Type
Product Ingestion	Azure Service Bus Queue	Structured JSON
User Manual Ingestion	Azure Blob Storage Upload	PDF / Documents

Both ingestion pipelines run inside the same Azure Function App, but as separate functions.

**📌 Features**

Create/update Cognitive Search index with vector search (HNSW)

Generate embeddings using Azure OpenAI

Upload product and PDF chunk documents into Azure Search

Perform pure vector search and hybrid search (keyword + vector)

Extract, chunk, embed, and index PDF manuals

**⚙️ Prerequisites**

Azure Subscription

Azure Cognitive Search (Basic tier or higher)

Azure OpenAI resource with Embedding deployment

Python 3.9+

**Pdf Data Source**

[IPhone16 User Manual](https://github.com/RijoyP/RijoyP/blob/main/assets/iphone16pro.pdf)

[Readme Document](https://github.com/RijoyP/RijoyP/blob/main/assets/gitreporeadme.pdf)

**AI-Powered Ingestion Platform**

Azure Functions · Azure Service Bus · Azure Storage · Azure Cognitive Search · Azure OpenAI

**High-Level Architecture**

```
            ┌────────────────────┐
            │   Product Source   │
            └─────────┬──────────┘
                      │
                      ▼
          ┌───────────────────────┐
          │   Azure Service Bus   │
          │   (product-created)   │
          └───────────┬───────────┘
                      │
                      ▼
      ┌────────────────────────────────┐
      │ Azure Function: product_ingest │
      └───────────────┬────────────────┘
                      │
                      ▼
            Azure Cognitive Search
                      │
                      ▼
                Azure OpenAI

```

```

           User Uploads Manual (PDF)
                     │
                     ▼
             Azure Blob Storage
                     │
                     ▼
      ┌───────────────────────────────┐
      │ Azure Function: manual_ingest │
      └──────────────┬────────────────┘
                     │
                     ▼
           Azure Cognitive Search
                     │
                Azure OpenAI
```

1️⃣ **Product Ingestion Flow (Service Bus → Azure Function)**
Purpose

Ingest structured product data asynchronously using an event-driven architecture.

**Step 1: Product Data Published to Service Bus**

Product events are sent to an Azure Service Bus Queue (product-created-queue).

```
Example Product Message
{
  "ProductId": "1001",
  "Name": "Wireless Headphones",
  "Description": "Noise cancelling wireless headphones",
  "Price": 149.99,
  "ImageFile": "headphones.png"
}

```

**Step 2: Azure Function Triggered**

The product_ingest Azure Function is triggered automatically when a new message arrives.

Responsibilities

Deserialize product message

Build searchable content

Generate vector embeddings using Azure OpenAI

Index product data into Azure Cognitive Search

Product Ingestion Function (Conceptual)
def main(msg: func.ServiceBusMessage):
    product = json.loads(msg.get_body().decode("utf-8"))

    content = f"{product['Name']} {product.get('Description', '')}"
    embedding = generate_embedding(content)

    document = {
        "id": f"prod_{product['ProductId']}",
        "content": content,
        "embedding": embedding
    }

    product_search_service.upload_documents([document])

Output

Product documents indexed into Cognitive Search

Searchable via filters, keyword search, and semantic AI queries

2️⃣ **User Manual Ingestion Flow (Blob Storage → Azure Function)**
Purpose

Ingest unstructured documents such as:

User manuals

PDFs

Technical documentation
**
Step 1: Manual Uploaded to Blob Storage**

Users upload manuals to a specific Azure Blob container (e.g., manuals).

```
manuals/
 └── iphone16-user-guide.pdf

```

**Step 2: Azure Function Triggered**

The manual_ingest Azure Function is triggered by a Blob Storage event.

Responsibilities

Read uploaded document

Extract text from PDF

Split text into chunks

Generate embeddings per chunk using Azure OpenAI

Index chunks into Azure Cognitive Search

Manual Ingestion Function (Conceptual)
def main(blob: bytes, name: str):
    text = extract_text_from_pdf(blob)

    documents = []
    for i, chunk in enumerate(chunk_text(text)):
        documents.append({
            "id": f"{name}_chunk_{i}",
            "content": chunk,
            "embedding": generate_embedding(chunk)
        })

    manual_search_service.upload_documents(documents)

Output

Manuals indexed as multiple searchable chunks

Enables semantic and contextual search across documents

3️⃣** Azure Cognitive Search Indexes**

Two separate indexes are used:

Index Name	Content
product-index	Product metadata & descriptions
manual-index	Chunked user manual content

Each index stores:

Content

Metadata

Vector embeddings

4️⃣** Azure OpenAI Integration (RAG)**

Azure OpenAI is used to:

Generate embeddings during ingestion

Answer user queries using retrieved documents

Example Query

“How do I reset the wireless headphones?”

Azure OpenAI retrieves relevant chunks from manual-index and generates a contextual answer.

Key Benefits

Single Azure Function App with multiple ingestion triggers

Event-driven and scalable

Supports both structured and unstructured data

AI-powered search and Q&A

Production-ready architecture

Reliability & Scalability

Azure Service Bus ensures message durability

Azure Functions auto-scale based on load

Blob-triggered ingestion handles large files

Dead-letter queues support retry and error handling

**Summary**

This solution demonstrates a complete AI ingestion pipeline where:

Products are ingested via Service Bus–triggered Azure Functions

User manuals are ingested via Blob-triggered Azure Functions

Data is indexed in Azure Cognitive Search

---

### 1. Catalog API

**Purpose**: Product catalog and inventory management

**Tech Stack**:
- **Backend:** ASP.NET Core (.NET 8)
- **Database:** PostgreSQL (with Marten for document/event storage)
- **Messaging:** Azure Service Bus (Product Data)
- **Blob Storage:** Azure Storage Account (for product manuals/files)
- **Background Processing:** Hosted Service (Outbox pattern)

**Key Responsibilities:**
- Product and category management
- Real-time inventory tracking
- Search and filtering

**Why PostgreSQL?** Complex queries, JSON support, full-text search, ACID compliance for inventory

#### Architecture Overview

**1. Unit of Work**
- Ensures all changes to the database (via Marten/PostgreSQL) are committed atomically.
- Used for transactional consistency across product and outbox message storage.

**2. Outbox Pattern**
- Domain events (e.g., `ProductCreated`) are stored in an Outbox table as part of the same transaction as business data.
- Guarantees that events are only published if the transaction commits.

**3. Background Service**
- A hosted service scans the Outbox table for unprocessed messages.
- Publishes events to Azure Service Bus and marks them as processed.
- Ensures reliable, eventually consistent integration with other services.

**4. Azure Service Bus**
- Used for decoupled, reliable event delivery to downstream services.

**5. Azure Blob Storage**
- Product manuals are uploaded and stored securely.

---

### 2. InventoryApi

**InventoryApi** is a robust, production-ready .NET 9.0 Razor Pages service for managing inventory across multiple stores. It provides APIs for inventory reservation, allocation, release, and confirmation, supporting distributed, event-driven architectures and seamless integration with other microservices.

---

## Features

- **Multi-Store Inventory Management:**  
  Track and manage inventory for products across multiple physical or logical stores.
- **Inventory Reservation:**  
  Reserve inventory for orders, ensuring stock is held for pending transactions.
- **Release & Confirm Operations:**  
  - **Release:** Frees up reserved inventory if an order is canceled or fails.
  - **Confirm:** Finalizes inventory deduction when an order is completed.
- **Allocation Strategies:**  
  Pluggable strategies (e.g., LargestAvailable, SplitAcrossStores) to determine how inventory is allocated across stores.
- **Event-Driven Outbox Pattern:**  
  Reliable event publishing for inventory changes using an outbox store and background publisher.
- **OpenTelemetry Instrumentation:**  
  Built-in distributed tracing and metrics for observability.
- **Swagger/OpenAPI:**  
  Interactive API documentation and testing.
- **CORS Support:**  
  Configurable cross-origin resource sharing for secure API access.
- **Serilog Logging:**  
  Structured, high-performance logging.

---

## Architecture Overview

- **CQRS & MediatR**: Command and Query Responsibility Segregation for clean separation of reads and writes.
- **Marten**: PostgreSQL-backed document storage for inventory and reservation data.
- **Dependency Injection**: Scoped and singleton services for repositories, strategies, and background jobs.
- **Outbox Publisher**: Ensures reliable event delivery to message brokers (e.g., RabbitMQ).

---

## Key Concepts

### Multi-Store Inventory

- Each product can exist in multiple stores.
- Inventory is tracked per store and per product.
- Allocation strategies determine which store(s) fulfill a reservation.

### Reservation Flow

1. **Reserve**:  
   - Client requests to reserve a quantity of a product.
   - Allocation strategy selects the best store(s).
   - Inventory is marked as reserved, and an outbox event is created.
2. **Confirm**:  
   - Upon successful order/payment, the reservation is confirmed.
   - Inventory is deducted, and a confirmation event is published.
3. **Release**:  
   - If an order is canceled or fails, the reservation is released.
   - Inventory is made available again, and a release event is published.

### Allocation Strategies

- **LargestAvailableStrategy** (default):  
  Allocates from the store with the largest available stock.
- **SplitAcrossStoresStrategy** (optional):  
  Splits allocation across multiple stores if needed.

---

## Project Structure

---

### 3. Basket API

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

### 4. Discount API

**Purpose**: Coupon and promotion management

**Tech Stack**: ASP.NET Core (.NET 8) + SQLite + Redis

**Key Responsibilities:**
- Coupon code validation
- Discount calculation engine
- Promotion rules management
- Time-bound offers

**Why SQLite?** Lightweight, low data volume, simplified deployment for rules-based data

---

### 5. Ordering API - Domain-Driven Design

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

#### Context Mapping

```
┌─────────────────────────────────────────────────────────┐
│                  CONTEXT MAP                            │
│                                                         │
│  Catalog Context                                        │
│       │                                                 │
│       │ (ACL)                                           │
│       ▼                                                 │
│  Ordering Context ◄──(Customer/Supplier)── Basket       │
│       │                                                 │
│       │ (Partnership)                                   │
│       ▼                                                 │
│  Payment Context                                        │
│                                                         │         
└─────────────────────────────────────────────────────────┘
```

**Integration Patterns:**

🔄 **Workflow Details (Event-Driven Messaging with RabbitMQ)**

This system follows an asynchronous event-driven workflow using RabbitMQ as the message broker and MassTransit for message orchestration.

1️⃣ **Basket Checkout Workflow**

Producer: Basket API
Exchange: basket-checkout (fanout/topic – MassTransit managed)

Steps:

User initiates checkout from Basket API.

Basket API validates basket and user data.

Basket API publishes BasketCheckoutEvent to RabbitMQ.

Basket data is deleted after successful publish.

Event Published:

BasketCheckoutEvent

2️⃣ **Order Creation Workflow**

Consumer: Order API
Queue: order-basket-checkout-queue

Steps:

Order API consumes BasketCheckoutEvent.

Event is mapped to CreateOrderCommand.

Order is persisted with Pending status.

Order aggregate raises OrderCreated domain event.

3️⃣ **Order → Payment Workflow**

Producer: Order API
Exchange: order-payment

Steps:

OrderCreated domain event is handled internally.

Order API publishes OrderPaymentEvent to RabbitMQ (feature-flag controlled).

Event Published:

OrderPaymentEvent

4️⃣ **Payment Processing Workflow**

Consumer: Payment API
Queue: payment-order-created-queue

Steps:

Payment API consumes OrderPaymentEvent.

Payment details are validated.

Payment is processed (simulated gateway).

Payment result is determined:

Success → PaymentCompletedEvent

Failure → PaymentFailedEvent

5️⃣ **Order Status Update Workflow**

Consumer: Order API
Queues:

order-payment-completed-queue

order-payment-failed-queue

Steps:

Order API consumes payment result events.

Order status is updated:

Completed → on PaymentCompletedEvent

Cancelled → on PaymentFailedEvent

```
┌─────────────────────────────────────────────────────────┐
│ ┌────────────┐                                          │
│ │ Basket API │                                          │
│ └─────┬──────┘                                          │
│       │  BasketCheckoutEvent                            │
│       ▼                                                 │
│ ┌──────────────────────┐                                │
│ │ RabbitMQ Exchange    │                                │
│ │ basket-checkout      │                                │
│ └─────┬────────────────┘                                │
│       │                                                 │
│       ▼                                                 │
│ ┌────────────┐                                          │
│ │ Order API  │                                          │ 
│ └─────┬──────┘                                          │
│       │  OrderPaymentEvent                              │
│       ▼                                                 │
│ ┌──────────────────────┐                                │
│ │ RabbitMQ Exchange    │                                │
│ │ order-payment        │                                │
│ └─────┬────────────────┘                                │
│       │                                                 │
│       ▼                                                 │
│ ┌────────────┐                                          │
│ │ Payment API│                                          │
│ └─────┬──────┘                                          │
│       │  PaymentCompletedEvent /                        │
│       │  PaymentFailedEvent                             │
│       ▼                                                 │
│ ┌──────────────────────────┐                            │
│ │ RabbitMQ Exchange        │                            │
│ │ payment-status           │                            │
│ └─────┬────────────────────┘                            │
│       │                                                 │
│       ▼                                                 │
│ ┌────────────┐                                          │
│ │ Order API  │                                          │
│ └────────────┘                                          │
└─────────────────────────────────────────────────────────┘

```

**Catalog → Ordering**
- Ordering translates Catalog models to its own domain objects
- Prevents breaking changes from cascading

**Basket → Ordering (Event-Driven)**
- Basket publishes events, Ordering subscribes
- Loose coupling through RabbitMQ

**Ordering → Payment (Synchronous)**
- Direct API calls with shared payment concepts
- Transactional consistency maintained

**Discount → Ordering**
- Ordering calls Discount API for price calculations
- Read-only relationship with graceful degradation

**Technology Stack Explained**
**Carter - Minimal API Endpoints**

Lightweight alternative to traditional Controllers
Functional endpoint definitions
Clean, organized API routes
Supports route grouping and modules

**MediatR - CQRS Implementation**

Commands: Handle write operations (Create, Update, Delete)
Queries: Handle read operations (Get, List, Search)
Separates read and write concerns
Pipeline behaviors for validation, logging, and performance tracking

**FluentValidation - Input Validation**

Strongly-typed validation rules
Automatic validation in MediatR pipeline
Clear error messages
Reusable validation logic

**Marten - Event Sourcing (Ordering Service)**

PostgreSQL as document database
Event store for order history
Stores aggregates as JSON documents
Optimistic concurrency control
Event projections to read models

**Entity Framework Core - Traditional ORM**

Used in Catalog and Discount services
Relational data access
LINQ query support
Database migrations
Change tracking

**Building Blocks - Shared Libraries**
Location: eShopMicroservices/BuildingBlocks/
Reusable components shared across all microservices:

BuildingBlocks.CQRS: ICommand, IQuery, ICommandHandler, IQueryHandler interfaces
BuildingBlocks.Messaging: Domain events, integration events, MassTransit configuration
BuildingBlocks.Behaviors: Validation, logging, and performance pipeline behaviors
BuildingBlocks.Exceptions: Custom domain exceptions

**Polly - Resilience & Retry Policies**

Handles transient failures
Retry policies with exponential backoff
Circuit breaker pattern
Timeout policies
Applied to HTTP clients and database operations

**OpenTelemetry - Distributed Tracing**

End-to-end request tracing across all services
Automatic instrumentation for HTTP, SQL, RabbitMQ
Exports traces to Jaeger
Correlates logs, traces, and metrics
Performance monitoring

**Serilog - Structured Logging**

JSON-formatted logs
Enriched with context (trace IDs, user IDs, etc.)
Centralized logging to Elasticsearch
Queryable log data
Different log levels per environment

**gRPC - Inter-Service Communication**

High-performance RPC framework
Type-safe service contracts
Used for synchronous service-to-service calls
Example: Basket API calls Discount API via gRPC for real-time discount calculations

**CQRS Pattern in Action**
Write Path (Commands):

User submits request → API endpoint (Carter)
Command created and validated (FluentValidation)
Command sent through MediatR pipeline
Command handler processes business logic
Changes persisted to database
Domain events published to RabbitMQ

**Read Path (Queries):**

User requests data → API endpoint (Carter)
Query created and sent through MediatR
Query handler retrieves data
Data cached in Redis (if applicable)
Response returned to user

**Benefits:**

Optimized read and write models
Independent scaling of reads vs writes
Improved performance with caching
Clear separation of concerns

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

**Repository**: [GitOps/infrastructure](https://github.com/RijoyP/GitOps/tree/main/eShopMicroService/infrastructure)

**Deployment Order:**

```
1. FluxCD System          → GitOps operator
2. Ingress NGINX          → External traffic routing
3. Cert Manager           → TLS certificate automation
4. RabbitMQ               → Message broker
5. Logging Stack          → Elasticsearch + Kibana (or PLG)
6. Monitoring Stack       → Prometheus + Grafana + Alertmanager
7. Tracing                → Jaeger with OpenTelemetry
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

![Pipeline Diagram](https://github.com/RijoyP/RijoyP/blob/main/assets/pipelinenew.png)

**Pipeline Files**: [eShopMicroservices/.pipeline](https://github.com/RijoyP/eShopMicroservices/tree/main/.pipelines)

### Automated Profile Generation

**Script**: [generate-profiles.ps1](https://github.com/RijoyP/CICD-Templates/blob/main/applications/scripts/generate-profiles.ps1)

Automated Profile Generation
Script: generate-profiles.ps1
Purpose: Automatically generates Azure DevOps pipeline YAML files for different techinical stack which will help the developement team can use easily.
How It Works:

Scans Services directory to discover all microservices
Loads pipeline templates and environment configurations
For each technical stack × task combination, generates complete pipeline YAML
Validates generated files for syntax and schema
Outputs to applications/profiles/ directory

Generated Profiles: [CICD-Templates/applications/profiles](https://github.com/RijoyP/CICD-Templates/tree/main/applications/profiles)

Sample Generated Files: 

| Profile | Build | Linting | Unit Tests | SCA | Publish | Parameters |
|---------|-------|---------|------------|-----|--------|------------|
| dotnet/dotnet-build-linting-none-all-all.yaml | dotnet/build-dotnet.yaml | dotnet/linting-dotnet.yaml | none/unit-tests-none.yaml | sca/sca-all.yaml | publish/publish-all.yaml | vmImage, poolName, applicationFolder, dotnetVersion, sonarQubeProjectKey, sonarQubeProjectName, sonarQubeServiceConnection, trivySeverity, trivyFormat, serviceConnection, dockerfilePath, buildArgs, dockerContext, trivyExitCode, trivyIgnoreUnfixed, trivyTimeout, acrRegistry, imageName, imageTag |
| dotnet/dotnet-build-linting-none-all-build-image.yaml | publish/publish-build-image.yaml | dotnet/linting-dotnet.yaml | none/unit-tests-none.yaml | sca/sca-all.yaml | - | vmImage, poolName, applicationFolder, dotnetVersion, sonarQubeProjectKey, sonarQubeProjectName, sonarQubeServiceConnection, trivySeverity, trivyFormat, dockerfilePath, imageName, imageTag, buildArgs, dockerContext |
| dotnet/dotnet-build-linting-none-all-none.yaml | dotnet/build-dotnet.yaml | dotnet/linting-dotnet.yaml | none/unit-tests-none.yaml | sca/sca-all.yaml | publish/publish-none.yaml | vmImage, poolName, applicationFolder, dotnetVersion, sonarQubeProjectKey, sonarQubeProjectName, sonarQubeServiceConnection, trivySeverity, trivyFormat |
| dotnet/dotnet-build-linting-none-all-push-acr.yaml | dotnet/build-dotnet.yaml | dotnet/linting-dotnet.yaml | none/unit-tests-none.yaml | sca/sca-all.yaml | publish/publish-push-acr.yaml | vmImage, poolName, applicationFolder, dotnetVersion, sonarQubeProjectKey, sonarQubeProjectName, sonarQubeServiceConnection, trivySeverity, trivyFormat, serviceConnection, dockerfilePath, dockerContext, buildArgs, acrRegistry, imageName, imageTag |
| dotnet/dotnet-build-linting-none-all-scan-image.yaml | dotnet/build-dotnet.yaml | dotnet/linting-dotnet.yaml | none/unit-tests-none.yaml | sca/sca-all.yaml | publish/publish-scan-image.yaml | vmImage, poolName, applicationFolder, dotnetVersion, sonarQubeProjectKey, sonarQubeProjectName, sonarQubeServiceConnection, trivySeverity, trivyFormat, imageName, imageTag, trivyExitCode, trivyIgnoreUnfixed, trivyTimeout |
| react/react-build-none-unit-tests-all-build-image.yaml | publish/publish-build-image.yaml | none/linting-none.yaml | react/unit-tests-react.yaml | sca/sca-all.yaml | - | applicationFolder, nodeVersion, buildCommand, buildOutputFolder, vmImage, poolName, testCommand, codeCoverageThreshold, sonarQubeProjectKey, sonarQubeProjectName, sonarQubeServiceConnection, trivySeverity, trivyFormat, dockerfilePath, imageName, imageTag, buildArgs, dockerContext |
| react/react-build-none-unit-tests-all-none.yaml | react/build-react.yaml | none/linting-none.yaml | react/unit-tests-react.yaml | sca/sca-all.yaml | publish/publish-none.yaml | applicationFolder, nodeVersion, buildCommand, buildOutputFolder, vmImage, poolName, testCommand, codeCoverageThreshold, sonarQubeProjectKey, sonarQubeProjectName, sonarQubeServiceConnection, trivySeverity, trivyFormat |
| react/react-build-none-unit-tests-all-push-acr.yaml | react/build-react.yaml | none/linting-none.yaml | react/unit-tests-react.yaml | sca/sca-all.yaml | publish/publish-push-acr.yaml | applicationFolder, nodeVersion, buildCommand, buildOutputFolder, vmImage, poolName, testCommand, codeCoverageThreshold, sonarQubeProjectKey, sonarQubeProjectName, sonarQubeServiceConnection, trivySeverity, trivyFormat, serviceConnection, dockerfilePath, dockerContext, buildArgs, acrRegistry, imageName, imageTag |
| react/react-build-none-unit-tests-all-scan-image.yaml | react/build-react.yaml | none/linting-none.yaml | react/unit-tests-react.yaml | sca/sca-all.yaml | publish/publish-scan-image.yaml | applicationFolder, nodeVersion, buildCommand, buildOutputFolder, vmImage, poolName, testCommand, codeCoverageThreshold, sonarQubeProjectKey, sonarQubeProjectName, sonarQubeServiceConnection, trivySeverity, trivyFormat, imageName, imageTag, trivyExitCode, trivyIgnoreUnfixed, trivyTimeout |
| react/react-build-none-unit-tests-none-all.yaml | react/build-react.yaml | none/linting-none.yaml | react/unit-tests-react.yaml | sca/sca-none.yaml | publish/publish-all.yaml | applicationFolder, nodeVersion, buildCommand, buildOutputFolder, vmImage, poolName, testCommand, codeCoverageThreshold, serviceConnection, dockerfilePath, buildArgs, dockerContext, trivySeverity, trivyExitCode, trivyIgnoreUnfixed, trivyTimeout, acrRegistry, imageName, imageTag |



---

## 🔄 GitOps with FluxCD

### Repository Structure

**Repository**: [GitOps](https://github.com/RijoyP/GitOps)

```
GitOps/
├── infrastructure/             # Platform components
│   ├── logging/                # Elasticsearch, Kibana
│   ├── tracing/                # Jaeger, Zipkin
│   ├── monitoring/             # Prometheus, Grafana
│   └── messaging/              # RabbitMQ
│
└── apps/                       # Microservice deployments
    └── base/                   #base overlays
        └── ingress/
        └── frontend-react/
        |    ├── dev
        |    ├── stg
        |    └── prod
        ├── catalog-api/
        |    ├── dev
        |    ├── stg
        |    └── prod
        ├── basket-api/
        |    ├── dev
        |    ├── stg
        |    └── prod
        ├── discount-api/
        |    ├── dev
        |    ├── stg
        |    └── prod
        └── ordering-api/
             ├── dev
             ├── stg
             └── prod



```

### FluxCD Workflow

How FluxCD Works with Helm Base + Overlay
Base HelmRelease Template (apps/base/helmrelease.yaml):

Generic template with placeholder values
References the boilerplate Helm chart from ACR
Used as foundation for all services

Service-Specific Overlays (e.g., apps/basket/dev/):

kustomization.yaml: Patches the base HelmRelease

Replaces placeholders with service-specific names
Generates ConfigMaps and Secrets from values files
Injects environment-specific configurations


values-dev.yaml: Contains all service configurations

Image repository and tag
Replica count
Resource limits
Database connections
Redis configuration
RabbitMQ settings
Ingress rules
Health check settings


**Deployment Flow:**

```
1. CI Pipeline completes
   └─> Builds Docker image: basket-api:v1.0.50
   └─> Pushes to ACR
   └─> Updates GitOps repo: apps/basket/dev/values-dev.yaml
   └─> Commits new image tag

2. FluxCD Source Controller (every 1-5 minutes)
   └─> Detects Git commit in GitOps repo

3. FluxCD Kustomize Controller
   └─> Reads apps/basket/dev/kustomization.yaml
   └─> Loads base HelmRelease template
   └─> Generates ConfigMap from values-dev.yaml
   └─> Generates Secret from values-dev.yaml
   └─> Patches base template:
       • Replaces name: app-placeholder → basketapi-dev
       • Injects ConfigMap reference
       • Injects Secret reference

4. FluxCD Helm Controller
   └─> Reads HelmRelease: basketapi-dev
   └─> Fetches Helm chart from ACR
   └─> Merges values from ConfigMap and Secret
   └─> Renders Helm templates
   └─> Applies to AKS namespace: flux-apps

5. Kubernetes
   └─> Creates/Updates Deployment: basketapi-dev (2 replicas)
   └─> Creates/Updates Service: basketapi-dev
   └─> Creates/Updates Ingress: basket-api-dev.eshop.com
   └─> Health checks pass
   └─> Deployment ready

6. FluxCD Notification
   └─> Posts success message to Teams/Slack
```

Key Benefits:

Single Helm Chart: One boilerplate chart used for all services
Environment-Specific Values: Each environment has unique configuration
No Helm CLI: FluxCD manages everything automatically
Git as Source of Truth: All changes tracked and auditable
Automated Reconciliation: Cluster state always matches Git
Easy Rollback: Git revert automatically reverts deployment

### FluxCD Reconciliation Loop

```
┌─────────────────────────────────────────────────────┐
│  FluxCD Reconciliation Loop                         │
│                                                     │
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
│                                                     │
│  ◄────── Continuous Reconciliation (1-5 min) ───────┤
└─────────────────────────────────────────────────────┘
```

Key Features:

Drift Detection: Auto-corrects manual changes back to Git state
Progressive Delivery: Canary deployments with Flagger
Automated Rollback: Reverts on health check failures
Multi-Environment: Separate overlays for dev/staging/prod

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

**Stack: Serilog → Elasticsearch → Kibana**

![Kibana Dashboard](https://github.com/RijoyP/RijoyP/blob/main/assets/elasticsearch.png)

How It Works:

Serilog: Structured logging library integrated into all .NET microservices
Elasticsearch: Stores and indexes log data for fast searching
Kibana: Provides visualization dashboards and log exploration UI

Features:

Centralized structured log aggregation from all services
Full-text search and filtering across all logs
Pre-built Kibana dashboards for error tracking and service health
Trace ID correlation for distributed debugging
Real-time log streaming and alerting

**Deployment**: [GitOps/infrastructure/logging](https://github.com/RijoyP/GitOps/tree/main/eShopMicroService/infrastructure/logging)

---

### Tracing

**Stack: Zipkin + Jaeger with OpenTelemetry → Grafana**

**Zipkin**

![zipkinlist](https://github.com/RijoyP/RijoyP/blob/main/assets/zipkin_list_new.png)

![zipkin](https://github.com/RijoyP/RijoyP/blob/main/assets/zipkin_new.png)

![zipkin_flow](https://github.com/RijoyP/RijoyP/blob/main/assets/zipkin_flow_new.png)

**Jaeger**

![jaeger_ist](https://github.com/RijoyP/RijoyP/blob/main/assets/jeager_all.png)

![jaeger](https://github.com/RijoyP/RijoyP/blob/main/assets/jaeger.png)

How It Works:

OpenTelemetry: Instruments .NET services to collect trace data
Zipkin: Collects and processes trace spans from services
Jaeger: Provides advanced trace storage and querying capabilities
Grafana: Visualizes traces with unified dashboard alongside metrics

**Features:**

End-to-end request tracing across all microservices

**Deployment**: [GitOps/infrastructure/tracing](https://github.com/RijoyP/GitOps/tree/main/eShopMicroService/infrastructure/tracing)

---

### Monitoring

**Stack: Prometheus + Grafana**

![Grafana Dashhboard](https://github.com/RijoyP/RijoyP/blob/main/assets/grafana.png)

How It Works:

Prometheus: Scrapes metrics from all services and infrastructure
Grafana: Provides unified dashboards for metrics, traces, and logs

Metrics Collected:

Application: Request rates, response latencies, error rates, HTTP status codes
Infrastructure: CPU usage, memory consumption, disk I/O, network throughput

**Deployment**: [GitOps/infrastructure/monitoring](https://github.com/RijoyP/GitOps/tree/main/eShopMicroService/infrastructure/monitoring)

---

### Messaging

**Component**: RabbitMQ

**Features:**
- Event-driven communication between services
- Durable queues for reliable message delivery
- Message tracing and monitoring

**Deployment**: [GitOps/infrastructure/messaging](https://github.com/RijoyP/GitOps/tree/main/eShopMicroService/infrastructure/messaging/)

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

**Resilience**: Circuit breakers, retry policies, health checks

**Security**: Azure AD authentication, managed identities, Key Vault integration, network policies

**Observability**: Comprehensive logging, tracing, and monitoring across all layers

**Automation**: Fully automated CI/CD pipelines, GitOps-driven deployments

---

## 🚀 Deployment Summary

1. **Infrastructure**: Terraform provisions Azure resources (AKS, ACR, databases)
2. **Platform**: FluxCD deploys infrastructure components (logging, monitoring, messaging)
3. **Applications**: CI/CD builds images, updates GitOps repo, FluxCD auto-deploys to AKS
4. **Monitoring**: Observability stack provides full system visibility

**All deployments are automated, version-controlled, and auditable through Git.**

**Built with ❤️ using .NET 8, Azure, Kubernetes, and modern DevOps practices**
