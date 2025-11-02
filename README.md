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

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>eShop Microservices Architecture</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }

        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            padding: 20px;
            overflow-x: auto;
        }

        .container {
            background: white;
            border-radius: 12px;
            padding: 30px;
            box-shadow: 0 20px 60px rgba(0,0,0,0.3);
            min-width: 1200px;
        }

        h1 {
            text-align: center;
            color: #2c3e50;
            margin-bottom: 10px;
            font-size: 28px;
        }

        .subtitle {
            text-align: center;
            color: #7f8c8d;
            margin-bottom: 30px;
            font-size: 14px;
        }

        .layer {
            margin-bottom: 20px;
            border: 2px solid #e0e0e0;
            border-radius: 8px;
            padding: 15px;
            position: relative;
        }

        .layer-title {
            position: absolute;
            top: -12px;
            left: 20px;
            background: white;
            padding: 0 10px;
            font-weight: bold;
            font-size: 13px;
            color: #34495e;
        }

        .layer-content {
            display: flex;
            justify-content: space-around;
            align-items: center;
            flex-wrap: wrap;
            gap: 15px;
            min-height: 80px;
        }

        .component {
            padding: 15px 25px;
            border-radius: 8px;
            font-size: 13px;
            font-weight: 500;
            text-align: center;
            box-shadow: 0 4px 6px rgba(0,0,0,0.1);
            transition: transform 0.2s, box-shadow 0.2s;
            cursor: pointer;
        }

        .component:hover {
            transform: translateY(-2px);
            box-shadow: 0 6px 12px rgba(0,0,0,0.15);
        }

        .component-title {
            font-weight: bold;
            margin-bottom: 5px;
        }

        .component-subtitle {
            font-size: 11px;
            opacity: 0.8;
        }

        .client { background: linear-gradient(135deg, #667eea 0%, #764ba2 100%); color: white; }
        .gateway { background: linear-gradient(135deg, #f093fb 0%, #f5576c 100%); color: white; }
        .service { background: linear-gradient(135deg, #4facfe 0%, #00f2fe 100%); color: white; }
        .database { background: linear-gradient(135deg, #43e97b 0%, #38f9d7 100%); color: white; }
        .messaging { background: linear-gradient(135deg, #fa709a 0%, #fee140 100%); color: white; }
        .k8s { background: linear-gradient(135deg, #30cfd0 0%, #330867 100%); color: white; }
        .gitops { background: linear-gradient(135deg, #a8edea 0%, #fed6e3 100%); color: #333; }
        .observability { background: linear-gradient(135deg, #ff9a9e 0%, #fecfef 100%); color: #333; }
        .cicd { background: linear-gradient(135deg, #ffecd2 0%, #fcb69f 100%); color: #333; }

        .arrow {
            text-align: center;
            font-size: 24px;
            color: #7f8c8d;
            margin: 10px 0;
        }

        .services-grid {
            display: grid;
            grid-template-columns: repeat(4, 1fr);
            gap: 15px;
        }

        .service-detail {
            padding: 12px;
            border-radius: 6px;
            font-size: 11px;
        }

        .db-grid {
            display: grid;
            grid-template-columns: repeat(4, 1fr);
            gap: 15px;
        }

        .infra-grid {
            display: grid;
            grid-template-columns: repeat(3, 1fr);
            gap: 15px;
        }

        .obs-grid {
            display: grid;
            grid-template-columns: repeat(4, 1fr);
            gap: 15px;
        }

        .small-component {
            padding: 10px 15px;
            font-size: 11px;
        }

        .legend {
            display: flex;
            justify-content: center;
            gap: 20px;
            margin-top: 30px;
            padding: 15px;
            background: #f8f9fa;
            border-radius: 8px;
        }

        .legend-item {
            display: flex;
            align-items: center;
            gap: 8px;
            font-size: 12px;
        }

        .legend-box {
            width: 30px;
            height: 20px;
            border-radius: 4px;
        }

        @media print {
            body {
                background: white;
                padding: 0;
            }
            .container {
                box-shadow: none;
            }
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>🏗️ eShop Microservices - High Level Architecture</h1>
        <p class="subtitle">Production-Grade Cloud-Native E-Commerce System</p>

        <!-- Client Layer -->
        <div class="layer">
            <div class="layer-title">CLIENT LAYER</div>
            <div class="layer-content">
                <div class="component client">
                    <div class="component-title">Shopping.Web</div>
                    <div class="component-subtitle">ASP.NET Core MVC/Blazor</div>
                </div>
            </div>
        </div>

        <div class="arrow">↓ HTTPS</div>

        <!-- API Gateway Layer -->
        <div class="layer">
            <div class="layer-title">API GATEWAY LAYER</div>
            <div class="layer-content">
                <div class="component gateway">
                    <div class="component-title">YARP API Gateway</div>
                    <div class="component-subtitle">Azure AD Authentication • Routing • Rate Limiting</div>
                </div>
            </div>
        </div>

        <div class="arrow">↓</div>

        <!-- Microservices Layer -->
        <div class="layer">
            <div class="layer-title">MICROSERVICES LAYER</div>
            <div class="layer-content">
                <div class="services-grid">
                    <div class="component service service-detail">
                        <div class="component-title">Catalog API</div>
                        <div class="component-subtitle">.NET 8</div>
                        <div style="margin-top: 8px; font-size: 10px;">
                            Products<br>
                            Categories<br>
                            Inventory
                        </div>
                    </div>
                    <div class="component service service-detail">
                        <div class="component-title">Basket API</div>
                        <div class="component-subtitle">.NET 8</div>
                        <div style="margin-top: 8px; font-size: 10px;">
                            Cart Mgmt<br>
                            Sessions<br>
                            Checkout
                        </div>
                    </div>
                    <div class="component service service-detail">
                        <div class="component-title">Discount API</div>
                        <div class="component-subtitle">.NET 8</div>
                        <div style="margin-top: 8px; font-size: 10px;">
                            Coupons<br>
                            Promotions<br>
                            Rules
                        </div>
                    </div>
                    <div class="component service service-detail">
                        <div class="component-title">Ordering API</div>
                        <div class="component-subtitle">.NET 8 + DDD</div>
                        <div style="margin-top: 8px; font-size: 10px;">
                            Orders<br>
                            Payments<br>
                            Fulfillment
                        </div>
                    </div>
                </div>
            </div>
        </div>

        <div class="arrow">↓</div>

        <!-- Data Layer -->
        <div class="layer">
            <div class="layer-title">DATA PERSISTENCE LAYER</div>
            <div class="layer-content">
                <div class="db-grid">
                    <div class="component database small-component">
                        <div class="component-title">PostgreSQL</div>
                        <div class="component-subtitle">Catalog DB</div>
                    </div>
                    <div class="component database small-component">
                        <div class="component-title">PostgreSQL + Redis</div>
                        <div class="component-subtitle">Basket DB + Cache</div>
                    </div>
                    <div class="component database small-component">
                        <div class="component-title">SQLite</div>
                        <div class="component-subtitle">Discount DB</div>
                    </div>
                    <div class="component database small-component">
                        <div class="component-title">SQL Server</div>
                        <div class="component-subtitle">Ordering DB</div>
                    </div>
                </div>
            </div>
        </div>

        <div class="arrow">↓</div>

        <!-- Messaging Layer -->
        <div class="layer">
            <div class="layer-title">EVENT-DRIVEN MESSAGING LAYER</div>
            <div class="layer-content">
                <div class="component messaging">
                    <div class="component-title">RabbitMQ</div>
                    <div class="component-subtitle">Event Bus • Async Communication • Queues & Exchanges</div>
                </div>
            </div>
        </div>

        <div class="arrow">↓</div>

        <!-- Kubernetes Layer -->
        <div class="layer">
            <div class="layer-title">AZURE KUBERNETES SERVICE (AKS)</div>
            <div class="layer-content">
                <div class="infra-grid">
                    <div class="component k8s small-component">
                        <div class="component-title">Ingress Controller</div>
                        <div class="component-subtitle">NGINX</div>
                    </div>
                    <div class="component k8s small-component">
                        <div class="component-title">Service Mesh</div>
                        <div class="component-subtitle">Traffic Mgmt</div>
                    </div>
                    <div class="component k8s small-component">
                        <div class="component-title">Azure Key Vault</div>
                        <div class="component-subtitle">Secrets</div>
                    </div>
                </div>
            </div>
        </div>

        <div class="arrow">↕ GitOps Sync</div>

        <!-- GitOps Layer -->
        <div class="layer">
            <div class="layer-title">GITOPS WITH FLUXCD</div>
            <div class="layer-content">
                <div class="infra-grid">
                    <div class="component gitops small-component">
                        <div class="component-title">Source Controller</div>
                        <div class="component-subtitle">Git Monitor</div>
                    </div>
                    <div class="component gitops small-component">
                        <div class="component-title">Kustomize Controller</div>
                        <div class="component-subtitle">Apply Manifests</div>
                    </div>
                    <div class="component gitops small-component">
                        <div class="component-title">GitOps Repository</div>
                        <div class="component-subtitle">/apps + /infrastructure</div>
                    </div>
                </div>
            </div>
        </div>

        <!-- Observability Layer -->
        <div class="layer">
            <div class="layer-title">OBSERVABILITY STACK</div>
            <div class="layer-content">
                <div class="obs-grid">
                    <div class="component observability small-component">
                        <div class="component-title">Logging</div>
                        <div class="component-subtitle">ELK / PLG Stack</div>
                    </div>
                    <div class="component observability small-component">
                        <div class="component-title">Tracing</div>
                        <div class="component-subtitle">Jaeger</div>
                    </div>
                    <div class="component observability small-component">
                        <div class="component-title">Monitoring</div>
                        <div class="component-subtitle">Prometheus</div>
                    </div>
                    <div class="component observability small-component">
                        <div class="component-title">Visualization</div>
                        <div class="component-subtitle">Grafana</div>
                    </div>
                </div>
            </div>
        </div>

        <!-- CI/CD Layer -->
        <div class="layer">
            <div class="layer-title">CI/CD PIPELINE</div>
            <div class="layer-content">
                <div class="component cicd">
                    <div class="component-title">Azure DevOps Pipeline</div>
                    <div class="component-subtitle">Build → Test → Docker Build → ACR Push → GitOps Update → FluxCD Deploy</div>
                </div>
            </div>
        </div>

        <!-- Legend -->
        <div class="legend">
            <div class="legend-item">
                <div class="legend-box" style="background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);"></div>
                <span>Client Apps</span>
            </div>
            <div class="legend-item">
                <div class="legend-box" style="background: linear-gradient(135deg, #4facfe 0%, #00f2fe 100%);"></div>
                <span>Microservices</span>
            </div>
            <div class="legend-item">
                <div class="legend-box" style="background: linear-gradient(135deg, #43e97b 0%, #38f9d7 100%);"></div>
                <span>Databases</span>
            </div>
            <div class="legend-item">
                <div class="legend-box" style="background: linear-gradient(135deg, #30cfd0 0%, #330867 100%);"></div>
                <span>Kubernetes</span>
            </div>
            <div class="legend-item">
                <div class="legend-box" style="background: linear-gradient(135deg, #a8edea 0%, #fed6e3 100%);"></div>
                <span>GitOps</span>
            </div>
        </div>
    </div>
</body>
</html>


