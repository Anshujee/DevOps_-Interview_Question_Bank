# Production Grade Kubernetes — Complete Real World Guide

> This document explains exactly how Kubernetes is used in a real production environment.
> Every concept is explained using the **AzureShop** project as a real-world example.
> Read this end to end to understand the full picture of production Kubernetes.

---

## Table of Contents

1. [The Big Picture — What Does Production Kubernetes Look Like?](#1-the-big-picture)
2. [Cluster Setup — AKS in Azure](#2-cluster-setup)
3. [Namespace Strategy](#3-namespace-strategy)
4. [Node Pools — Organising Your Compute](#4-node-pools)
5. [Deploying Applications — Deployments and ReplicaSets](#5-deploying-applications)
6. [Resource Requests and Limits](#6-resource-requests-and-limits)
7. [Networking — Services and Ingress](#7-networking)
8. [Configuration Management — ConfigMaps and Secrets](#8-configuration-management)
9. [Health Checks — Liveness, Readiness, Startup Probes](#9-health-checks)
10. [Auto Scaling — HPA and Cluster Autoscaler](#10-auto-scaling)
11. [Storage — Persistent Volumes](#11-storage)
12. [Security — RBAC, Network Policies, Pod Security](#12-security)
13. [CI/CD Pipeline Integration](#13-cicd-pipeline-integration)
14. [Helm — Package Manager for Kubernetes](#14-helm)
15. [Monitoring and Logging](#15-monitoring-and-logging)
16. [High Availability and Disaster Recovery](#16-high-availability-and-disaster-recovery)
17. [Day 2 Operations — Upgrades, Rollbacks, Maintenance](#17-day-2-operations)
18. [The Complete AzureShop Production Flow](#18-complete-azureshop-production-flow)

---

## 1. The Big Picture

Before jumping into details, let me give you a complete mental picture of what production Kubernetes looks like in a real company.

**What we had in AzureShop:**

```
Internet
    |
    ↓
Azure Application Gateway (WAF + Load Balancer)
    |
    ↓
AKS Cluster (3 node pools, 9 nodes total)
    |
    ├── dev namespace
    │     ├── user-service (1 pod)
    │     ├── order-service (1 pod)
    │     ├── payment-service (1 pod)
    │     └── product-service (1 pod)
    │
    ├── staging namespace
    │     ├── user-service (2 pods)
    │     ├── order-service (2 pods)
    │     ├── payment-service (2 pods)
    │     └── product-service (2 pods)
    │
    ├── production namespace
    │     ├── user-service (3 pods — spread across 3 nodes)
    │     ├── order-service (3 pods)
    │     ├── payment-service (3 pods)
    │     └── product-service (3 pods)
    │
    └── monitoring namespace
          ├── prometheus (1 pod)
          ├── grafana (1 pod)
          └── alertmanager (1 pod)
```

Every component in this picture has a reason for existing. Let me explain each one from the ground up.

---

## 2. Cluster Setup

### 2.1 Creating the AKS Cluster

In AzureShop, the entire AKS cluster was created using Terraform — not the Azure Portal. This is the production-grade approach because it is repeatable, version-controlled, and auditable.

```hcl
resource "azurerm_kubernetes_cluster" "main" {
  name                = "azureshop-aks-prod"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  dns_prefix          = "azureshop"
  kubernetes_version  = "1.28.3"

  # System node pool — runs Kubernetes system components
  default_node_pool {
    name                = "system"
    node_count          = 3
    vm_size             = "Standard_D2s_v3"
    os_disk_size_gb     = 128
    vnet_subnet_id      = azurerm_subnet.aks.id
    zones               = ["1", "2", "3"]    # Spread across 3 availability zones
    only_critical_addons_enabled = true      # Only system pods here
  }

  # Managed identity — AKS authenticates to Azure using this
  identity {
    type = "SystemAssigned"
  }

  # Networking — Azure CNI gives pods real VNet IPs
  network_profile {
    network_plugin    = "azure"
    network_policy    = "calico"    # Enables NetworkPolicy support
    load_balancer_sku = "standard"
    outbound_type     = "loadBalancer"
  }

  # RBAC with Azure AD integration
  azure_active_directory_role_based_access_control {
    managed                = true
    admin_group_object_ids = [var.aks_admin_group_id]
  }

  # Monitoring integration
  oms_agent {
    log_analytics_workspace_id = azurerm_log_analytics_workspace.main.id
  }

  tags = {
    environment = "production"
    project     = "azureshop"
    managed-by  = "terraform"
  }
}
```

**Why each setting matters:**

- `kubernetes_version` — always pin the version. Never let it auto-upgrade in production.
- `zones = ["1", "2", "3"]` — spreading nodes across 3 availability zones means if one Azure data centre has an issue, your cluster keeps running.
- `network_policy = "calico"` — enables NetworkPolicy so you can control pod-to-pod traffic.
- `azure_active_directory_role_based_access_control` — only people in the approved Azure AD group can access the cluster.
- `oms_agent` — sends all container logs and metrics to Azure Log Analytics automatically.

---

### 2.2 Networking — Azure CNI

In production, we always use **Azure CNI** (not kubenet). With Azure CNI, every pod gets a real IP address from the VNet subnet — just like a VM would. This means:

- Pods can be reached directly from other Azure services (databases, App Gateway, etc.)
- No NAT overhead
- No port conflicts
- Full network visibility and security group compatibility

```
VNet: 10.0.0.0/8
  └── AKS Subnet: 10.1.0.0/16
        ├── Node 1: 10.1.0.4
        │     ├── user-service pod: 10.1.0.10
        │     ├── order-service pod: 10.1.0.11
        │     └── product-service pod: 10.1.0.12
        ├── Node 2: 10.1.0.5
        │     └── payment-service pod: 10.1.0.13
        └── Node 3: 10.1.0.6
              └── prometheus pod: 10.1.0.14
```

Every pod has a real routable IP in the VNet.

---

## 3. Namespace Strategy

Namespaces in production Kubernetes are not just organisational labels — they are security and resource boundaries.

**AzureShop namespace structure:**

```bash
kubectl get namespaces

NAME              STATUS
default           Active    # Never use this for applications
kube-system       Active    # Kubernetes system components
kube-public       Active    # Public cluster information
dev               Active    # Development environment
staging           Active    # Staging/QA environment
production        Active    # Production environment
monitoring        Active    # Prometheus, Grafana, Alertmanager
ingress-nginx     Active    # Nginx ingress controller
cert-manager      Active    # TLS certificate management
```

**Why separate namespaces:**

1. **Resource isolation** — each namespace has its own ResourceQuota. Dev cannot consume all cluster resources and starve production.
2. **Security isolation** — NetworkPolicies are scoped to namespaces. Production pods cannot be accessed from dev.
3. **Access control** — developers have full access to dev, read-only to staging, no direct access to production. All prod changes go through the pipeline.
4. **Clean organisation** — `kubectl get pods -n production` shows only production pods.

**Resource Quota for each namespace:**

```yaml
# production-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "20"          # Total CPU requests in namespace
    requests.memory: "40Gi"     # Total memory requests
    limits.cpu: "40"            # Total CPU limits
    limits.memory: "80Gi"       # Total memory limits
    pods: "50"                  # Maximum pods
    services: "20"              # Maximum services
    persistentvolumeclaims: "10"
```

```yaml
# dev-quota.yaml — dev gets less resources
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: dev
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "8Gi"
    pods: "20"
```

---

## 4. Node Pools

In production AKS, you never use just one node pool. Different workloads have different requirements — you need different node sizes and configurations for each.

**AzureShop node pool design:**

```hcl
# System Node Pool — runs Kubernetes system components only
resource "azurerm_kubernetes_cluster_node_pool" "system" {
  name                  = "system"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.main.id
  vm_size               = "Standard_D2s_v3"    # 2 CPU, 8GB RAM
  node_count            = 3
  zones                 = ["1", "2", "3"]
  node_taints           = ["CriticalAddonsOnly=true:NoSchedule"]
  # Taint ensures only system pods land here
}

# Application Node Pool — runs all microservices
resource "azurerm_kubernetes_cluster_node_pool" "application" {
  name                  = "apppool"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.main.id
  vm_size               = "Standard_D4s_v3"    # 4 CPU, 16GB RAM
  node_count            = 3
  min_count             = 3
  max_count             = 10
  enable_auto_scaling   = true    # Scales based on demand
  zones                 = ["1", "2", "3"]
}

# Monitoring Node Pool — runs Prometheus, Grafana
resource "azurerm_kubernetes_cluster_node_pool" "monitoring" {
  name                  = "monitorpool"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.main.id
  vm_size               = "Standard_D2s_v3"
  node_count            = 2
  node_taints           = ["dedicated=monitoring:NoSchedule"]
  node_labels           = { "dedicated" = "monitoring" }
}
```

**Why this design:**
- System pool is tainted — app pods cannot land there, keeping system components healthy
- App pool has autoscaling — handles traffic spikes automatically
- Monitoring pool is dedicated — monitoring always has resources even during traffic spikes

---

## 5. Deploying Applications

### 5.1 Deployment

In production, you never run bare pods. You always use a **Deployment**. A Deployment manages a **ReplicaSet** which manages multiple identical **Pods**.

```
Deployment (desired state definition)
    └── ReplicaSet (maintains pod count)
          ├── Pod 1 (user-service instance)
          ├── Pod 2 (user-service instance)
          └── Pod 3 (user-service instance)
```

**Complete production-grade Deployment for AzureShop user-service:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  namespace: production
  labels:
    app: user-service
    version: v2.1.0
    team: backend
spec:
  replicas: 3    # Always 3 pods in production for HA

  selector:
    matchLabels:
      app: user-service

  # Rolling update strategy — zero downtime deployments
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1         # Create 1 extra pod during update
      maxUnavailable: 0   # Never take pods down before new ones are ready

  template:
    metadata:
      labels:
        app: user-service
        version: v2.1.0

    spec:
      # Only pull from ACR using managed identity
      serviceAccountName: user-service-sa

      # Spread pods across different nodes — HA guarantee
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - user-service
              topologyKey: kubernetes.io/hostname
              # No two user-service pods on the same node

      containers:
        - name: user-service
          image: azureshopacr.azurecr.io/user-service:v2.1.0
          imagePullPolicy: IfNotPresent

          ports:
            - containerPort: 3000
              name: http

          # Environment variables from ConfigMap and Secrets
          env:
            - name: NODE_ENV
              value: "production"
            - name: PORT
              value: "3000"
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: user-service-secrets
                  key: database-url
            - name: REDIS_URL
              valueFrom:
                configMapKeyRef:
                  name: user-service-config
                  key: redis-url

          # Resource requests and limits — MANDATORY in production
          resources:
            requests:
              cpu: "250m"       # 0.25 CPU cores guaranteed
              memory: "256Mi"   # 256MB RAM guaranteed
            limits:
              cpu: "500m"       # Maximum 0.5 CPU cores
              memory: "512Mi"   # Maximum 512MB RAM

          # Health checks
          startupProbe:
            httpGet:
              path: /health
              port: 3000
            failureThreshold: 30
            periodSeconds: 10
            # App has 300 seconds to start (30 * 10)

          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 0
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3

          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 0
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 3

      # Graceful shutdown — give app time to finish in-flight requests
      terminationGracePeriodSeconds: 30
```

### 5.2 Pod Anti-Affinity — Why It Matters in Production

`podAntiAffinity` with `requiredDuringSchedulingIgnoredDuringExecution` ensures no two pods of the same service land on the same node.

**Why this matters:**

Without anti-affinity:
```
Node 1: user-service pod1, user-service pod2, user-service pod3
Node 2: (empty)
Node 3: (empty)
```
If Node 1 goes down — all 3 user-service pods die simultaneously — complete service outage.

With anti-affinity:
```
Node 1: user-service pod1
Node 2: user-service pod2
Node 3: user-service pod3
```
If Node 1 goes down — 2 of 3 pods still running — service continues with reduced capacity.

---

## 6. Resource Requests and Limits

This is one of the most important production Kubernetes concepts that is often skipped in tutorials but critical in real environments.

**Requests vs Limits:**

| | Request | Limit |
|---|---|---|
| **What it is** | The minimum resources the pod **needs** | The maximum resources the pod **can use** |
| **Used for** | Scheduling decisions — Kubernetes finds a node with enough available resources | Enforcement — container is killed (OOMKilled) or throttled if it exceeds this |
| **Effect if too low** | Pod may be OOMKilled if actual usage exceeds limit | N/A |
| **Effect if too high** | Pod cannot be scheduled — no node has enough resources | N/A |

**In AzureShop — our production resource strategy:**

```yaml
resources:
  requests:
    cpu: "250m"       # Ask for 0.25 cores — Kubernetes guarantees this
    memory: "256Mi"   # Ask for 256MB — Kubernetes guarantees this
  limits:
    cpu: "500m"       # Allow burst up to 0.5 cores
    memory: "512Mi"   # Hard cap at 512MB — OOMKilled if exceeded
```

**CPU is measured in millicores:**
- `1000m` = 1 full CPU core
- `500m` = 0.5 CPU core
- `250m` = 0.25 CPU core

**What happens without resource limits:**
- One runaway pod can consume all node CPU — starving other pods
- One memory leak crashes the node — taking down all pods on it
- Kubernetes scheduler cannot make good decisions about pod placement

---

## 7. Networking

### 7.1 Services — Internal Connectivity

A **Service** gives a stable IP and DNS name to a group of pods. Pods die and restart with new IPs — but the Service IP never changes.

**Three Service types we use in production:**

**ClusterIP — Internal communication between services:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: user-service
  namespace: production
spec:
  type: ClusterIP    # Only accessible inside the cluster
  selector:
    app: user-service
  ports:
    - port: 80          # Service port
      targetPort: 3000  # Container port
      protocol: TCP
```

Inside the cluster, the `order-service` reaches `user-service` using:
```
http://user-service.production.svc.cluster.local:80
# or simply within the same namespace:
http://user-service:80
```

Kubernetes DNS automatically resolves `user-service` to the ClusterIP — regardless of which pods are behind it or their individual IPs.

---

**LoadBalancer — External access for a single service:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-gateway
  namespace: production
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"  # Internal LB only
spec:
  type: LoadBalancer
  selector:
    app: api-gateway
  ports:
    - port: 443
      targetPort: 8080
```

Creates an Azure Load Balancer with a public (or internal) IP. Used when a single service needs direct external access.

---

### 7.2 Ingress — The Production Approach

In production, you do not create a LoadBalancer Service for every microservice — that would create dozens of Azure Load Balancers and cost a fortune. Instead, you use one **Ingress Controller** that receives all external traffic and routes it to the correct service based on the URL path or hostname.

```
Internet
    |
    ↓
Azure Application Gateway / Nginx Ingress Controller
(Single external IP — one Load Balancer)
    |
    ├── /api/users      → user-service:80
    ├── /api/orders     → order-service:80
    ├── /api/payments   → payment-service:80
    └── /api/products   → product-service:80
```

**AzureShop Ingress resource:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: azureshop-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"  # Auto TLS certificates
spec:
  tls:
    - hosts:
        - api.azureshop.com
      secretName: azureshop-tls    # TLS certificate stored as a Secret

  rules:
    - host: api.azureshop.com
      http:
        paths:
          - path: /api/users
            pathType: Prefix
            backend:
              service:
                name: user-service
                port:
                  number: 80

          - path: /api/orders
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 80

          - path: /api/payments
            pathType: Prefix
            backend:
              service:
                name: payment-service
                port:
                  number: 80
```

One Ingress, one external IP, one Load Balancer — routes to all four microservices based on path.

---

## 8. Configuration Management

### 8.1 ConfigMap — Non-sensitive configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: user-service-config
  namespace: production
data:
  redis-url: "redis://redis-service:6379"
  log-level: "info"
  max-connections: "100"
  feature-flags: |
    NEW_UI=true
    BETA_CHECKOUT=false
```

Use ConfigMaps for:
- Feature flags
- Service URLs (non-sensitive)
- Log levels
- Application settings that change between environments

### 8.2 Secrets — Sensitive configuration

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: user-service-secrets
  namespace: production
type: Opaque
stringData:
  database-url: "postgres://admin:password@db-service:5432/users"
  jwt-secret: "supersecretjwtkey"
  stripe-api-key: "sk_live_xxx"
```

**Important:** In production, we never store Secrets in Git. Instead, we use **Azure Key Vault** with the **Secrets Store CSI Driver** to mount secrets directly from Key Vault into pods:

```yaml
# Mount Azure Key Vault secrets directly into the pod
volumes:
  - name: secrets-store
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: "azureshop-keyvault"

volumeMounts:
  - name: secrets-store
    mountPath: "/mnt/secrets"
    readOnly: true
```

The pod reads secrets from `/mnt/secrets/database-url` — secrets never exist in the cluster as Kubernetes Secret objects, they come directly from Azure Key Vault.

---

## 9. Health Checks

Production Kubernetes uses three types of probes for every container. Each serves a different purpose.

```
Startup Probe   → Is the app done starting up?
Liveness Probe  → Is the app still alive and healthy?
Readiness Probe → Is the app ready to receive traffic?
```

**Complete probe configuration for AzureShop:**

```yaml
# Startup probe — runs FIRST, disables liveness until app is up
startupProbe:
  httpGet:
    path: /health
    port: 3000
  failureThreshold: 30     # 30 attempts
  periodSeconds: 10        # Every 10 seconds
  # Total: 300 seconds (5 min) for app to start — then hands off to liveness

# Liveness probe — runs AFTER startup succeeds
# Restarts the container if this fails
livenessProbe:
  httpGet:
    path: /health          # Returns 200 if app process is alive
    port: 3000
  periodSeconds: 10        # Check every 10 seconds
  timeoutSeconds: 5        # Timeout after 5 seconds
  failureThreshold: 3      # Restart after 3 consecutive failures

# Readiness probe — controls whether pod receives traffic
# Pod is removed from Service endpoints if this fails
readinessProbe:
  httpGet:
    path: /ready           # Returns 200 only if app is ready to serve requests
                           # Can return 503 if DB not connected yet
  port: 3000
  periodSeconds: 5         # Check every 5 seconds
  timeoutSeconds: 3
  failureThreshold: 3
```

**The difference between /health and /ready endpoints:**

```javascript
// /health — simple alive check
app.get('/health', (req, res) => {
  res.status(200).json({ status: 'alive' })
})

// /ready — deep check — only 200 if truly ready for traffic
app.get('/ready', async (req, res) => {
  try {
    await db.query('SELECT 1')      // Check DB connection
    await redis.ping()              // Check Redis connection
    res.status(200).json({ status: 'ready' })
  } catch (err) {
    res.status(503).json({ status: 'not ready', error: err.message })
  }
})
```

**Why this matters in production:**

During a rolling update, new pods start up and go through this sequence:
1. Startup probe runs — pod stays out of Service until startup completes
2. Once startup passes, liveness probe begins
3. Readiness probe begins — pod only joins Service when `/ready` returns 200
4. Old pod is only terminated AFTER new pod is confirmed ready

This guarantees zero-downtime deployments.

---

## 10. Auto Scaling

### 10.1 Horizontal Pod Autoscaler (HPA)

HPA automatically scales the number of pods based on CPU or memory usage.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: user-service-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: user-service

  minReplicas: 3     # Never go below 3 pods
  maxReplicas: 10    # Never go above 10 pods

  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70    # Scale up when CPU > 70%

    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80    # Scale up when memory > 80%
```

**How it works in AzureShop:**

```
Normal traffic:     3 pods  (CPU 40%)
Traffic spike:      CPU rises to 75%
HPA detects:        75% > 70% threshold
HPA scales to:      5 pods
CPU drops to:       45%

Traffic drops:      CPU falls to 20%
HPA waits:          5 minutes (cool-down period)
HPA scales down:    3 pods
```

### 10.2 Cluster Autoscaler

HPA adds more pods — but what if there are no nodes with enough resources to schedule them?

Cluster Autoscaler adds new nodes to the cluster automatically when pods are pending due to insufficient resources:

```hcl
# In Terraform — enable autoscaling on node pool
resource "azurerm_kubernetes_cluster_node_pool" "application" {
  enable_auto_scaling = true
  min_count           = 3
  max_count           = 10
}
```

**Scale-up flow:**
```
Traffic spike
    ↓
HPA adds pods (e.g. 3 → 7 pods)
    ↓
4 new pods are PENDING (no space on existing 3 nodes)
    ↓
Cluster Autoscaler detects pending pods
    ↓
Adds 2 new nodes to the app node pool
    ↓
Pending pods get scheduled on new nodes
    ↓
All 7 pods running
```

**Scale-down flow:**
```
Traffic drops
    ↓
HPA reduces pods (7 → 3)
    ↓
Some nodes are now empty or underutilised
    ↓
Cluster Autoscaler waits 10 minutes (cooldown)
    ↓
Drains and removes underutilised nodes
    ↓
Back to 3 nodes
```

---

## 11. Storage

Not all applications are stateless. Databases, message queues, and file stores need persistent storage that survives pod restarts.

### 11.1 Storage Classes in AKS

```bash
kubectl get storageclasses

NAME                    PROVISIONER
default                 disk.csi.azure.com    # Azure Premium SSD
managed-premium         disk.csi.azure.com    # Azure Premium SSD
azurefile               file.csi.azure.com    # Azure Files (shared)
azurefile-premium       file.csi.azure.com    # Azure Files Premium
```

### 11.2 PersistentVolumeClaim — Requesting Storage

```yaml
# Request 10GB of Premium SSD storage
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce    # Only one pod can mount this at a time
  storageClassName: managed-premium
  resources:
    requests:
      storage: 10Gi
```

### 11.3 Using PVC in a StatefulSet

For stateful workloads like databases, we use **StatefulSet** instead of Deployment. StatefulSets give each pod:
- A stable, predictable name (`postgres-0`, `postgres-1`, `postgres-2`)
- Its own dedicated PVC that persists across pod restarts

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
spec:
  serviceName: postgres
  replicas: 1

  template:
    spec:
      containers:
        - name: postgres
          image: postgres:15-alpine
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data

  # Each pod gets its own PVC automatically
  volumeClaimTemplates:
    - metadata:
        name: postgres-data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: managed-premium
        resources:
          requests:
            storage: 50Gi
```

**In AzureShop:** We ran PostgreSQL as a managed Azure Database (not inside Kubernetes) for production workloads — managed databases handle backups, HA, and maintenance automatically. We only ran databases inside Kubernetes for dev and staging environments.

---

## 12. Security

### 12.1 RBAC — Who Can Do What

```yaml
# Role — defines permissions within a namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer-role
  namespace: dev
rules:
  - apiGroups: ["apps"]
    resources: ["deployments", "pods"]
    verbs: ["get", "list", "watch"]    # Read only
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get", "list"]             # Can read logs

---
# RoleBinding — assigns the role to a user/group
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: dev
subjects:
  - kind: Group
    name: "azureshop-developers"    # Azure AD group
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io
```

**AzureShop RBAC structure:**

| Role | Who | What they can do |
|---|---|---|
| `cluster-admin` | DevOps team | Everything |
| `developer-role` | Dev team | Read pods/logs in dev namespace only |
| `staging-viewer` | QA team | Read-only access to staging |
| `ci-deployer` | Azure DevOps pipeline | Deploy to production namespace only |

### 12.2 Network Policies — Microsegmentation

By default, all pods in Kubernetes can talk to all other pods. In production, this is a security risk. Network Policies lock down pod-to-pod communication.

```yaml
# Allow user-service to only talk to: order-service and the database
# Block all other traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: user-service-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: user-service

  policyTypes:
    - Ingress
    - Egress

  ingress:
    # Only allow traffic from the ingress controller
    - from:
        - namespaceSelector:
            matchLabels:
              name: ingress-nginx
      ports:
        - port: 3000

  egress:
    # Only allow outbound to order-service
    - to:
        - podSelector:
            matchLabels:
              app: order-service
      ports:
        - port: 80

    # Allow DNS resolution
    - to: []
      ports:
        - port: 53
          protocol: UDP

    # Allow outbound to database
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - port: 5432
```

---

## 13. CI/CD Pipeline Integration

This is where everything comes together. Here is the complete Azure DevOps pipeline for deploying a microservice to AKS in AzureShop:

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
      - main
  paths:
    include:
      - services/user-service/**

variables:
  IMAGE_NAME: azureshopacr.azurecr.io/user-service
  IMAGE_TAG: $(Build.BuildId)
  K8S_NAMESPACE: production

stages:

  # Stage 1 — Build and Test
  - stage: Build
    jobs:
      - job: BuildAndTest
        pool:
          vmImage: ubuntu-latest
        steps:
          - script: npm ci && npm test
            displayName: "Install and Test"

          - task: Docker@2
            displayName: "Build Docker Image"
            inputs:
              command: build
              dockerfile: services/user-service/Dockerfile
              tags: $(IMAGE_TAG)

          - script: |
              trivy image --exit-code 1 --severity HIGH,CRITICAL \
                $(IMAGE_NAME):$(IMAGE_TAG)
            displayName: "Security Scan — Trivy"

          - task: Docker@2
            displayName: "Push to ACR"
            inputs:
              command: push
              containerRegistry: azureshop-acr-connection
              tags: $(IMAGE_TAG)

  # Stage 2 — Deploy to Dev
  - stage: DeployDev
    dependsOn: Build
    jobs:
      - deployment: DeployDev
        environment: dev
        strategy:
          runOnce:
            deploy:
              steps:
                - task: KubernetesManifest@0
                  displayName: "Deploy to Dev"
                  inputs:
                    action: deploy
                    kubernetesServiceConnection: aks-dev-connection
                    namespace: dev
                    manifests: k8s/user-service/deployment.yaml
                    containers: $(IMAGE_NAME):$(IMAGE_TAG)

                - script: |
                    kubectl rollout status deployment/user-service -n dev --timeout=5m
                  displayName: "Verify Rollout"

  # Stage 3 — Deploy to Staging (after dev succeeds)
  - stage: DeployStaging
    dependsOn: DeployDev
    jobs:
      - deployment: DeployStaging
        environment: staging    # Requires manual approval
        strategy:
          runOnce:
            deploy:
              steps:
                - task: KubernetesManifest@0
                  inputs:
                    action: deploy
                    namespace: staging
                    manifests: k8s/user-service/deployment.yaml
                    containers: $(IMAGE_NAME):$(IMAGE_TAG)

  # Stage 4 — Deploy to Production (requires approval)
  - stage: DeployProduction
    dependsOn: DeployStaging
    jobs:
      - deployment: DeployProd
        environment: production    # Team lead must approve in Azure DevOps
        strategy:
          runOnce:
            deploy:
              steps:
                - task: KubernetesManifest@0
                  inputs:
                    action: deploy
                    namespace: production
                    manifests: k8s/user-service/deployment.yaml
                    containers: $(IMAGE_NAME):$(IMAGE_TAG)

                - script: |
                    kubectl rollout status deployment/user-service \
                      -n production --timeout=10m
                  displayName: "Verify Production Rollout"
```

**The deployment flow in detail:**

```
Developer pushes code to main branch
              ↓
Pipeline triggers automatically
              ↓
Stage 1 — Build
  → npm test (unit tests)
  → docker build
  → trivy scan (security)
  → docker push to ACR
              ↓
Stage 2 — Deploy to Dev (automatic)
  → kubectl apply deployment with new image tag
  → Rolling update begins:
      New pod starts (v2.1.0)
      Startup probe runs
      Readiness probe passes
      New pod joins Service
      Old pod removed from Service
      Old pod terminates gracefully
  → kubectl rollout status (waits for completion)
              ↓
Stage 3 — Deploy to Staging (automatic after dev)
  → Same rolling update process
  → QA team tests
              ↓
Stage 4 — Deploy to Production (MANUAL APPROVAL required)
  → Team lead reviews and approves in Azure DevOps
  → Same rolling update — zero downtime
  → Verify rollout completes successfully
```

---

## 14. Helm

**Helm** is the package manager for Kubernetes. Instead of maintaining separate YAML files for dev, staging, and prod — you write one **Helm Chart** with templates and pass different values per environment.

**AzureShop Helm chart structure:**

```
charts/
└── user-service/
    ├── Chart.yaml          # Chart metadata
    ├── values.yaml         # Default values
    ├── values-dev.yaml     # Dev overrides
    ├── values-staging.yaml # Staging overrides
    ├── values-prod.yaml    # Production overrides
    └── templates/
        ├── deployment.yaml
        ├── service.yaml
        ├── ingress.yaml
        ├── hpa.yaml
        ├── configmap.yaml
        └── secret.yaml
```

**values.yaml — default values:**

```yaml
replicaCount: 1
image:
  repository: azureshopacr.azurecr.io/user-service
  tag: latest
  pullPolicy: IfNotPresent

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 200m
    memory: 256Mi

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 5
```

**values-prod.yaml — production overrides:**

```yaml
replicaCount: 3

resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
```

**Deploying with Helm:**

```bash
# Deploy to dev
helm upgrade --install user-service ./charts/user-service \
  -f charts/user-service/values-dev.yaml \
  -n dev \
  --set image.tag=$(BUILD_ID)

# Deploy to production
helm upgrade --install user-service ./charts/user-service \
  -f charts/user-service/values-prod.yaml \
  -n production \
  --set image.tag=$(BUILD_ID)
```

**Rollback with Helm:**

```bash
# See revision history
helm history user-service -n production

REVISION  STATUS     CHART             DESCRIPTION
1         deployed   user-service-1.0  Initial release
2         deployed   user-service-1.1  Image updated to v2.1.0
3         failed     user-service-1.2  Upgrade failed

# Rollback to revision 2 (last known good)
helm rollback user-service 2 -n production
```

---

## 15. Monitoring and Logging

### 15.1 Monitoring Stack

In AzureShop, every AKS cluster had:

```
Prometheus    → Scrapes metrics from all pods every 15 seconds
Grafana       → Visualises metrics in dashboards
Alertmanager  → Sends alerts when thresholds are breached
```

**Pods expose metrics via annotations:**

```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"     # Prometheus scrapes this pod
    prometheus.io/port: "3000"
    prometheus.io/path: "/metrics"   # Metrics endpoint
```

**Alerting rules:**

```yaml
# Alert when pod restarts more than 3 times in 15 minutes
- alert: PodCrashLooping
  expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Pod {{ $labels.pod }} is crash looping"

# Alert when memory usage > 85%
- alert: HighMemoryUsage
  expr: container_memory_usage_bytes / container_spec_memory_limit_bytes > 0.85
  for: 5m
  labels:
    severity: warning
```

### 15.2 Logging

All container logs flow to Azure Log Analytics via the OMS Agent installed on every node:

```
Pod logs (stdout/stderr)
    ↓
OMS Agent (DaemonSet on every node)
    ↓
Azure Log Analytics Workspace
    ↓
Azure Monitor / Container Insights dashboards
```

**Querying logs in Azure Monitor:**

```kusto
ContainerLog
| where LogEntry contains "ERROR"
| where ContainerName == "user-service"
| where TimeGenerated > ago(1h)
| project TimeGenerated, LogEntry, PodName
| order by TimeGenerated desc
```

---

## 16. High Availability and Disaster Recovery

### 16.1 Pod Disruption Budget

PDB ensures a minimum number of pods are always running — even during node maintenance or cluster upgrades:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: user-service-pdb
  namespace: production
spec:
  minAvailable: 2    # At least 2 pods must be running at all times
  selector:
    matchLabels:
      app: user-service
```

With 3 replicas and `minAvailable: 2`, Kubernetes will only evict 1 pod at a time during maintenance.

### 16.2 Multi-AZ Deployment

All node pools in AzureShop used `zones: ["1", "2", "3"]`. Combined with pod anti-affinity, this means:

```
Availability Zone 1  → user-service pod1, order-service pod1
Availability Zone 2  → user-service pod2, order-service pod2
Availability Zone 3  → user-service pod3, order-service pod3
```

If Azure Zone 1 goes down completely — 2 out of 3 pods are still running in zones 2 and 3. The service degrades but does not go down.

---

## 17. Day 2 Operations

### 17.1 Rolling Updates — Zero Downtime Deployment

```bash
# Update image to a new version
kubectl set image deployment/user-service \
  user-service=azureshopacr.azurecr.io/user-service:v2.2.0 \
  -n production

# Watch the rolling update
kubectl rollout status deployment/user-service -n production

# Output:
Waiting for deployment "user-service" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "user-service" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "user-service" rollout to finish: 1 old replicas are pending termination...
deployment "user-service" successfully rolled out
```

### 17.2 Rollback

```bash
# Immediate rollback to previous version
kubectl rollout undo deployment/user-service -n production

# Rollback to a specific revision
kubectl rollout history deployment/user-service -n production
kubectl rollout undo deployment/user-service --to-revision=3 -n production
```

### 17.3 Node Maintenance

```bash
# Drain a node before maintenance
kubectl cordon <node-name>    # Stop new pods being scheduled
kubectl drain <node-name> \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=60           # Give pods 60 seconds to finish

# Perform maintenance...

# Return node to service
kubectl uncordon <node-name>
```

---

## 18. The Complete AzureShop Production Flow

Here is the complete end-to-end journey of a single code change — from developer laptop to production:

```
1. Developer writes code and pushes to feature branch
              ↓
2. Pull Request raised — peer code review
              ↓
3. PR merged to main branch
              ↓
4. Azure DevOps pipeline triggers automatically
              ↓
5. npm test — unit tests run
              ↓
6. docker build — multi-stage build
   Final image: ~85 MB (node:18-alpine, production deps only)
              ↓
7. trivy scan — security vulnerabilities checked
   HIGH/CRITICAL CVEs → pipeline fails, no deployment
              ↓
8. docker push → azureshopacr.azurecr.io/user-service:$(BUILD_ID)
              ↓
9. helm upgrade in DEV namespace
   Rolling update: new pod starts → probes pass → old pod removed
              ↓
10. kubectl rollout status — confirms successful rollout
              ↓
11. Automated smoke tests against dev endpoint
              ↓
12. helm upgrade in STAGING namespace (automatic)
              ↓
13. QA team runs regression tests in staging
              ↓
14. Team lead approves production deployment in Azure DevOps
              ↓
15. helm upgrade in PRODUCTION namespace
    Rolling update with PDB enforcement:
    - maxUnavailable: 0 (no pods go down before new ones are ready)
    - maxSurge: 1 (one extra pod created during update)
    - readinessProbe must pass before old pod is removed
              ↓
16. kubectl rollout status confirms all 3 pods updated
              ↓
17. Grafana dashboards monitored for 30 minutes
    Alertmanager watches error rate, latency, pod restarts
              ↓
18. If anything goes wrong:
    helm rollback user-service 2 -n production
    Previous version restored in ~60 seconds
              ↓
19. Change is live in production ✅
    Full audit trail: Git commit → PR → Pipeline run → Helm release
```

---

**This is what production-grade Kubernetes looks like in a real company.**

Every piece has a purpose. Nothing is optional. The combination of all these components gives you:
- **Zero-downtime deployments** — rolling updates with readiness probes
- **High availability** — multi-AZ, pod anti-affinity, PDB
- **Auto scaling** — HPA + Cluster Autoscaler handles traffic spikes
- **Security** — RBAC, NetworkPolicies, Key Vault, non-root containers
- **Observability** — Prometheus, Grafana, Azure Monitor
- **Fast recovery** — Helm rollback in 60 seconds
- **Full audit trail** — every change tracked in Git and pipeline logs
