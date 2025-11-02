# eShopMicroServices — High Level Architecture & Service Responsibilities

Summary
- eShopMicroServices is a containerized .NET microservices suite composed of four primary backend services (Basket, Discount, Order, Catalog).  
- Deployed on AKS using Flux GitOps and Azure infrastructure provisioned via Terraform (CICD-Templates + terraform-modules).  
- CI builds images, pushes to ACR, and updates the GitOps repo; Flux reconciles cluster state.

Mermaid: high-level diagram
```mermaid
graph TB
  subgraph Clients
    Web[Web App]
    Mobile[Mobile App]
    Admin[Admin UI]
  end

  API[API Gateway (YARP / Nginx)]
  Clients --> API

  subgraph AKS[AKS Cluster]
    API --> Basket[Basket API]
    API --> Catalog[Catalog API]
    API --> Discount[Discount API]
    API --> Order[Order API]

    Basket --> Redis[(Redis Cache)]
    Basket --> RabbitMQ[(RabbitMQ)]
    Order --> RabbitMQ
    Catalog --> SQLServer[(SQL Server)]
    Order --> SQLServer
    Discount --> Postgres[(PostgreSQL)]

    Observability[(Logging / Tracing / Metrics)]
    Observability --> Prometheus[(Prometheus)]
    Observability --> Jaeger[(Jaeger)]
    Observability --> EFK[(Elasticsearch / Kibana)]
  end

  CI[CI/CD (CICD-Templates)]
  CI --> ACR[(Azure Container Registry)]
  ACR --> AKS
  CI --> GitOps[(Flux GitOps repo)]
  GitOps --> AKS

  Infra[(Terraform modules)]
  Infra --> Azure[(Azure resources: AKS, ACR, KeyVault, Networking...)]
```

Services — purpose, importance, and key integrations

- Basket API
  - Purpose: manage user shopping carts (add/update/remove items) and initiate checkout.
  - Why it matters: central to checkout UX; transient state impacts conversion rates.
  - Data & integrations:
    - Redis for fast, ephemeral cart storage and TTL-based cleanup.
    - Calls Discount API to validate/apply coupons during checkout.
    - Publishes checkout events to RabbitMQ for eventual Order creation and downstream processing.
  - Operational focus: low-latency reads/writes, cache reliability, event durability.

- Discount API
  - Purpose: host discount rules, coupon validation, price adjustments and promotions.
  - Why it matters: enforces business pricing logic and prevents fraud/discount misuse.
  - Data & integrations:
    - PostgreSQL for relational discount models and history/audit.
    - Exposed endpoints consumed by Basket and Order for validation and pricing.
  - Operational focus: ACID integrity for rules, low-latency lookups, safe migrations for rules.

- Order API
  - Purpose: persist orders, handle order lifecycle (created, paid, shipped), orchestrate fulfillment.
  - Why it matters: source of truth for purchases, billing, fulfillment and customer history.
  - Data & integrations:
    - SQL Server (transactional DB) for orders, line items, payments, and history.
    - Consumes checkout events from RabbitMQ to create orders asynchronously (decouples frontend latency).
    - May publish order-status events to other systems (email, inventory, billing).
  - Operational focus: transactional integrity, idempotency when consuming messages, retention and backup.

- Catalog API
  - Purpose: product catalog CRUD, categories, inventory surface, search endpoints.
  - Why it matters: primary product data provider for UI and other services; impacts discoverability and inventory accuracy.
  - Data & integrations:
    - SQL Server for product and inventory data.
    - Exposes endpoints to clients and internal services; may integrate with search/cache layers.
  - Operational focus: read scalability, cache strategies, synchronization with upstream data sources.

Infrastructure & platform components

- RabbitMQ (Messaging)
  - Role: event bus for checkout/order flows, decouples producers (Basket) from consumers (Order, fulfillment).
  - Importance: enables asynchronous processing and resilience under load; durable queues for reliability.

- Redis (Cache)
  - Role: fast key-value store for Basket session/cart data and frequently read data.
  - Importance: reduces DB load and improves responsiveness for cart operations.

- Databases
  - PostgreSQL: Discount service (relational rules, moderate scale).
  - SQL Server: Order and Catalog (transactional workloads, relational constraints).

- API Gateway (YARP / Nginx)
  - Role: central ingress, routing, authentication offload, rate-limiting and TLS termination.
  - Importance: protects and simplifies client-to-service access; central place for cross-cutting concerns.

Observability & cross-cutting concerns
- Logging: EFK stack (Elasticsearch / Fluent Bit / Kibana) for centralized logs.
- Metrics: Prometheus + Grafana for service and infra metrics.
- Tracing: Jaeger/OpenTelemetry for distributed traces (checkout/order flows).
- Secrets: Azure Key Vault for credentials; integrate via secrets-store CSI or service injection.
- Security: Azure AD for auth, network policies and least-privilege RBAC.

CI/CD, GitOps & IaC
- CICD-Templates: central pipeline profiles + generate-profiles.ps1 to produce service pipelines that build, scan, push images and update GitOps.
- ACR: image registry for built images.
- GitOps (Flux): apps/ and infrastructure/ repos contain HelmReleases, kustomize overlays and infra charts. Pipelines update image tags or Helm values; Flux reconciles into AKS.
- Terraform modules: reusable modules in terraform-modules/Modules for AKS, ACR, KeyVault, DBs, networking used by Infra/main.tf for provisioning environments.

Typical end-to-end flows (brief)
- Checkout flow: User -> Basket -> apply Discount -> publish checkout event -> RabbitMQ -> Order consumes -> persist order -> trigger downstream events (email/fulfillment).
- Product update: Admin -> Catalog DB update -> cache invalidation -> optional event for downstream sync.

Operational recommendations
- Scale Basket and Catalog replicas for read-heavy traffic; use HPA (metrics).
- Ensure RabbitMQ queue durability and consumer idempotency.
- Back up SQL Server/Postgres regularly; use managed Azure services where appropriate.
- Use feature toggles and safe DB migrations for rollout.
- Monitor SLOs: cart success rate, order throughput, event lag, DB latency.

Quick repo pointers
- Services code: c:\Learn\github\eShopMicroServices\Services\*
- GitOps manifests: c:\Learn\github\GitOps\eShopMicroService\ (apps/ and infrastructure/)
- Terraform modules: c:\Learn\github\terraform-modules\Modules\*
- CICD templates & generator: c:\Learn\github\CICD-Templates\applications\scripts\generate-profiles.ps1
- Helm boilerplate chart: c:\Learn\github\helm-templates\app-boilerplate-chart

If needed, a short README per service with build/run/deploy steps can be created next. 
