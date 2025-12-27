# Docker & Kubernetes with Azure: Complete Learning Guide
## Focus: Microservices Architecture with Azure Services (.NET/C#)

---

## TABLE OF CONTENTS
1. [Core Concepts](#core-concepts)
2. [Architecture Overview](#architecture-overview)
3. [Local Development Setup](#local-development-setup)
4. [Docker Fundamentals for Microservices](#docker-fundamentals)
5. [Docker Layers Deep Dive](#docker-layers-deep-dive)
6. [Dockerfile Best Practices](#dockerfile-best-practices)
7. [Kubernetes Fundamentals](#kubernetes-fundamentals)
8. [Azure Container Services](#azure-container-services)
9. [Multi-Container Setup](#multi-container-setup)
10. [Authentication & Accessibility](#authentication-accessibility)
11. [Step-by-Step Implementation (.NET/C#)](#step-by-step-implementation)
12. [Service Bus Communication in C#](#service-bus-communication)
13. [Testing & Troubleshooting](#testing-troubleshooting)

---

## CORE CONCEPTS

### What is Docker?
**Docker** is a containerization platform that packages your application + dependencies into a standardized unit (container).

**Key Terms:**
- **Image**: A blueprint (like a recipe) containing application code, runtime, libraries, and environment variables
- **Container**: A running instance of an image (like baking a cake from the recipe)
- **Registry**: A repository to store and share images (Docker Hub, Azure Container Registry)
- **Layer**: Individual instruction in Dockerfile (each creates a cacheable layer)

**Why Docker?**
- Consistency: "Works on my machine" → "Works everywhere"
- Lightweight: Containers share the host OS kernel (unlike VMs)
- Isolation: Each container runs independently
- Reproducibility: Same image produces same results every time

### What is Kubernetes?
**Kubernetes (K8s)** is an orchestration platform that manages containers at scale.

**Key Concepts:**
- **Cluster**: A group of machines (nodes) running containers
- **Pod**: Smallest deployable unit (usually 1 container, but can have multiple)
- **Service**: A stable network endpoint for accessing pods
- **Deployment**: Declarative way to manage pod replicas
- **Namespace**: Virtual cluster within a cluster (for multi-tenancy)

**Why Kubernetes?**
- Auto-scaling: Add/remove pods based on demand
- Self-healing: Restart failed containers automatically
- Load balancing: Distribute traffic across pods
- Rolling updates: Update without downtime

### Azure Kubernetes Service (AKS)
**AKS** is Microsoft's managed Kubernetes service (you manage apps, Azure manages infrastructure).

**Key Azure Components:**
- **Azure Container Registry (ACR)**: Private image registry
- **Azure Cosmos DB**: NoSQL database (globally distributed)
- **Azure Service Bus**: Message broker for inter-service communication
- **Azure Container Instances**: Run containers without Kubernetes (for simple workloads)
- **Azure Key Vault**: Secure secret management

---

## ARCHITECTURE OVERVIEW

### Your Microservices Setup

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client Application                        │
└──────────────────────────────┬──────────────────────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │  API Gateway/       │
                    │  Load Balancer      │
                    │  (NGINX Ingress)    │
                    └──────────┬──────────┘
                               │
        ┌──────────────────────┼──────────────────────┐
        │                      │                      │
   ┌────▼──────┐       ┌──────▼──────┐       ┌──────▼──────┐
   │ Service 1 │       │ Service 2   │       │ Service 3   │
   │ (.NET App)│       │ (.NET App)  │       │ (.NET App)  │
   │           │       │             │       │             │
   │ Order API │       │ Delivery API│       │ Payment API │
   └────┬──────┘       └──────┬──────┘       └──────┬──────┘
        │                      │                     │
        │                      │                     │
   ┌────▼──────────────────────▼──────────────────────▼─────┐
   │         Azure Service Bus (Message Queue)              │
   │  - Decouples services                                 │
   │  - Enables asynchronous communication               │
   │  - Ensures reliable message delivery                │
   └────────────────────────────────────────────────────────┘
        │                      │                     │
   ┌────▼──────┐       ┌──────▼──────┐       ┌──────▼──────┐
   │ Azure      │       │ Container   │       │ SQL Server  │
   │ Cosmos DB  │       │ Instance    │       │ Database    │
   │ (NoSQL DB) │       │ (Database)  │       │             │
   └────────────┘       └─────────────┘       └─────────────┘

All running in: Azure Kubernetes Service (AKS) Cluster
Orchestrated by: Kubernetes
Images stored in: Azure Container Registry (ACR)
Secrets stored in: Azure Key Vault
```

### Communication Flow

1. **Service-to-Service**: Via Azure Service Bus (asynchronous, event-driven)
2. **Service-to-Database**: Direct connection (within VNet)
3. **Client-to-Services**: Via API Gateway (NGINX Ingress)

---

## LOCAL DEVELOPMENT SETUP

### Prerequisites
- **Docker Desktop** (includes Docker CLI + Kubernetes for local testing)
- **.NET 7+ SDK** (for building C# applications)
- **kubectl** (Kubernetes CLI)
- **Azure CLI** (for Azure integration)
- **Git** (for version control)
- **Visual Studio Code** or **Visual Studio** (IDE)

### Installation Steps

#### Step 1: Install Docker Desktop
```bash
# macOS/Windows: Download from docker.com
# Linux: 
sudo apt-get update
sudo apt-get install docker.io
sudo usermod -aG docker $USER  # Run Docker without sudo
```

#### Step 2: Install .NET SDK
```bash
# macOS:
brew install dotnet

# Windows: Download from https://dotnet.microsoft.com/download
# Linux: https://docs.microsoft.com/en-us/dotnet/core/install/linux
```

#### Step 3: Install kubectl
```bash
# Using Azure CLI:
az aks install-cli

# Or standalone:
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

#### Step 4: Install Azure CLI
```bash
# macOS:
brew install azure-cli

# Windows/Linux: Visit https://docs.microsoft.com/cli/azure/install-azure-cli
```

#### Step 5: Enable Kubernetes in Docker Desktop
- Open Docker Desktop Settings
- Go to Kubernetes tab
- Check "Enable Kubernetes"
- Wait for initialization (may take 5-10 minutes)

#### Step 6: Verify Installation
```bash
docker --version
dotnet --version
kubectl version --client
az --version
```

---

## DOCKER FUNDAMENTALS FOR MICROSERVICES

### Understanding Dockerfile

A Dockerfile is like a recipe for building an image. Each instruction creates a layer.

```dockerfile
# Start from a base image
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS builder

# Set working directory inside container
WORKDIR /source

# Copy project files
COPY . .

# Restore NuGet packages and build
RUN dotnet restore "OrderService.csproj"
RUN dotnet publish -c Release -o /app

# Runtime stage (smaller final image)
FROM mcr.microsoft.com/dotnet/aspnet:7.0

WORKDIR /app

# Copy built application from builder stage
COPY --from=builder /app .

# Expose port (for documentation)
EXPOSE 5000

# Set environment variables
ENV ASPNETCORE_URLS=http://+:5000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:5000/health || exit 1

# Command to run when container starts
ENTRYPOINT ["dotnet", "OrderService.dll"]
```

**Explanation:**
- **FROM**: Base image (OS + runtime)
- **WORKDIR**: Directory where commands execute
- **COPY**: Copy files from host to container
- **RUN**: Execute commands during build (creates layers, cached)
- **EXPOSE**: Document which ports the app uses
- **ENV**: Environment variables
- **HEALTHCHECK**: Container health monitoring
- **ENTRYPOINT**: Default executable
- **CMD**: Default arguments

### Key Docker Concepts for Microservices

#### 1. Multi-stage Builds (Reduces Image Size)

```dockerfile
# Stage 1: Build stage (large - includes SDK)
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS builder
WORKDIR /source
COPY . .
RUN dotnet restore
RUN dotnet publish -c Release -o /app

# Stage 2: Runtime stage (small - only runtime)
FROM mcr.microsoft.com/dotnet/aspnet:7.0
WORKDIR /app
COPY --from=builder /app .
EXPOSE 5000
ENTRYPOINT ["dotnet", "OrderService.dll"]
```

**Why?**
- SDK image: ~1.5 GB (has compilers, build tools)
- Runtime image: ~150 MB (just runtime)
- Final image: Only runtime included, 10x smaller!

#### 2. Layer Caching (Optimization)

```dockerfile
# Bad: Changes to code bust entire cache
FROM mcr.microsoft.com/dotnet/sdk:7.0
COPY . .
RUN dotnet restore
RUN dotnet publish

# Good: Separate restore and code copy
FROM mcr.microsoft.com/dotnet/sdk:7.0
COPY *.csproj ./
RUN dotnet restore                    # Layer cached if .csproj unchanged
COPY . .                              # Only this rebuilds when code changes
RUN dotnet publish                    # Rebuilt if code layer changed
```

**Why?** Docker caches layers. If code changes but project file doesn't, restore step is skipped.

#### 3. Environment Variables & Configuration

```dockerfile
# Set during build
ENV SERVICE_NAME=order-service
ENV LOG_LEVEL=INFO
ENV ASPNETCORE_ENVIRONMENT=Production
```

**In Kubernetes, override at runtime:**
```yaml
env:
  - name: SERVICE_NAME
    value: "order-service"
  - name: LOG_LEVEL
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: log-level
  - name: DATABASE_CONNECTION_STRING
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: connection-string
```

#### 4. Health Checks

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:5000/health || exit 1
```

**In .NET, implement health endpoint:**
```csharp
app.MapGet("/health", () => Results.Ok(new { status = "healthy" }));
```

**In Kubernetes, use liveness/readiness probes:**
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 5000
  initialDelaySeconds: 10
  periodSeconds: 30

readinessProbe:
  httpGet:
    path: /ready
    port: 5000
  initialDelaySeconds: 5
  periodSeconds: 10
```

---

## DOCKER LAYERS DEEP DIVE

### Understanding Docker Layers

**Every Dockerfile instruction creates a layer (except FROM and LABEL).**

```
Dockerfile:
  FROM mcr.microsoft.com/dotnet/sdk:7.0          → Layer 1 (base image)
  WORKDIR /app                                   → Layer 2 (metadata only)
  COPY *.csproj ./                               → Layer 3 (new files)
  RUN dotnet restore                             → Layer 4 (runs command)
  COPY . .                                       → Layer 5 (more files)
  RUN dotnet publish -c Release -o /out          → Layer 6 (command result)
  FROM mcr.microsoft.com/dotnet/aspnet:7.0       → Layer 7 (new base)
  WORKDIR /app                                   → Layer 8 (metadata)
  COPY --from=builder /out .                     → Layer 9 (files from layer 6)
  EXPOSE 5000                                    → Layer 10 (metadata)
  ENTRYPOINT ["dotnet", "app.dll"]               → Layer 11 (metadata)

Image Size = Layer 7 (150MB) + Layer 9 (50MB) = ~200MB
(All previous layers discarded in multi-stage build)
```

### Layer Caching Mechanism

```
Build 1: docker build -t myapp:v1.0 .
  Layer 1: From cache (base image)
  Layer 2: From cache (WORKDIR)
  Layer 3: From cache (COPY *.csproj) - dependencies unchanged
  Layer 4: From cache (RUN dotnet restore) - no need to re-run
  Layer 5: BUILD (COPY . .) - code changed, rebuild
  Layer 6: BUILD (RUN dotnet publish) - must rebuild

Build 2: docker build -t myapp:v1.1 . (only code changed)
  Layer 1-4: Use cache (FAST - 1 second)
  Layer 5-6: Rebuild (4 seconds)
  Total: 5 seconds instead of 20 seconds
```

### Layer Optimization Tips

```dockerfile
# ❌ BAD: Large layer, waste of space
FROM mcr.microsoft.com/dotnet/sdk:7.0
COPY . .                                        # Copies 500MB
RUN dotnet restore && dotnet publish ...        # Creates 600MB layer

# ✅ GOOD: Separate concerns, smaller layers
FROM mcr.microsoft.com/dotnet/sdk:7.0
COPY *.csproj ./ *.sln ./                       # Only 5MB
RUN dotnet restore                              # Creates 300MB layer
COPY . .                                        # Only 5MB (source code)
RUN dotnet publish -c Release -o /app           # Creates 50MB layer

# ❌ BAD: Multiple RUN statements create multiple layers
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y git
RUN apt-get clean                               # Total: 4 layers

# ✅ GOOD: Chain commands in single RUN
RUN apt-get update && \
    apt-get install -y curl git && \
    apt-get clean                               # Single layer, cache busting only once
```

### Viewing Layers

```bash
# See image layers and size
docker history myapp:v1.0

# OUTPUT:
# IMAGE          CREATED            SIZE      CREATED BY
# abc123         2 hours ago        50MB      ENTRYPOINT ["dotnet" "app.dll"]
# def456         2 hours ago        100MB     RUN dotnet publish -c Release
# ghi789         2 hours ago        5MB       COPY . .
# jkl012         3 hours ago        300MB     RUN dotnet restore
# mno345         3 hours ago        150MB     FROM mcr.microsoft.com/dotnet/...

# Inspect image layers in detail
docker inspect myapp:v1.0
```

---

## DOCKERFILE BEST PRACTICES

### .NET-Specific Best Practices

#### 1. Use specific versions
```dockerfile
# ❌ Bad: Latest changes, unpredictable
FROM mcr.microsoft.com/dotnet/aspnet:latest

# ✅ Good: Fixed version
FROM mcr.microsoft.com/dotnet/aspnet:7.0.11
```

#### 2. Run as non-root user
```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:7.0

# Create non-root user
RUN useradd -m -u 1001 appuser

WORKDIR /app
COPY --from=builder /app .

# Change ownership
RUN chown -R appuser:appuser /app

# Switch to non-root user
USER appuser

EXPOSE 5000
ENTRYPOINT ["dotnet", "OrderService.dll"]
```

#### 3. Don't run as root (security)
```dockerfile
# ❌ Bad: Running as root
FROM mcr.microsoft.com/dotnet/aspnet:7.0
COPY --from=builder /app .
ENTRYPOINT ["dotnet", "app.dll"]  # Runs as root!

# ✅ Good: Running as non-root
FROM mcr.microsoft.com/dotnet/aspnet:7.0
RUN useradd -m appuser
USER appuser
COPY --from=builder /app .
ENTRYPOINT ["dotnet", "app.dll"]  # Runs as appuser
```

#### 4. Minimize attack surface
```dockerfile
# ❌ Bad: Includes unnecessary tools
FROM mcr.microsoft.com/dotnet/sdk:7.0
COPY . .
RUN dotnet publish

# ✅ Good: Only runtime in final image
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS builder
COPY . .
RUN dotnet publish -o /app

FROM mcr.microsoft.com/dotnet/aspnet:7.0
COPY --from=builder /app .
ENTRYPOINT ["dotnet", "app.dll"]
```

#### 5. Set proper environment for ASP.NET Core
```dockerfile
ENV ASPNETCORE_URLS=http://+:5000
ENV ASPNETCORE_ENVIRONMENT=Production
ENV DOTNET_RUNNING_IN_CONTAINER=true
```

---

## DOCKER COMPOSE (Local Multi-Container Setup)

### Complete docker-compose.yml Example

```yaml
version: '3.8'

services:
  # Service 1: Order Service (.NET)
  order-service:
    build:
      context: ./services/OrderService
      dockerfile: Dockerfile
    container_name: order-service
    ports:
      - "5000:5000"
    environment:
      - SERVICE_NAME=order-service
      - LOG_LEVEL=Information
      - ASPNETCORE_ENVIRONMENT=Development
      - ServiceBusConnectionString=Endpoint=sb://localhost:5672/;...
      - CosmosConnectionString=mongodb://cosmosdb:27017
    depends_on:
      - service-bus
      - cosmosdb
    networks:
      - app-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5000/health"]
      interval: 30s
      timeout: 3s
      retries: 3

  # Service 2: Delivery Service (.NET)
  delivery-service:
    build:
      context: ./services/DeliveryService
      dockerfile: Dockerfile
    container_name: delivery-service
    ports:
      - "5001:5001"
    environment:
      - SERVICE_NAME=delivery-service
      - LOG_LEVEL=Information
      - ASPNETCORE_ENVIRONMENT=Development
      - ServiceBusConnectionString=Endpoint=sb://localhost:5672/;...
      - CosmosConnectionString=mongodb://cosmosdb:27017
    depends_on:
      - service-bus
      - cosmosdb
    networks:
      - app-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5001/health"]
      interval: 30s
      timeout: 3s
      retries: 3

  # Cosmos DB Emulator
  cosmosdb:
    image: mcr.microsoft.com/cosmosdb/linux/azure-cosmos-emulator:latest
    container_name: cosmos-emulator
    ports:
      - "8081:8081"
      - "27017:27017"
    environment:
      AZURE_COSMOS_EMULATOR_PARTITION_COUNT: 4
      AZURE_COSMOS_EMULATOR_ENABLE_DATA_PERSISTENCE: "true"
    volumes:
      - cosmos-data:/data/db
    networks:
      - app-network
    restart: unless-stopped

  # RabbitMQ (Azure Service Bus equivalent for local testing)
  service-bus:
    image: rabbitmq:3.12-management
    container_name: service-bus
    ports:
      - "5672:5672"      # AMQP
      - "15672:15672"    # Management UI
    environment:
      RABBITMQ_DEFAULT_USER: guest
      RABBITMQ_DEFAULT_PASS: guest
    volumes:
      - service-bus-data:/var/lib/rabbitmq
    networks:
      - app-network
    restart: unless-stopped

  # SQL Server (optional, for Service 3)
  sqlserver:
    image: mcr.microsoft.com/mssql/server:2019-latest
    container_name: sqlserver
    ports:
      - "1433:1433"
    environment:
      ACCEPT_EULA: "Y"
      SA_PASSWORD: "P@ssw0rd123"
    volumes:
      - sqlserver-data:/var/opt/mssql
    networks:
      - app-network
    restart: unless-stopped

volumes:
  cosmos-data:
  service-bus-data:
  sqlserver-data:

networks:
  app-network:
    driver: bridge
```

**Environment Variables File (.env):**
```bash
SERVICE_BUS_CONNECTION_STRING=Endpoint=sb://service-bus:5672/;SharedAccessKeyName=guest;SharedAccessKey=guest
COSMOS_CONNECTION_STRING=mongodb://localhost:27017
SQL_CONNECTION_STRING=Server=sqlserver,1433;User Id=sa;Password=P@ssw0rd123;
```

**Run locally:**
```bash
# Start all services
docker-compose up -d

# View logs
docker-compose logs -f order-service
docker-compose logs -f delivery-service

# Stop services
docker-compose down

# Clean up volumes
docker-compose down -v
```

---

## KUBERNETES FUNDAMENTALS

### Kubernetes Architecture

```
┌──────────────────────────────────────────────┐
│          Kubernetes Cluster                  │
├────────────────┬─────────────────────────────┤
│ Control Plane  │       Worker Nodes          │
│ (Master)       │                             │
│                │  ┌───────────────────────┐  │
│ - API Server   │  │  Node 1               │  │
│ - Scheduler    │  │  ┌─────────────────┐  │  │
│ - Controller   │  │  │  Pod (Service1) │  │  │
│ - etcd (DB)    │  │  │  ┌────────────┐ │  │  │
│                │  │  │  │ Container  │ │  │  │
│                │  │  │  │ (.NET App) │ │  │  │
│                │  │  │  └────────────┘ │  │  │
│                │  │  └─────────────────┘  │  │
│                │  │  ┌─────────────────┐  │  │
│                │  │  │  Pod (Service2) │  │  │
│                │  │  │  ┌────────────┐ │  │  │
│                │  │  │  │ Container  │ │  │  │
│                │  │  │  │ (.NET App) │ │  │  │
│                │  │  │  └────────────┘ │  │  │
│                │  │  └─────────────────┘  │  │
│                │  └───────────────────────┘  │
│                │                             │
│                │  ┌───────────────────────┐  │
│                │  │  Node 2               │  │
│                │  │  ┌─────────────────┐  │  │
│                │  │  │  Pod (Service3) │  │  │
│                │  │  │  ┌────────────┐ │  │  │
│                │  │  │  │ Container  │ │  │  │
│                │  │  │  │ (.NET App) │ │  │  │
│                │  │  │  └────────────┘ │  │  │
│                │  │  └─────────────────┘  │  │
│                │  └───────────────────────┘  │
└────────────────┴─────────────────────────────┘
```

### Core Kubernetes Objects

#### 1. Pod (Smallest Unit)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: order-service-pod
  namespace: default
  labels:
    app: order-service
spec:
  containers:
  - name: order-service
    image: myacr.azurecr.io/order-service:v1.0
    ports:
    - containerPort: 5000
      name: http
    env:
    - name: ASPNETCORE_ENVIRONMENT
      value: "Production"
    - name: LOG_LEVEL
      value: "Information"
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "256Mi"
        cpu: "500m"
```

**Why rarely create pods directly?** Use Deployments instead (handles updates, scaling, self-healing).

#### 2. Deployment (Manage Pod Replicas)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service-deployment
  labels:
    app: order-service
spec:
  replicas: 3                    # Run 3 copies
  strategy:
    type: RollingUpdate          # Gradual update strategy
    rollingUpdate:
      maxSurge: 1                # One extra pod during update
      maxUnavailable: 0          # No pods down during update
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
      - name: order-service
        image: myacr.azurecr.io/order-service:v1.0
        ports:
        - containerPort: 5000
        env:
        - name: ASPNETCORE_URLS
          value: "http://+:5000"
        - name: ServiceBusConnectionString
          valueFrom:
            secretKeyRef:
              name: service-bus-secret
              key: connection-string
        - name: CosmosConnectionString
          valueFrom:
            secretKeyRef:
              name: cosmos-secret
              key: connection-string
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        livenessProbe:           # Restart if unhealthy
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 10
          periodSeconds: 30
          timeoutSeconds: 3
          failureThreshold: 3
        readinessProbe:          # Remove from service if not ready
          httpGet:
            path: /ready
            port: 5000
          initialDelaySeconds: 5
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 3
```

**Key fields explained:**
- `replicas`: Number of pod copies Kubernetes maintains
- `strategy`: How to update (RollingUpdate, Recreate)
- `selector`: Which pods this deployment manages
- `template`: Pod specification
- `resources`: CPU/memory requests and limits
- `livenessProbe`: Restart if fails (app crashed)
- `readinessProbe`: Remove from service if fails (temporary issue)

#### 3. Service (Network Endpoint)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: order-service
  labels:
    app: order-service
spec:
  selector:
    app: order-service
  type: ClusterIP              # Internal only (default)
  ports:
  - protocol: TCP
    port: 80                   # Service port (internal)
    targetPort: 5000           # Pod port (container)
    name: http
```

**Service types:**
- **ClusterIP**: Internal DNS only (default, efficient)
- **NodePort**: Expose on each node (30000-32767)
- **LoadBalancer**: Cloud provider assigns external IP (expensive)
- **ExternalName**: Map to external DNS

#### 4. ConfigMap (Non-Sensitive Configuration)
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: microservices
data:
  LOG_LEVEL: "Information"
  ASPNETCORE_ENVIRONMENT: "Production"
  API_TIMEOUT: "30"
  FEATURE_FLAGS: |
    {
      "newFeature": true,
      "betaFeature": false
    }
```

**Use in Deployment:**
```yaml
env:
- name: LOG_LEVEL
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: LOG_LEVEL
```

#### 5. Secret (Sensitive Data - Connection Strings, Passwords)
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: service-bus-secret
  namespace: microservices
type: Opaque
stringData:
  connection-string: "Endpoint=sb://myservicebus.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=..."
```

**Create from command line:**
```bash
kubectl create secret generic service-bus-secret \
  --from-literal=connection-string='Endpoint=sb://...' \
  -n microservices
```

**Use in Deployment:**
```yaml
env:
- name: ServiceBusConnectionString
  valueFrom:
    secretKeyRef:
      name: service-bus-secret
      key: connection-string
```

#### 6. Ingress (External HTTP/HTTPS Access)
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-gateway
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls-cert
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /orders
        pathType: Prefix
        backend:
          service:
            name: order-service
            port:
              number: 80
      - path: /delivery
        pathType: Prefix
        backend:
          service:
            name: delivery-service
            port:
              number: 80
```

**Requires ingress controller** (NGINX installed in AKS by default).

#### 7. Namespace (Multi-tenancy/Isolation)
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: microservices
  labels:
    name: microservices
```

**Deploy to namespace:**
```bash
kubectl apply -f deployment.yaml -n microservices
```

---

## AZURE CONTAINER SERVICES

### Azure Container Registry (ACR)

```bash
# Create ACR
az acr create \
  --resource-group myResourceGroup \
  --name mycontainerregistry \
  --sku Standard

# Build and push .NET image
az acr build \
  --registry mycontainerregistry \
  --image order-service:v1.0 \
  --file Dockerfile \
  ./services/OrderService

# List repositories
az acr repository list --name mycontainerregistry

# List tags
az acr repository show-tags \
  --name mycontainerregistry \
  --repository order-service
```

### Azure Kubernetes Service (AKS)

```bash
# Create AKS cluster
az aks create \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --node-count 3 \
  --vm-set-type VirtualMachineScaleSets \
  --network-plugin azure \
  --enable-managed-identity \
  --generate-ssh-keys

# Connect kubectl
az aks get-credentials \
  --resource-group myResourceGroup \
  --name myAKSCluster

# Link ACR to AKS
az aks update \
  --name myAKSCluster \
  --resource-group myResourceGroup \
  --attach-acr mycontainerregistry
```

### Azure Cosmos DB (NoSQL Database)

```bash
# Create Cosmos DB account (MongoDB API)
az cosmosdb create \
  --name mycosmosdb \
  --resource-group myResourceGroup \
  --kind MongoDB

# Get connection string
az cosmosdb keys list \
  --name mycosmosdb \
  --resource-group myResourceGroup \
  --type connection-strings

# Store in Kubernetes secret
kubectl create secret generic cosmos-secret \
  --from-literal=connection-string='mongodb://...' \
  -n microservices
```

### Azure Service Bus (Message Queue)

```bash
# Create Service Bus namespace
az servicebus namespace create \
  --resource-group myResourceGroup \
  --name myservicebus \
  --sku Standard

# Create queue
az servicebus queue create \
  --resource-group myResourceGroup \
  --namespace-name myservicebus \
  --name order-queue

# Create topic (for pub-sub)
az servicebus topic create \
  --resource-group myResourceGroup \
  --namespace-name myservicebus \
  --name order-events

# Get connection string
az servicebus namespace authorization-rule keys list \
  --resource-group myResourceGroup \
  --namespace-name myservicebus \
  --name RootManageSharedAccessKey

# Store in Kubernetes secret
kubectl create secret generic service-bus-secret \
  --from-literal=connection-string='Endpoint=sb://...' \
  -n microservices
```

### Azure Key Vault (Secrets Management)

```bash
# Create Key Vault
az keyvault create \
  --resource-group myResourceGroup \
  --name mykeyvault

# Store secrets
az keyvault secret set \
  --vault-name mykeyvault \
  --name ServiceBusConnectionString \
  --value 'Endpoint=sb://...'

az keyvault secret set \
  --vault-name mykeyvault \
  --name CosmosConnectionString \
  --value 'mongodb://...'

# Grant AKS managed identity access
az keyvault set-policy \
  --name mykeyvault \
  --object-id <aks-managed-identity-id> \
  --secret-permissions get list
```

---

## MULTI-CONTAINER SETUP

### Project Structure
```
microservices/
├── docker-compose.yml
├── .env
├── services/
│   ├── OrderService/
│   │   ├── Dockerfile
│   │   ├── Program.cs
│   │   ├── OrderService.csproj
│   │   ├── Controllers/
│   │   │   └── OrderController.cs
│   │   └── Services/
│   │       └── ServiceBusPublisher.cs
│   ├── DeliveryService/
│   │   ├── Dockerfile
│   │   ├── Program.cs
│   │   ├── DeliveryService.csproj
│   │   └── Services/
│   │       └── ServiceBusConsumer.cs
│   └── PaymentService/
│       ├── Dockerfile
│       └── Program.cs
└── k8s/
    ├── namespace.yaml
    ├── secrets.yaml
    ├── configmap.yaml
    ├── order-service-deployment.yaml
    ├── delivery-service-deployment.yaml
    ├── services.yaml
    └── ingress.yaml
```

---

## AUTHENTICATION & ACCESSIBILITY

### Authentication Types in Kubernetes

#### 1. Managed Identity (AKS - Recommended)

```bash
# Enable managed identity on AKS cluster
az aks update \
  --name myAKSCluster \
  --resource-group myResourceGroup \
  --enable-managed-identity

# Assign roles to identity
az role assignment create \
  --assignee <managed-identity-object-id> \
  --role "Cosmos DB Data Reader" \
  --scope /subscriptions/<subscription-id>/resourceGroups/myResourceGroup/providers/Microsoft.DocumentDB/databaseAccounts/mycosmosdb
```

**In .NET Code:**
```csharp
// Use managed identity to authenticate to Cosmos DB
var credential = new DefaultAzureCredential();
var client = new CosmosClient("https://mycosmosdb.documents.azure.com:443/", credential);
```

#### 2. Kubernetes Secrets (For Connection Strings)

```bash
# Create secret
kubectl create secret generic database-secret \
  --from-literal=username=admin \
  --from-literal=password=P@ssw0rd123 \
  -n microservices
```

**In Deployment:**
```yaml
env:
- name: DB_USER
  valueFrom:
    secretKeyRef:
      name: database-secret
      key: username
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: database-secret
      key: password
```

#### 3. Pod-to-Pod Communication (Network Policies)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: order-service-allow-from-gateway
  namespace: microservices
spec:
  podSelector:
    matchLabels:
      app: order-service
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 5000
```

**What this means:**
- Order service accepts traffic ONLY from NGINX Ingress controller
- All other traffic is denied (Zero Trust)

#### 4. RBAC (Role-Based Access Control)

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: order-service-sa
  namespace: microservices

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: order-service-role
  namespace: microservices
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: order-service-rolebinding
  namespace: microservices
subjects:
- kind: ServiceAccount
  name: order-service-sa
  namespace: microservices
roleRef:
  kind: Role
  name: order-service-role
  apiGroup: rbac.authorization.k8s.io
```

### Accessibility (Ingress & API Gateway)

#### NGINX Ingress Configuration

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-gateway
  namespace: microservices
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/rate-limit: "100"
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls-cert
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /orders
        pathType: Prefix
        backend:
          service:
            name: order-service
            port:
              number: 80
      - path: /delivery
        pathType: Prefix
        backend:
          service:
            name: delivery-service
            port:
              number: 80
```

---

## STEP-BY-STEP IMPLEMENTATION (.NET/C#)

### Phase 1: Local Development

#### Step 1: Create .NET Project

```bash
# Create ASP.NET Core project
dotnet new webapi -n OrderService
cd OrderService

# Add NuGet packages
dotnet add package Azure.Messaging.ServiceBus
dotnet add package Azure.Cosmos
```

#### Step 2: Create Order Service (Program.cs)

```csharp
using Azure.Messaging.ServiceBus;
using Azure.Cosmos;
using Microsoft.AspNetCore.Mvc;

var builder = WebApplication.CreateBuilder(args);

// Add services
builder.Services.AddControllers();
builder.Services.AddHealthChecks();

// Add Service Bus
var serviceBusConnectionString = builder.Configuration["ServiceBusConnectionString"];
builder.Services.AddSingleton(new ServiceBusClient(serviceBusConnectionString));

// Add Cosmos DB
var cosmosConnectionString = builder.Configuration["CosmosConnectionString"];
builder.Services.AddSingleton(new CosmosClient(cosmosConnectionString));

var app = builder.Build();

// Configure middleware
app.UseHttpsRedirection();
app.UseAuthorization();

// Health check endpoints
app.MapGet("/health", () => Results.Ok(new { status = "healthy" }))
   .WithName("Health")
   .WithOpenApi();

app.MapGet("/ready", () => Results.Ok(new { status = "ready" }))
   .WithName("Ready")
   .WithOpenApi();

// Map controllers
app.MapControllers();

app.Run();
```

**appsettings.json:**
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "ServiceBusConnectionString": "Endpoint=sb://localhost:5672/;SharedAccessKeyName=guest;SharedAccessKey=guest",
  "CosmosConnectionString": "mongodb://localhost:27017"
}
```

#### Step 3: Create Order Controller (OrderController.cs)

```csharp
using Azure.Messaging.ServiceBus;
using Microsoft.AspNetCore.Mvc;
using System.Text.Json;

[ApiController]
[Route("api/[controller]")]
public class OrderController : ControllerBase
{
    private readonly ServiceBusClient _serviceBusClient;
    private readonly ILogger<OrderController> _logger;

    public OrderController(ServiceBusClient serviceBusClient, ILogger<OrderController> logger)
    {
        _serviceBusClient = serviceBusClient;
        _logger = logger;
    }

    [HttpPost]
    public async Task<ActionResult<OrderResponse>> CreateOrder([FromBody] CreateOrderRequest request)
    {
        try
        {
            var orderId = Guid.NewGuid().ToString();
            
            // Create order object
            var order = new
            {
                OrderId = orderId,
                Customer = request.Customer,
                Items = request.Items,
                Total = request.Total,
                Timestamp = DateTime.UtcNow
            };

            // Send message to Service Bus
            var sender = _serviceBusClient.CreateSender("order-queue");
            var message = new ServiceBusMessage(JsonSerializer.Serialize(order))
            {
                ContentType = "application/json",
                Subject = "OrderCreated"
            };

            await sender.SendMessageAsync(message);
            _logger.LogInformation($"Order {orderId} sent to Service Bus");

            return Ok(new OrderResponse 
            { 
                OrderId = orderId, 
                Status = "pending" 
            });
        }
        catch (Exception ex)
        {
            _logger.LogError($"Error creating order: {ex.Message}");
            return StatusCode(500, new { error = ex.Message });
        }
    }

    [HttpGet("{orderId}")]
    public async Task<ActionResult<OrderStatusResponse>> GetOrderStatus(string orderId)
    {
        // In real scenario, fetch from Cosmos DB
        return Ok(new OrderStatusResponse 
        { 
            OrderId = orderId, 
            Status = "in_progress" 
        });
    }
}

public class CreateOrderRequest
{
    public string Customer { get; set; }
    public List<string> Items { get; set; }
    public decimal Total { get; set; }
}

public class OrderResponse
{
    public string OrderId { get; set; }
    public string Status { get; set; }
}

public class OrderStatusResponse
{
    public string OrderId { get; set; }
    public string Status { get; set; }
}
```

#### Step 4: Create Dockerfile for .NET Service

```dockerfile
# Build stage
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS builder
WORKDIR /source
COPY ["OrderService.csproj", "./"]
RUN dotnet restore "OrderService.csproj"
COPY . .
RUN dotnet publish -c Release -o /app --no-restore

# Runtime stage
FROM mcr.microsoft.com/dotnet/aspnet:7.0
WORKDIR /app
RUN useradd -m -u 1001 appuser
COPY --from=builder /app .
RUN chown -R appuser:appuser /app
USER appuser

EXPOSE 5000
ENV ASPNETCORE_URLS=http://+:5000
ENV ASPNETCORE_ENVIRONMENT=Production

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:5000/health || exit 1

ENTRYPOINT ["dotnet", "OrderService.dll"]
```

#### Step 5: Delivery Service with Service Bus Consumer

```csharp
// BackgroundService for consuming Service Bus messages
using Azure.Messaging.ServiceBus;
using Microsoft.Extensions.Hosting;

public class ServiceBusConsumerService : BackgroundService
{
    private readonly ServiceBusClient _serviceBusClient;
    private readonly ILogger<ServiceBusConsumerService> _logger;

    public ServiceBusConsumerService(ServiceBusClient serviceBusClient, ILogger<ServiceBusConsumerService> logger)
    {
        _serviceBusClient = serviceBusClient;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        var processor = _serviceBusClient.CreateProcessor("order-queue");
        processor.ProcessMessageAsync += ProcessMessageAsync;
        processor.ProcessErrorAsync += ProcessErrorAsync;

        await processor.StartProcessingAsync(stoppingToken);
        _logger.LogInformation("Service Bus consumer started");

        while (!stoppingToken.IsCancellationRequested)
        {
            await Task.Delay(1000, stoppingToken);
        }

        await processor.StopProcessingAsync();
    }

    private async Task ProcessMessageAsync(ProcessMessageEventArgs args)
    {
        string body = args.Message.Body.ToString();
        _logger.LogInformation($"Received message: {body}");
        
        // Process message (create delivery, save to DB, etc)
        
        await args.CompleteMessageAsync(args.Message);
    }

    private Task ProcessErrorAsync(ProcessErrorEventArgs args)
    {
        _logger.LogError($"Error processing message: {args.Exception.Message}");
        return Task.CompletedTask;
    }
}

// Register in Program.cs
builder.Services.AddHostedService<ServiceBusConsumerService>();
```

#### Step 6: Run Locally with Docker Compose

```bash
# Build images
docker-compose build

# Start services
docker-compose up -d

# View logs
docker-compose logs -f order-service
docker-compose logs -f delivery-service

# Test
curl -X POST http://localhost:5000/api/order \
  -H "Content-Type: application/json" \
  -d '{
    "customer": "John Doe",
    "items": ["item1", "item2"],
    "total": 99.99
  }'
```

---

## SERVICE BUS COMMUNICATION IN C#

### Publishing Messages (Order Service)

```csharp
public class ServiceBusPublisher
{
    private readonly ServiceBusClient _client;
    private readonly ILogger<ServiceBusPublisher> _logger;

    public ServiceBusPublisher(ServiceBusClient client, ILogger<ServiceBusPublisher> logger)
    {
        _client = client;
        _logger = logger;
    }

    public async Task PublishOrderCreatedAsync(Order order)
    {
        try
        {
            var sender = _client.CreateSender("order-queue");
            
            var message = new ServiceBusMessage(JsonSerializer.Serialize(order))
            {
                ContentType = "application/json",
                Subject = "OrderCreated",
                CorrelationId = order.OrderId
            };

            await sender.SendMessageAsync(message);
            _logger.LogInformation($"Published OrderCreated event for order {order.OrderId}");
        }
        catch (Exception ex)
        {
            _logger.LogError($"Error publishing message: {ex.Message}");
            throw;
        }
    }
}
```

### Consuming Messages (Delivery Service)

```csharp
public class OrderMessageProcessor
{
    private readonly CosmosClient _cosmosClient;
    private readonly ILogger<OrderMessageProcessor> _logger;

    public OrderMessageProcessor(CosmosClient cosmosClient, ILogger<OrderMessageProcessor> logger)
    {
        _cosmosClient = cosmosClient;
        _logger = logger;
    }

    public async Task ProcessOrderCreatedAsync(string messageBody)
    {
        try
        {
            var order = JsonSerializer.Deserialize<Order>(messageBody);
            
            // Create delivery record in Cosmos DB
            var container = _cosmosClient.GetDatabase("deliveries")
                                        .GetContainer("shipments");
            
            var delivery = new Delivery
            {
                Id = Guid.NewGuid().ToString(),
                OrderId = order.OrderId,
                Status = "in_transit",
                CreatedAt = DateTime.UtcNow,
                EstimatedDelivery = DateTime.UtcNow.AddDays(3)
            };

            await container.CreateItemAsync(delivery);
            _logger.LogInformation($"Created delivery for order {order.OrderId}");
        }
        catch (Exception ex)
        {
            _logger.LogError($"Error processing message: {ex.Message}");
            throw;
        }
    }
}
```

---

## PHASE 2: DEPLOY TO AZURE

### Step 1: Build and Push Images

```bash
# Build images
docker build -t myacr.azurecr.io/order-service:v1.0 ./services/OrderService
docker build -t myacr.azurecr.io/delivery-service:v1.0 ./services/DeliveryService

# Push to ACR
docker push myacr.azurecr.io/order-service:v1.0
docker push myacr.azurecr.io/delivery-service:v1.0

# Or use ACR build (builds in Azure)
az acr build --registry myacr --image order-service:v1.0 ./services/OrderService
az acr build --registry myacr --image delivery-service:v1.0 ./services/DeliveryService
```

### Step 2: Create Kubernetes Manifests

**k8s/namespace.yaml:**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: microservices
  labels:
    name: microservices
```

**k8s/secrets.yaml:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: service-bus-secret
  namespace: microservices
type: Opaque
stringData:
  connection-string: "Endpoint=sb://myservicebus.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=..."

---
apiVersion: v1
kind: Secret
metadata:
  name: cosmos-secret
  namespace: microservices
type: Opaque
stringData:
  connection-string: "mongodb://mycosmosdb.mongo.cosmos.azure.com:10255/admin?ssl=true&retryWrites=false&replicaSet=globaldb&maxIdleTimeMS=120000&appName=@mycosmosdb@;..."
```

**k8s/order-service-deployment.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: microservices
  labels:
    app: order-service
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
      - name: order-service
        image: myacr.azurecr.io/order-service:v1.0
        imagePullPolicy: Always
        ports:
        - containerPort: 5000
          name: http
        env:
        - name: ASPNETCORE_URLS
          value: "http://+:5000"
        - name: ASPNETCORE_ENVIRONMENT
          value: "Production"
        - name: ServiceBusConnectionString
          valueFrom:
            secretKeyRef:
              name: service-bus-secret
              key: connection-string
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 10
          periodSeconds: 30
          timeoutSeconds: 3
        readinessProbe:
          httpGet:
            path: /ready
            port: 5000
          initialDelaySeconds: 5
          periodSeconds: 10
          timeoutSeconds: 3

---
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: microservices
  labels:
    app: order-service
spec:
  selector:
    app: order-service
  type: ClusterIP
  ports:
  - protocol: TCP
    port: 80
    targetPort: 5000
    name: http
```

**k8s/delivery-service-deployment.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: delivery-service
  namespace: microservices
  labels:
    app: delivery-service
spec:
  replicas: 2
  selector:
    matchLabels:
      app: delivery-service
  template:
    metadata:
      labels:
        app: delivery-service
    spec:
      containers:
      - name: delivery-service
        image: myacr.azurecr.io/delivery-service:v1.0
        imagePullPolicy: Always
        ports:
        - containerPort: 5001
          name: http
        env:
        - name: ASPNETCORE_URLS
          value: "http://+:5001"
        - name: ASPNETCORE_ENVIRONMENT
          value: "Production"
        - name: ServiceBusConnectionString
          valueFrom:
            secretKeyRef:
              name: service-bus-secret
              key: connection-string
        - name: CosmosConnectionString
          valueFrom:
            secretKeyRef:
              name: cosmos-secret
              key: connection-string
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 5001
          initialDelaySeconds: 10
          periodSeconds: 30

---
apiVersion: v1
kind: Service
metadata:
  name: delivery-service
  namespace: microservices
  labels:
    app: delivery-service
spec:
  selector:
    app: delivery-service
  type: ClusterIP
  ports:
  - protocol: TCP
    port: 80
    targetPort: 5001
    name: http
```

**k8s/ingress.yaml:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-gateway
  namespace: microservices
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /api/order
        pathType: Prefix
        backend:
          service:
            name: order-service
            port:
              number: 80
      - path: /api/delivery
        pathType: Prefix
        backend:
          service:
            name: delivery-service
            port:
              number: 80
```

### Step 3: Deploy to AKS

```bash
# Create namespace
kubectl apply -f k8s/namespace.yaml

# Create secrets
kubectl apply -f k8s/secrets.yaml

# Deploy services
kubectl apply -f k8s/order-service-deployment.yaml
kubectl apply -f k8s/delivery-service-deployment.yaml
kubectl apply -f k8s/ingress.yaml

# Verify deployment
kubectl get pods -n microservices
kubectl get services -n microservices
kubectl get ingress -n microservices

# Watch deployment progress
kubectl rollout status deployment/order-service -n microservices
kubectl rollout status deployment/delivery-service -n microservices
```

---

## TESTING & TROUBLESHOOTING

### Common Issues & Solutions

#### 1. Pod Cannot Connect to Azure Service Bus
```bash
# Check pod logs
kubectl logs <pod-name> -n microservices

# Check if secret exists
kubectl get secrets -n microservices

# Describe pod for events
kubectl describe pod <pod-name> -n microservices

# Verify connection string format
kubectl get secret service-bus-secret -n microservices -o jsonpath='{.data.connection-string}' | base64 -d
```

#### 2. Images Not Pulling from ACR
```bash
# Check image pull status
kubectl describe pod <pod-name> -n microservices

# Re-attach ACR to AKS
az aks update --name myAKSCluster --resource-group myResourceGroup --attach-acr myacr

# Verify image path
kubectl get deployment order-service -n microservices -o yaml | grep image
```

#### 3. Service Not Accessible via Ingress
```bash
# Check ingress
kubectl get ingress api-gateway -n microservices -o yaml

# Check ingress controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx

# Port forward to test directly
kubectl port-forward svc/order-service 8080:80 -n microservices
curl http://localhost:8080/health
```

#### 4. Cosmos DB Connection Fails
```bash
# Test from pod
kubectl exec -it <pod-name> -n microservices -- /bin/bash

# Install curl in pod
apt-get update && apt-get install -y curl

# Test DNS resolution
nslookup mycosmosdb.mongo.cosmos.azure.com

# Check connection string
echo $CosmosConnectionString
```

### Debugging Commands

```bash
# View logs
kubectl logs <pod-name> -n microservices -f

# Enter container
kubectl exec -it <pod-name> -n microservices -- /bin/bash

# Port forward
kubectl port-forward pod/<pod-name> 5000:5000 -n microservices

# Check events
kubectl get events -n microservices --sort-by='.lastTimestamp'

# Monitor resources
kubectl top pods -n microservices
kubectl top nodes

# Test service-to-service communication
kubectl exec -it <order-service-pod> -n microservices -- \
  curl http://delivery-service/api/delivery/12345
```

### Monitoring & Observability

```bash
# Enable monitoring addon
az aks enable-addons \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --addons monitoring

# Stream logs from multiple pods
kubectl logs -l app=order-service -n microservices -f

# Check pod metrics
kubectl top pods -n microservices

# Get detailed pod information
kubectl describe pod <pod-name> -n microservices
```

---

## SUMMARY & LEARNING CHECKLIST

### Key Concepts Covered

1. **Docker Layers**: Each Dockerfile instruction creates a cacheable layer
2. **Multi-stage Builds**: Reduces final image size (SDK → Runtime)
3. **Layer Caching**: Speed up builds by not rebuilding unchanged layers
4. **.NET/C# in Containers**: ASP.NET Core, health checks, configuration
5. **Kubernetes Objects**: Pods, Deployments, Services, Secrets, Ingress
6. **Azure Services**: ACR, AKS, Cosmos DB, Service Bus
7. **Service-to-Service Communication**: Asynchronous via Service Bus
8. **Authentication**: Managed Identity, Secrets, RBAC
9. **Deployment Strategies**: Rolling updates, readiness/liveness probes

### Learning Progression

- [ ] Understand Docker layers and caching
- [ ] Build multi-stage Dockerfiles for .NET
- [ ] Run docker-compose with multiple services
- [ ] Understand Kubernetes objects (Pods, Deployments, Services)
- [ ] Write and deploy Kubernetes manifests
- [ ] Deploy to local Kubernetes (Docker Desktop)
- [ ] Create AKS cluster and configure kubectl
- [ ] Push .NET images to ACR
- [ ] Deploy multi-container application to AKS
- [ ] Configure Azure Service Bus for inter-service communication
- [ ] Set up Cosmos DB and access from AKS pods
- [ ] Implement C# Service Bus publisher/consumer
- [ ] Configure secrets and authentication
- [ ] Set up Ingress for external access
- [ ] Monitor and troubleshoot deployments

### What Was Missing & Now Added

✅ **Docker Layer-by-layer explanation** with caching mechanism
✅ **.NET/C# specific examples** (ASP.NET Core, Service Bus SDK)
✅ **Service Bus communication in C#** (Publisher/Consumer patterns)
✅ **Multi-stage build details** for .NET
✅ **Kubernetes security** (Managed Identity, RBAC, Network Policies)
✅ **Complete code examples** (Controllers, Background Services)
✅ **Cosmos DB integration** in C#
✅ **Health checks** in .NET
✅ **Environment configuration** in K8s
✅ **Deployment strategies** (Rolling updates)

---

## ADDITIONAL RESOURCES

### Official Documentation
- Kubernetes: https://kubernetes.io/docs/
- Azure AKS: https://learn.microsoft.com/en-us/azure/aks/
- .NET on Docker: https://docs.microsoft.com/dotnet/architecture/microservices/container-docker-introduction/
- Azure Service Bus: https://learn.microsoft.com/en-us/azure/service-bus-messaging/
- Azure Cosmos DB: https://learn.microsoft.com/en-us/azure/cosmos-db/

### .NET/C# Resources
- Azure SDK for .NET: https://github.com/Azure/azure-sdk-for-net
- ASP.NET Core Documentation: https://learn.microsoft.com/en-us/aspnet/core/
- Microservices .NET: https://dotnet.microsoft.com/en-us/learn/aspnet/microservices-architecture

### Useful Tools
- kubectl: Kubernetes CLI
- Docker CLI: Container management
- Azure CLI: Azure resource management
- k9s: Terminal UI for Kubernetes
- Lens: Kubernetes IDE
- Azure Storage Explorer

### Best Practices
- Use resource requests/limits for scheduling
- Implement liveness & readiness probes
- Use ConfigMaps for non-sensitive config
- Use Secrets for sensitive data
- Enable RBAC for fine-grained access control
- Use network policies for Zero Trust networking
- Implement comprehensive logging and monitoring
- Use health check endpoints in your APIs
- Tag images with semantic versioning
- Use private registries (ACR) for security
