# eShopMicroServices — Architecture & High‑Level Design

Summary
- eShopMicroServices is a collection of 4 backend services (Basket, Discount, Order, Catalog) built as containerized .NET services and deployed on AKS using GitOps (Flux) and Azure infrastructure-as-code (Terraform).
- CI/CD uses the common CICD-Templates repository to generate pipeline profiles, build images, push to ACR and update GitOps for deployment.

High-level architecture (Mermaid)
```mermaid
%% high-level eShop microservices architecture
graph TB
  subgraph Clients
    Web[Web App]
    Mobile[Mobile App]
  end

  API[API Gateway (YARP / Nginx)]
  Clients --> API

  subgraph AKS
    API --> Basket
    API --> Catalog
    API --> Discount
    API --> Order

    Basket --> Redis[(Redis Cache)]
    Basket --> RabbitMQ[(RabbitMQ)]
    Order --> RabbitMQ
    Catalog --> SQLServer[(SQL Server)]
    Order --> SQLServer
    Discount --> Postgres[(PostgreSQL)]
    Observability[(Logging / Tracing / Metrics)]
    Observability -->|Scrapes / Collects| Prometheus[(Prometheus)] & Jaeger[(Jaeger)] & EFK[(Elasticsearch)]
  end

  CI[CI/CD (CICD-Templates)]
  CI --> ACR[(Azure Container Registry)]
  ACR --> AKS
  CI --> GitOps[(Flux GitOps repo)]
  GitOps --> AKS
  Infra[(Terraform modules)]
  Infra --> Azure[(Azure resources: AKS, ACR, KeyVault, Networking...)]
```

Services — responsibilities and data stores
- Basket API
  - Purpose: cart management, checkout initiation.
  - Storage: Redis for fast cart storage and session-like data.
  - Messaging: publishes checkout/checkout-initiated events to RabbitMQ.
  - Key files: Services/Basket/ (source), values-basketapi.yaml.

- Discount API
  - Purpose: coupon/discount rules, price adjustments.
  - Storage: PostgreSQL (see terraform-modules/Modules/Postgres*/).
  - Access: Dapper/EF (service code in Services/Discount/).
  - Used by Basket at checkout to validate and apply discounts.

- Order API
  - Purpose: create and track orders, persist order history.
  - Storage: SQL Server (SqlServer / SqlDatabase modules in terraform-modules).
  - Messaging: consumes checkout events from RabbitMQ to create orders; may publish order events.

- Catalog API
  - Purpose: product and inventory management, search.
  - Storage: SQL Server (Catalog DB).
  - Provides product endpoints used by web/mobile and other services.

Infrastructure & IaC
- Terraform modules: c:\Learn\github\terraform-modules\Modules
  - Modules for AKS, ACR, KeyVault, Postgres, SQL Server, Storage, Networking, etc.
  - Use these modules in Infra/main.tf under eShopMicroServices/Infra for environment provisioning.

- CICD-Templates: c:\Learn\github\CICD-Templates
  - Contains pipeline profile templates and `generate-profiles.ps1` to auto-generate pipeline YAML profiles per app.
  - Pipeline flow (typical): build → unit-tests → docker build → push to ACR → update GitOps (or Helm/manifest repo).

- Helm templates: c:\Learn\github\helm-templates
  - app-boilerplate-chart is the common chart used to deploy services to AKS (deployment, service, ingress, hpa, secrets).
  - CI pipelines import/use this chart, patch values for each service environment.

GitOps & Flux
- GitOps repository: c:\Learn\github\GitOps\eShopMicroService
  - Two main areas: apps/ (app manifests / HelmReleases / kustomize overlays) and infrastructure/ (logging, monitoring, messaging, ingress, istio).
  - clusters/ contains per-cluster overlays and Flux bootstrap manifests (gotk-components, gotk-sync).
  - Deploy order:
    1. Flux components bootstrap (flux controllers).
    2. Infra manifests (logging, monitoring, messaging — e.g., RabbitMQ helmRelease in infrastructure/messaging/rabbitmq).
    3. Base application HelmRelease/HelmRepository (apps/base).
    4. Environment overlays (apps/<service>/{dev,stg,prod}) which reference image tags and environment-specific values.

Observability, logging & tracing
- Logging: EFK (Elasticsearch, Fluentd/Fluent Bit, Kibana) deployed via GitOps infrastructure logging manifests.
- Metrics: Prometheus + Grafana (dashboards in GitOps/monitoring).
- Tracing: Jaeger/Zipkin (tracing manifests in GitOps/tracing).
- Instrumentation: services integrate OpenTelemetry for traces/metrics.

Messaging & State
- RabbitMQ is deployed as a HelmRelease in GitOps/infrastructure/messaging/rabbitmq and used for async events (checkout, order events).
- Redis is used for Basket caching (deployed via GitOps/storage/redis or Azure Cache for Redis via Terraform).

Deployment flow (typical)
1. Developer pushes changes to service repo (e.g., Services/Basket).
2. CICD-Templates pipeline (generated from profiles) builds images, runs tests and security scans.
3. Built image is pushed to ACR.
4. Pipeline updates the GitOps apps repo (image tag in values YAML or HelmRelease).
5. Flux in cluster detects Git change and reconciles (pulls new images / updates deployments).
6. Observability and infra resources are already managed via Flux infra repo overlays.

Where to look (quick pointers)
- Services code: c:\Learn\github\eShopMicroServices\Services\*
- Terraform modules: c:\Learn\github\terraform-modules\Modules\*
- Infra deployment for this project: c:\Learn\github\eShopMicroServices\Infra\main.tf
- CICD pipeline templates & profile generator: c:\Learn\github\CICD-Templates\applications\scripts\generate-profiles.ps1 and profiles/
- Helm boilerplate chart: c:\Learn\github\helm-templates\app-boilerplate-chart
- GitOps manifests: c:\Learn\github\GitOps\eShopMicroService\ (apps/ and infrastructure/)

Example data flows (short)
- Checkout:
  - Basket validates discounts with Discount API → publishes checkout event to RabbitMQ → Order API consumes and persists order → emit order-created event → downstream processes (email, fulfillment).
- Product update:
  - Catalog API update → DB updated → cache invalidation in Redis → optional event to Discount/Order services if price/availability changed.

Operational notes
- Use generate-profiles.ps1 to create pipeline profiles for each service and environment before creating pipelines.
- Use app-boilerplate-chart values files in GitOps/apps/<service>/<env>/values-dev.yaml to control deployment settings (replicas, resource limits, env vars, image tags).
- Secure secrets with Azure Key Vault and reference from pipelines and AKS via secrets-store CSI or Kubernetes secrets synced by Flux.

If you want, I can:
- Generate a polished markdown architecture diagram file (PNG/SVG) from the mermaid block.
- Create a short README per service (Basket/Discount/Order/Catalog) inside Services/* with run/build/deploy steps.
