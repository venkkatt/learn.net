# Docker, Docker Compose, and Kubernetes: A Comprehensive Guide for .NET Developers

## Table of Contents
1. [Docker Fundamentals](#docker-fundamentals)
2. [Docker Compose](#docker-compose)
3. [Kubernetes Basics](#kubernetes-basics)
4. [Cost Analysis](#cost-analysis)
5. [Azure Integration](#azure-integration)
6. [Interview Questions and Answers](#interview-questions-and-answers)

---

## Docker Fundamentals

### What Problems Does Docker Solve?

Docker addresses several critical software development challenges:

#### 1. **Environment Consistency ("It works on my machine")**
- Developers work on their local machines, but code fails in production due to dependency mismatches
- Docker packages the entire application with all dependencies, ensuring identical behavior across environments

#### 2. **Dependency Hell**
- Different applications require different library versions
- Traditional VMs and shared servers create conflicts
- Containers isolate dependencies completely

#### 3. **Slow Deployment**
- Traditional deployments take hours (provisioning servers, installing runtime, configuring environments)
- Docker containers deploy in seconds

#### 4. **Resource Inefficiency**
- Virtual machines require full operating systems (~2GB each)
- Containers share the host OS kernel, using ~10-50MB each
- Allows running hundreds of containers on a single server

#### 5. **Scalability Issues**
- Difficult to scale individual application components independently
- Containers enable horizontal scaling (spinning up multiple instances)

### How Docker Solves These Problems

#### Docker Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Host Operating System                 │
│                                                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  Container   │  │  Container   │  │  Container   │  │
│  │  (App + Dep) │  │  (App + Dep) │  │  (App + Dep) │  │
│  │              │  │              │  │              │  │
│  │  App Code    │  │  App Code    │  │  App Code    │  │
│  │  Libraries   │  │  Libraries   │  │  Libraries   │  │
│  │  Runtime     │  │  Runtime     │  │  Runtime     │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
│         ↓                  ↓                  ↓            │
│  ┌─────────────────────────────────────────────────────┐ │
│  │              Docker Engine                          │ │
│  │  - Container Runtime                               │ │
│  │  - Image Management                                │ │
│  │  - Network Management                              │ │
│  └─────────────────────────────────────────────────────┘ │
│         ↓                                                  │
│  ┌─────────────────────────────────────────────────────┐ │
│  │              Shared OS Kernel                       │ │
│  │  (All containers share this - Not VMs!)            │ │
│  └─────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

#### Key Concepts

**Images**: Blueprint/template containing application code, libraries, and runtime
```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:7.0
WORKDIR /app
COPY . .
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

**Containers**: Running instance of an image (like the difference between a class and object)

**Layers**: Docker images are built in layers. Each instruction creates a new layer
- Enables caching for faster builds
- Only changed layers need rebuilding

**Dockerfile**: Text file with instructions to build an image

### Docker Advantages

| Advantage | Benefit |
|-----------|---------|
| **Portability** | Run anywhere - local machine, cloud, data center |
| **Consistency** | Development = staging = production |
| **Efficiency** | 10-100x faster startup, 95% less resource usage than VMs |
| **Isolation** | Container failure doesn't affect others |
| **Scalability** | Spin up/down instances instantly |
| **DevOps** | Faster CI/CD pipelines, easier rollbacks |
| **Cost** | Use fewer servers, better resource utilization |

### Docker Disadvantages

| Disadvantage | Mitigation |
|-------------|-----------|
| **Learning Curve** | Requires understanding of containerization concepts |
| **Security** | Shared kernel means kernel vulnerabilities affect all containers. Run as non-root user |
| **Stateful Data** | Containers are ephemeral; need volumes for persistent data |
| **Debugging** | Can be harder than traditional development |
| **Orchestration** | Single container management is simple, but managing hundreds requires Kubernetes |

### Practical .NET Example

**Dockerfile for .NET Core Web API**
```dockerfile
# Build stage
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS builder
WORKDIR /src
COPY ["MyApi.csproj", "."]
RUN dotnet restore "MyApi.csproj"
COPY . .
RUN dotnet build "MyApi.csproj" -c Release -o /app/build

# Publish stage
FROM builder AS publish
RUN dotnet publish "MyApi.csproj" -c Release -o /app/publish

# Runtime stage
FROM mcr.microsoft.com/dotnet/aspnet:7.0
WORKDIR /app
COPY --from=publish /app/publish .

# Security: Run as non-root user
RUN useradd -m -u 1001 appuser && chown -R appuser /app
USER appuser

EXPOSE 8080
ENTRYPOINT ["dotnet", "MyApi.dll"]
```

**Build and Run**
```bash
# Build image
docker build -t myapi:1.0 .

# Run container
docker run -d -p 8080:8080 --name myapi-instance myapi:1.0

# View logs
docker logs myapi-instance

# Stop container
docker stop myapi-instance
```

---

## Docker Compose

### What is Docker Compose?

Docker Compose solves the problem of managing **multi-container applications** with a single command. Instead of:
```bash
docker run -d -e DB_HOST=db -e DB_USER=root mysql:8
docker run -d -p 80:8080 --link mysql-db myapi:1.0
docker run -d -p 6379:6379 redis:latest
```

You define everything in a YAML file and run:
```bash
docker-compose up -d
```

### How Docker Compose Solves Problems

1. **Container Orchestration**: Manages multiple containers as a single unit
2. **Networking**: Automatically creates a network and enables DNS-based service discovery
3. **Configuration Management**: Environment variables and secrets in one place
4. **Dependency Management**: Controls container startup order with `depends_on`
5. **Volume Management**: Persistent storage configuration
6. **Reproducibility**: Same setup across dev, test, and production

### Docker Compose Example: .NET API + SQL Server + Redis

```yaml
version: '3.9'

services:
  # .NET Core Web API
  api:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: myapi
    ports:
      - "8080:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__DefaultConnection=Server=db,1433;Initial Catalog=MyDb;User Id=sa;Password=@Sql2022;Encrypt=false
      - REDIS_HOST=redis
      - REDIS_PORT=6379
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # SQL Server Database
  db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    container_name: mydb
    environment:
      - SA_PASSWORD=@Sql2022
      - ACCEPT_EULA=Y
    ports:
      - "1433:1433"
    volumes:
      - db-data:/var/opt/mssql
      - ./init-db.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - app-network
    healthcheck:
      test: ["/opt/mssql-tools/bin/sqlcmd", "-S", "localhost", "-U", "sa", "-P", "@Sql2022", "-Q", "SELECT 1"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Redis Cache
  redis:
    image: redis:7-alpine
    container_name: myredis
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    networks:
      - app-network
    command: redis-server --appendonly yes

volumes:
  db-data:
  redis-data:

networks:
  app-network:
    driver: bridge
```

### Key Features

**Service Discovery**: Containers communicate using service names
```csharp
// In .NET code, connect to database using service name
var connectionString = "Server=db,1433;Initial Catalog=MyDb;...";

// Redis connection
var redis = ConnectionMultiplexer.Connect("redis:6379");
```

**Health Checks**: Define how to verify container health
```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 40s
```

**Volume Management**: Persistent and shared data
```yaml
volumes:
  - db-data:/var/opt/mssql          # Named volume
  - ./config.json:/app/config.json  # Bind mount
  - /tmp/cache:/app/cache           # Anonymous volume
```

### Common Docker Compose Commands

```bash
# Start all services
docker-compose up -d

# View logs
docker-compose logs -f api

# Stop services
docker-compose down

# Rebuild images
docker-compose up -d --build

# Scale service
docker-compose up -d --scale api=3

# Execute command in container
docker-compose exec api dotnet ef migrations add Initial

# Remove volumes too
docker-compose down -v
```

---

## Kubernetes Basics

### What is Kubernetes?

Kubernetes (K8s) is a **container orchestration platform** that automates deployment, scaling, and management of containerized applications. While Docker Compose handles multiple containers on a single machine, Kubernetes manages containers across **clusters of machines**.

### Problems Kubernetes Solves

1. **Multi-Node Deployment**: Deploy containers across multiple servers
2. **Automatic Scaling**: Scale up when load increases, scale down when it decreases
3. **Self-Healing**: Automatically restart failed containers
4. **Rolling Updates**: Update applications without downtime
5. **Load Balancing**: Distribute traffic across container replicas
6. **Resource Optimization**: Efficiently allocate CPU and memory
7. **High Availability**: Ensure service availability despite failures

### Kubernetes Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                         │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              Control Plane (Master)                   │   │
│  │  ┌─────────────┐  ┌──────────────┐  ┌────────────┐  │   │
│  │  │ API Server  │  │ etcd (Store) │  │ Scheduler  │  │   │
│  │  └─────────────┘  └──────────────┘  └────────────┘  │   │
│  │  ┌──────────────────────────────────────────────┐   │   │
│  │  │  Controller Manager (maintains desired state)│   │   │
│  │  └──────────────────────────────────────────────┘   │   │
│  └──────────────────────────────────────────────────────┘   │
│                           ↓                                  │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              Worker Nodes                            │   │
│  │                                                       │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────┐   │   │
│  │  │ Node 1       │  │ Node 2       │  │ Node 3   │   │   │
│  │  │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────┐ │   │   │
│  │  │ │ Pod      │ │  │ │ Pod      │ │  │ │ Pod  │ │   │   │
│  │  │ │ Container│ │  │ │Container │ │  │ │Cont..│ │   │   │
│  │  │ └──────────┘ │  │ └──────────┘ │  │ └──────┘ │   │   │
│  │  │ kubelet      │  │ kubelet      │  │ kubelet  │   │   │
│  │  └──────────────┘  └──────────────┘  └──────────┘   │   │
│  │                                                       │   │
│  └──────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

### Core Kubernetes Objects

#### 1. **Pod** - Smallest deployable unit (usually one container)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: app
    image: myapi:1.0
    ports:
    - containerPort: 8080
```

#### 2. **Deployment** - Manages replicas and updates
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  replicas: 3                    # Run 3 instances
  selector:
    matchLabels:
      app: myapi
  template:
    metadata:
      labels:
        app: myapi
    spec:
      containers:
      - name: api
        image: myapi:1.0
        ports:
        - containerPort: 8080
        env:
        - name: DB_HOST
          value: db-service
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

#### 3. **Service** - Network abstraction for pods
```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  type: LoadBalancer              # Expose externally
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: myapi
```

#### 4. **ConfigMap & Secret** - Configuration management
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
data:
  LOG_LEVEL: "Info"
  ASPNETCORE_ENVIRONMENT: "Production"

---
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  connection-string: "Server=db;User Id=sa;Password=secret"
```

#### 5. **StatefulSet** - For stateful applications (databases)
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql-stateful
spec:
  serviceName: mysql-service
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: mysql-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

#### 6. **PersistentVolume & PersistentVolumeClaim** - Storage
```yaml
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-data
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

#### 7. **Ingress** - External HTTP/HTTPS routing
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
```

### Kubernetes Scalability

**Horizontal Pod Autoscaler (HPA)**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

### Kubernetes vs Docker Compose

| Feature | Docker Compose | Kubernetes |
|---------|----------------|-----------|
| **Scope** | Single machine | Cluster of machines |
| **Use Case** | Local development, testing | Production deployments |
| **Complexity** | Simple | Complex |
| **Scaling** | Manual | Automatic |
| **Self-Healing** | No | Yes |
| **Rolling Updates** | No | Yes |
| **Learning Curve** | Easy | Steep |

---

## Cost Analysis

### Docker vs Traditional Hosting

**Why Docker Reduces Costs**

1. **Resource Efficiency**: 10-100x better than VMs
2. **Higher Density**: Pack more containers per server
3. **Faster Deployment**: Reduces operational overhead
4. **Rapid Scaling**: Pay for what you use, when you use it

### Cost Comparison Table

| Metric | Traditional VM | Docker Container |
|--------|----------------|------------------|
| **Size** | 2-5 GB | 20-200 MB |
| **Startup Time** | 30-60 seconds | 1-5 seconds |
| **Memory Overhead** | 512 MB - 1 GB per VM | 10-50 MB per container |
| **Instances per Server** | 4-10 VMs | 50-100+ containers |

### Real-World Cost Scenarios

**Small Application (5 services)**
- **Traditional Hosting**: $300-500/month (dedicated servers)
- **Docker (Single machine)**: $50-150/month (VM hosting)
- **Kubernetes (Managed - AKS)**: $150-300/month (includes management overhead)

**Large Application (50+ services)**
- **Traditional Hosting**: $3,000-5,000/month (multiple servers)
- **Docker (Multiple machines)**: $1,000-2,000/month
- **Kubernetes (Managed - AKS)**: $800-1,500/month (economies of scale + automation)

**Cost Savings Research**: Forrester found enterprises adopting Docker reduced infrastructure costs by **66%** and increased developer productivity by **43%**.

### Kubernetes Cost Optimization

1. **Use Auto-scaling**: Scale down during off-hours
2. **Use Spot Instances**: Up to 70% cheaper but can be interrupted
3. **Right-size Resources**: Set accurate CPU/memory requests
4. **Use Namespaces**: Isolate and control resource usage
5. **Monitor and Alert**: Identify waste early

---

## Azure Integration

### Azure Container Registry (ACR)

Stores Docker images in Azure's private registry.

```bash
# Login to ACR
az acr login --name myregistry

# Tag image
docker tag myapi:1.0 myregistry.azurecr.io/myapi:1.0

# Push image
docker push myregistry.azurecr.io/myapi:1.0

# View images
az acr repository list --name myregistry
```

### Azure Kubernetes Service (AKS)

**Create AKS cluster with ACR integration**
```bash
# Create resource group
az group create --name myResourceGroup --location eastus

# Create AKS cluster
az aks create \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --node-count 3 \
  --vm-set-type VirtualMachineScaleSets \
  --attach-acr myregistry \
  --generate-ssh-keys

# Get credentials
az aks get-credentials --resource-group myResourceGroup --name myAKSCluster

# Verify nodes
kubectl get nodes
```

**Deploy .NET Application to AKS**
```bash
# Create secret for database
kubectl create secret generic db-secret \
  --from-literal=connection-string='Server=mydb.database.windows.net;...'

# Apply deployment
kubectl apply -f deployment.yaml

# Check deployment
kubectl get deployments
kubectl get pods
kubectl get svc
```

### AKS Pricing

| Tier | Cost | Use Case |
|------|------|----------|
| **Free** | $0 | Development/testing, <10 nodes |
| **Standard** | ~$73/month per cluster | Production, SLA guaranteed (99.95%) |
| **Premium** | ~$438/month per cluster | Enterprise, extended support |

**AKS Pricing Model**:
- Free control plane (management is free)
- Pay only for worker node VMs
- Pay for storage, networking, and managed services

---

## Interview Questions and Answers

### Docker Interview Questions

#### Q1: What is the difference between a Docker Image and a Docker Container?

**Answer**:
- **Image**: A blueprint/template containing application code, libraries, runtime, and configuration. It's immutable.
- **Container**: A running instance of an image. It's mutable and has its own filesystem, network, and processes.

Analogy: Image = Class, Container = Object instance

```bash
# Create image from Dockerfile
docker build -t myapp:1.0 .

# Create and run container from image
docker run -d myapp:1.0   # Container is running instance
docker run -d myapp:1.0   # Another instance of same image
```

#### Q2: Explain Docker layers and why they matter

**Answer**:
Each instruction in a Dockerfile creates a layer. Docker uses layer caching to speed up builds.

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:7.0                 # Layer 1
WORKDIR /src                                          # Layer 2
COPY MyApi.csproj .                                   # Layer 3
RUN dotnet restore                                    # Layer 4 (most expensive)
COPY . .                                              # Layer 5
RUN dotnet publish -c Release -o /app/publish        # Layer 6
```

**Why it matters**:
- If only your source code changes (Layer 5), Docker reuses Layers 1-4
- This dramatically speeds up rebuilds
- Layer caching is crucial for CI/CD pipelines

#### Q3: What are Docker volumes and why would you use them?

**Answer**:
Volumes provide persistent storage for containers. There are three types:

1. **Named Volumes** (Recommended)
   - Managed by Docker
   - Survives container deletion
   - Use case: Databases, important data

```bash
docker volume create mydata
docker run -v mydata:/data myimage
```

2. **Bind Mounts**
   - Direct host path
   - Use case: Development (code editing)

```bash
docker run -v /host/path:/container/path myimage
```

3. **tmpfs Mount**
   - In-memory storage
   - Temporary data only
   - Use case: Cache, session data

```bash
docker run --tmpfs /tmp myimage
```

#### Q4: Explain Docker networking. How do containers communicate?

**Answer**:
Docker creates isolated networks. By default, containers can reach each other by container name.

**Networks**:
- **Bridge**: Default, containers on same network can communicate
- **Host**: Container uses host network (not isolated)
- **None**: No networking
- **Overlay**: For Docker Swarm

```bash
# Create custom network
docker network create mynet

# Run containers on same network
docker run -d --network mynet --name db mysql:8
docker run -d --network mynet --name app myapi:1.0

# From app container, reach db using hostname
curl http://db:3306   # Works due to Docker DNS
```

#### Q5: What's a .dockerignore file and why is it important?

**Answer**:
`.dockerignore` specifies files/directories to exclude from Docker build context (similar to `.gitignore`).

```dockerignore
node_modules
.git
.gitignore
*.md
.env
bin/
obj/
```

**Why**:
- Reduces image size
- Speeds up build (less to copy)
- Improves security (exclude secrets, credentials)

#### Q6: How would you secure a Docker image?

**Answer**:
1. Run as non-root user
2. Use minimal base images
3. Scan for vulnerabilities
4. Don't hardcode secrets
5. Use read-only filesystems where possible

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:7.0

# Create non-root user
RUN useradd -m -u 1001 appuser

WORKDIR /app
COPY . .

# Switch to non-root
USER appuser

EXPOSE 8080
ENTRYPOINT ["dotnet", "MyApi.dll"]
```

```bash
# Scan for vulnerabilities
docker scan myapi:1.0

# Run as non-root from CLI
docker run -u 1001 myimage
```

#### Q7: Explain multi-stage builds in Docker

**Answer**:
Multi-stage builds use multiple `FROM` statements to reduce final image size by excluding build tools.

```dockerfile
# Stage 1: Build
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS builder
WORKDIR /src
COPY MyApi.csproj .
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app/publish

# Stage 2: Runtime (only includes runtime, not SDK)
FROM mcr.microsoft.com/dotnet/aspnet:7.0
WORKDIR /app
COPY --from=builder /app/publish .
ENTRYPOINT ["dotnet", "MyApi.dll"]
```

**Why**:
- Final image: ~150MB (without SDK)
- Without multi-stage: ~500MB (with SDK)
- 70% size reduction!

#### Q8: What's the difference between CMD and ENTRYPOINT?

**Answer**:
- **ENTRYPOINT**: The command that runs when container starts (hard to override)
- **CMD**: Default arguments to ENTRYPOINT (easy to override)

```dockerfile
# Best practice: Use both
ENTRYPOINT ["dotnet"]
CMD ["MyApi.dll"]

# Container runs: dotnet MyApi.dll
```

```bash
# Override CMD only
docker run myimage app2.dll    # Runs: dotnet app2.dll

# Override ENTRYPOINT too
docker run --entrypoint /bin/bash myimage   # Runs bash
```

#### Q9: How would you debug a Docker container?

**Answer**:
```bash
# View logs
docker logs container-name
docker logs -f container-name    # Follow logs

# Execute command in running container
docker exec -it container-name /bin/bash
docker exec container-name ps aux

# Inspect container details
docker inspect container-name

# View resource usage
docker stats container-name

# Check health status
docker ps --filter "health=unhealthy"
```

#### Q10: Explain health checks in Docker

**Answer**:
Health checks verify container is functioning properly.

```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1
```

**States**:
- **healthy**: Last check passed
- **unhealthy**: Multiple checks failed
- **starting**: Initial period before first check

```bash
# View health status
docker ps   # Shows health column
```

---

### Docker Compose Interview Questions

#### Q1: Why would you use Docker Compose instead of plain Docker commands?

**Answer**:
Docker Compose manages multi-container applications with a single file.

**Without Compose** (tedious):
```bash
docker network create app-net
docker run -d --network app-net --name db mysql:8 -e MYSQL_PASSWORD=secret
docker run -d --network app-net -p 8080:8080 --link db myapi:1.0
docker run -d --network app-net -p 6379:6379 redis:7
```

**With Compose** (simple):
```bash
docker-compose up -d
```

**Benefits**:
- Single YAML file
- Automatic networking
- Automatic service discovery
- Dependency management
- Environment variables centralized
- Reproducible setup

#### Q2: Explain the docker-compose.yml structure

**Answer**:
```yaml
version: '3.9'          # API version

services:              # Define containers
  api:                 # Service name (used for DNS)
    build:            # Build from Dockerfile
      context: .
      dockerfile: Dockerfile
    ports:            # Map ports: host:container
      - "8080:8080"
    environment:      # Environment variables
      - DB_HOST=db
      - REDIS_HOST=redis
    depends_on:       # Service dependencies
      - db
      - redis
    networks:         # Which network to join
      - backend
    volumes:          # Storage
      - ./data:/app/data
    healthcheck:      # Health monitoring

volumes:              # Define named volumes
  db-data:

networks:             # Define networks
  backend:
    driver: bridge
```

#### Q3: How do services discover each other in Docker Compose?

**Answer**:
Docker Compose creates a built-in DNS that maps service names to container IPs.

```yaml
services:
  api:
    environment:
      - DB_HOST=db              # Service name 'db'
  db:
    image: mysql:8
```

In application code:
```csharp
// Connect using service name
var connectionString = "Server=db,1433;Initial Catalog=MyDb;...";
var connection = new SqlConnection(connectionString);
```

This works because:
1. Docker Compose creates network `project_default`
2. Both containers join this network
3. Docker DNS resolves 'db' to db container's IP
4. All containers can reach each other by service name

#### Q4: Explain depends_on and service health conditions

**Answer**:
`depends_on` ensures services start in correct order, but doesn't guarantee they're ready.

```yaml
services:
  api:
    depends_on:
      db:
        condition: service_healthy    # Wait until healthy
      cache:
        condition: service_started    # Just wait for start

  db:
    healthcheck:
      test: ["CMD", "mysql", "-uroot", "-ppassword"]
      interval: 10s
      timeout: 5s
      retries: 5
```

**Conditions**:
- `service_started`: Container started (default)
- `service_healthy`: Healthcheck passed
- `service_completed_successfully`: Container finished successfully

**Best Practice**: Always use `service_healthy` for databases.

#### Q5: How would you scale services in Docker Compose?

**Answer**:
```bash
# Scale a service
docker-compose up -d --scale api=3

# View all instances
docker-compose ps   # Shows api_1, api_2, api_3
```

**Limitations**:
- Can't map specific ports for multiple instances (ports conflict)
- Solution: Use LoadBalancer service or reverse proxy (Nginx)

**Better approach**:
```yaml
services:
  api:
    build: .
    expose:           # Don't publish, Nginx accesses internally
      - 8080

  nginx:
    image: nginx:latest
    ports:
      - "80:80"
    depends_on:
      - api
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf

  redis:
    image: redis:7
    deploy:
      replicas: 1   # Single instance
```

---

### Kubernetes Interview Questions

#### Q1: Explain the difference between Pods, Deployments, StatefulSets, and DaemonSets

**Answer**:

| Object | Purpose | Use Case |
|--------|---------|----------|
| **Pod** | Smallest unit, usually 1 container | Direct use rare; usually managed by other objects |
| **Deployment** | Manages stateless replicas, rolling updates | Web apps, APIs, stateless services |
| **StatefulSet** | Manages stateful replicas, stable identity | Databases, message queues, file systems |
| **DaemonSet** | Runs on every node | Monitoring, logging, networking agents |

**Example Use Cases**:
```yaml
# Deployment: Stateless API (multiple replicas OK)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3    # Can be any number, any node

---
# StatefulSet: Database (needs stable identity)
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  replicas: 3    # Creates mysql-0, mysql-1, mysql-2 (stable names)

---
# DaemonSet: Logging (one per node)
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
spec:
  template:      # Runs on every node automatically
```

#### Q2: Explain ReplicaSet vs Deployment

**Answer**:

**ReplicaSet**: Low-level object that ensures N replicas of pods are running
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: api-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:      # Pod template
```

**Deployment**: Higher-level object that manages ReplicaSets and enables rolling updates
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
  strategy:      # Rolling update strategy
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # One extra pod during update
      maxUnavailable: 0  # No downtime
```

**Why Deployment**:
- Automatically manages ReplicaSets
- Enables rolling updates
- Supports rollbacks
- Version control of configurations

**In practice**: Almost always use Deployment, never create ReplicaSets directly.

#### Q3: What are Services and the different service types?

**Answer**:
Services provide stable network endpoint for pods.

```yaml
# ClusterIP: Internal only (default)
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  type: ClusterIP     # Only accessible within cluster
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app: api

---
# NodePort: Expose on node (for development)
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  type: NodePort      # Accessible from outside on specific port
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30001 # External access: node-ip:30001
  selector:
    app: api

---
# LoadBalancer: Cloud provider load balancer (production)
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  type: LoadBalancer  # Cloud creates external LB
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app: api

---
# ExternalName: Proxy to external service
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: db.example.com
```

**Selection**:
- **ClusterIP**: Internal microservices communication
- **NodePort**: Development, quick access
- **LoadBalancer**: Production external exposure
- **Ingress**: Better than LoadBalancer for multiple services

#### Q4: Explain Ingress and when to use it instead of LoadBalancer

**Answer**:
Ingress is Layer 7 (application layer) routing, while LoadBalancer is Layer 4 (transport).

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
spec:
  ingressClassName: nginx    # Use NGINX ingress controller
  rules:
  # Route domain -> service
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
  - host: admin.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 80
```

**Advantages over LoadBalancer**:
- **Cost**: Single LB for multiple services
- **Routing**: Domain/path based routing
- **SSL/TLS**: Certificate management
- **Rate limiting, auth**: Built-in features

#### Q5: Explain ConfigMap and Secret, and when to use each

**Answer**:
Both store configuration, but Secret is for sensitive data.

```yaml
# ConfigMap: Non-sensitive configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "Info"
  ASPNETCORE_ENVIRONMENT: "Production"
  DATABASE_NAME: "MyDb"

---
# Secret: Sensitive data (base64 encoded)
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:  # Automatically base64 encoded
  db-user: "sa"
  db-password: "SecurePassword123"
```

**Usage in Pod**:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapi:1.0
    env:
    # From ConfigMap
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: LOG_LEVEL
    # From Secret
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: db-password
    # Or mount as files
    volumeMounts:
    - name: config
      mountPath: /etc/config
      readOnly: true
  volumes:
  - name: config
    configMap:
      name: app-config
```

**When to use**:
- **ConfigMap**: Application settings, non-sensitive
- **Secret**: Passwords, API keys, certificates

#### Q6: Explain PersistentVolume and PersistentVolumeClaim

**Answer**:
PersistentVolume (PV) is storage resource; PersistentVolumeClaim (PVC) is request for storage.

```yaml
# PersistentVolume: Cluster resource
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-data
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: standard
  hostPath:              # Local storage
    path: /mnt/data

---
# PersistentVolumeClaim: Pod's request
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-data
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: standard
  resources:
    requests:
      storage: 5Gi

---
# Pod using PVC
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: pvc-data
```

**Storage Classes for dynamic provisioning**:
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/azure-disk
parameters:
  kind: Managed
  storageaccounttype: Premium_LRS  # SSD
  cachingmode: ReadOnly

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-fast
spec:
  storageClassName: fast-ssd
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
```

#### Q7: Explain Liveness Probe and Readiness Probe

**Answer**:
Both monitor pod health but trigger different actions.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapi:1.0
    
    # Liveness: Is application alive?
    # If fails: Container is RESTARTED
    livenessProbe:
      httpGet:
        path: /health/live
        port: 8080
      initialDelaySeconds: 30    # Wait 30s before first check
      periodSeconds: 10          # Check every 10s
      timeoutSeconds: 5
      failureThreshold: 3        # Fail after 3 failures
    
    # Readiness: Is application ready for traffic?
    # If fails: Pod REMOVED FROM SERVICE (no restart)
    readinessProbe:
      httpGet:
        path: /health/ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 2
    
    # Startup: Wait for app to start (prevents premature checks)
    startupProbe:
      httpGet:
        path: /health/startup
        port: 8080
      initialDelaySeconds: 0
      periodSeconds: 1
      failureThreshold: 30       # Fail after 30s total
```

**Difference**:
- **Liveness**: Detects stuck/crashed processes → Restart
- **Readiness**: Detects not-yet-ready processes → No restart, remove from load balancer

#### Q8: Explain Horizontal Pod Autoscaler (HPA)

**Answer**:
Automatically scales pods based on metrics.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
  # Scale based on CPU
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization        # Percentage
        averageUtilization: 70   # Scale up if avg > 70%
  # Scale based on memory
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  # Custom metrics
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"
  # Scale-down behavior
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300    # Wait 5min before scaling down
      policies:
      - type: Percent
        value: 50                        # Scale down by 50% max
        periodSeconds: 15
```

**How it works**:
1. Metrics Server collects CPU/memory from containers
2. HPA evaluates metrics against targets
3. If avg CPU > 70%, scale up (add replicas)
4. If avg CPU < 70%, eventually scale down

#### Q9: How would you perform a rolling update in Kubernetes?

**Answer**:
Deployment automatically performs rolling updates.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1           # One extra pod during update (4 total)
      maxUnavailable: 0    # Never go below 3 pods (no downtime)
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: myapi:1.0
```

**Update process**:
```bash
# Update image
kubectl set image deployment/api api=myapi:2.0

# Watch rollout
kubectl rollout status deployment/api
kubectl rollout history deployment/api

# Rollback if issues
kubectl rollout undo deployment/api
kubectl rollout undo deployment/api --to-revision=1
```

**Timeline**:
```
Initial:           3 pods v1.0
After update 1:    3 pods v1.0 + 1 pod v2.0 (4 total)
After v1 stop:     2 pods v1.0 + 2 pods v2.0
After v1 stop:     1 pod v1.0 + 3 pods v2.0
Final:             3 pods v2.0 (zero downtime!)
```

#### Q10: Explain Kubernetes namespaces and RBAC

**Answer**:
Namespaces provide logical isolation; RBAC controls permissions.

```yaml
# Create namespace
apiVersion: v1
kind: Namespace
metadata:
  name: production

---
# Resource in namespace
apiVersion: v1
kind: Pod
metadata:
  namespace: production
  name: app-pod
spec:
  containers:
  - name: app
    image: myapi:1.0
```

```bash
# List pods in namespace
kubectl get pods -n production

# Set default namespace
kubectl config set-context --current --namespace=production
```

**RBAC Example**:
```yaml
# ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: production

---
# Role
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-role
  namespace: production
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]

---
# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-role-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: app-sa
  namespace: production
roleRef:
  kind: Role
  name: app-role
  apiGroup: rbac.authorization.k8s.io

---
# Pod using ServiceAccount
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
  namespace: production
spec:
  serviceAccountName: app-sa
  containers:
  - name: app
    image: myapi:1.0
```

---

### Scenario-Based Interview Questions

#### Scenario 1: Microservices Deployment Challenge

**Question**: "Design a system with 3 microservices (.NET API, Node.js worker, Python ML service), a database, and cache. Use Docker Compose for development and explain how you'd scale to production with Kubernetes."

**Answer**:

**Development (docker-compose.yml)**:
```yaml
version: '3.9'

services:
  api:
    build:
      context: ./api
      dockerfile: Dockerfile
    ports:
      - "5000:8080"
    environment:
      - DB_HOST=database
      - CACHE_HOST=redis
    depends_on:
      database:
        condition: service_healthy

  worker:
    build:
      context: ./worker
      dockerfile: Dockerfile
    environment:
      - QUEUE_HOST=redis
      - DB_HOST=database
    depends_on:
      - redis
      - database

  ml-service:
    build:
      context: ./ml
      dockerfile: Dockerfile
    ports:
      - "5001:5000"
    environment:
      - CACHE_HOST=redis

  database:
    image: mysql:8
    environment:
      - MYSQL_ROOT_PASSWORD=root
    ports:
      - "3306:3306"
    volumes:
      - db-data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping"]

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  db-data:
```

**Production (Kubernetes)**:
```bash
# 1. Build and push images
docker build -t myregistry.azurecr.io/api:1.0 ./api
docker push myregistry.azurecr.io/api:1.0

# 2. Create AKS cluster
az aks create --name prod-cluster --attach-acr myregistry

# 3. Deploy database (StatefulSet)
# 4. Deploy API (Deployment)
# 5. Configure services and ingress
# 6. Enable autoscaling (HPA)
# 7. Monitor with Prometheus/Grafana
```

**Key differences**:
- Docker Compose: Single host, good for development
- Kubernetes: Multi-host, auto-scaling, self-healing, production-ready

---

## Summary

**Docker** provides containerization for consistent, portable deployments. **Docker Compose** simplifies multi-container orchestration on a single machine. **Kubernetes** scales containers across clusters automatically.

**Cost Reduction**: Docker reduces infrastructure costs by 50-70% through better resource utilization. Kubernetes provides additional savings through automation and efficient scaling.

**Azure Integration**: Use ACR for private image storage and AKS for production Kubernetes deployments with managed services and enterprise support.