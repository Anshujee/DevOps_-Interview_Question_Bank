# DevOps Lead Interview — Complete Question & Answer Guide
> Role: DevOps Lead | Experience: 12 Years | Location: Pune, MH
> All answers are explained in simple English with real examples from the **AzureShop** project.

---

## Table of Contents

1. [CI/CD Fundamentals](#1-cicd-fundamentals)
2. [Jenkins & Jenkins Pipelines](#2-jenkins--jenkins-pipelines)
3. [Docker & Containerization](#3-docker--containerization)
4. [Kubernetes & AKS](#4-kubernetes--aks)
5. [Helm Charts](#5-helm-charts)
6. [Infrastructure as Code — Terraform](#6-infrastructure-as-code--terraform)
7. [Scripting — Python, Groovy, Shell](#7-scripting--python-groovy-shell)
8. [Git & GitHub Enterprise](#8-git--github-enterprise)
9. [Monitoring & Observability](#9-monitoring--observability)
10. [Security & DevSecOps](#10-security--devsecops)
11. [Microservices Architecture](#11-microservices-architecture)
12. [Cloud — Azure](#12-cloud--azure)
13. [Maven & Build Tools](#13-maven--build-tools)
14. [Node.js & npm](#14-nodejs--npm)
15. [Site Reliability Engineering (SRE)](#15-site-reliability-engineering-sre)
16. [Incident Management & Troubleshooting](#16-incident-management--troubleshooting)
17. [Leadership, Mentoring & Team Management](#17-leadership-mentoring--team-management)
18. [Scenario-Based Questions](#18-scenario-based-questions)

---

## 1. CI/CD Fundamentals

---

### Q1. What is CI/CD and why is it important?

**Answer:**

CI/CD stands for **Continuous Integration** and **Continuous Delivery/Deployment**.

Think of it like a factory assembly line. Every time a developer writes code and pushes it, CI/CD automatically picks it up, tests it, builds it, and delivers it — without anyone doing it manually.

- **Continuous Integration (CI):** Every time a developer pushes code, the system automatically runs tests and builds the application to make sure nothing is broken.
- **Continuous Delivery (CD):** After CI passes, the code is automatically prepared and deployed to environments like dev, staging, and production.

**Why it matters:**
- Without CI/CD, developers manually build, test, and deploy — which is slow and error-prone.
- With CI/CD, a code change can go from a developer's laptop to production in minutes, automatically and reliably.

**AzureShop Example:**

In our AzureShop project, every time a developer pushes code to the `user-service`, an Azure Pipeline automatically:
1. Runs unit tests
2. Builds the Docker image
3. Scans it for vulnerabilities
4. Pushes it to Azure Container Registry (ACR)
5. Deploys it to the AKS dev namespace

This whole process runs without anyone pressing a button.

---

### Q2. What is the difference between Continuous Delivery and Continuous Deployment?

**Answer:**

Both sound similar but there is one key difference:

| | Continuous Delivery | Continuous Deployment |
|---|---|---|
| Definition | Code is always ready to deploy, but a human presses the button for production | Code is automatically deployed to production without human approval |
| Human involvement | Yes, for production | No — fully automated |
| Risk | Lower, human reviews before prod | Higher, needs very strong test coverage |

**Simple analogy:**
- **Continuous Delivery** = The pizza is ready in the box. You decide when to open it.
- **Continuous Deployment** = The pizza is delivered to your table automatically as soon as it's ready.

**AzureShop Example:**

In AzureShop:
- Deploy to **dev** → fully automatic (Continuous Deployment)
- Deploy to **staging** → automatic after dev passes
- Deploy to **prod** → requires **manual approval** from the team lead (Continuous Delivery)

This is because production affects real users, so we want a human to review before releasing.

---

### Q3. What are the key stages of a CI/CD pipeline?

**Answer:**

A complete CI/CD pipeline has these stages:

```
Code Push → Build → Test → Code Quality → Security Scan → Package → Deploy Dev → Deploy Staging → Deploy Prod
```

**Each stage explained:**

1. **Source** — Developer pushes code to Git (Azure Repos / GitHub)
2. **Build** — Compile the code or install dependencies (npm install, pip install, mvn package)
3. **Unit Test** — Run automated tests to check if individual functions work correctly
4. **Code Quality (SAST)** — Tools like SonarCloud scan for bugs, code smells, security issues
5. **Docker Build** — Build the Docker image (package the app into a container)
6. **Security Scan** — Trivy or Defender scans the Docker image for vulnerabilities
7. **Push to Registry** — Push the image to ACR (Azure Container Registry)
8. **Deploy to Dev** — Automatically deploy to the dev Kubernetes namespace
9. **Integration Tests** — Test if services work together
10. **Deploy to Staging** — Deploy to staging for QA testing
11. **Manual Gate** — Team lead approves
12. **Deploy to Prod** — Deploy to production using blue/green or canary strategy

**AzureShop Example:**

The `user-service-ci.yaml` pipeline in AzureShop does exactly this. After every code push to `feature/*` or `dev`, all 12 steps run automatically.

---

### Q4. What is a pipeline trigger and what types are there?

**Answer:**

A **trigger** is the event that starts a pipeline automatically.

**Types of triggers:**

| Trigger Type | When it fires | Example |
|---|---|---|
| Push trigger | When code is pushed to a branch | Push to `main` starts prod deploy |
| PR trigger | When a Pull Request is opened | PR to `dev` runs CI checks |
| Scheduled trigger | At a fixed time (cron) | Every night at 2AM |
| Manual trigger | When a human clicks "Run" | Prod deployment requires manual click |
| Pipeline trigger | When another pipeline finishes | CD starts when CI finishes |

**AzureShop Example:**

```yaml
# In user-service-ci.yaml
trigger:
  branches:
    include:
      - dev
      - feature/*

pr:
  branches:
    include:
      - dev
```

This means: Run CI every time someone pushes to `dev` or any `feature/*` branch, and also on every PR targeting `dev`.

---

## 2. Jenkins & Jenkins Pipelines

---

### Q5. What is Jenkins and why is it used?

**Answer:**

Jenkins is an open-source automation server. Think of it as a **robot that runs your build, test, and deploy steps automatically** whenever you tell it to.

It is the most widely used CI/CD tool in the industry, especially in on-premise and enterprise environments.

**Key features:**
- Free and open source
- Huge plugin ecosystem (1800+ plugins)
- Supports any language, any cloud, any tool
- Can run on-prem or in cloud
- Supports distributed builds (multiple agents)

**Simple analogy:**
Jenkins is like a factory manager. You give it a list of tasks (build, test, deploy). Every time new code arrives, the manager automatically assigns workers (agents) to complete the tasks in order.

---

### Q6. What is a Jenkinsfile and what are the two types of Jenkins Pipelines?

**Answer:**

A **Jenkinsfile** is a text file that contains the definition of a Jenkins pipeline written in Groovy. You store it in your code repository, so the pipeline definition is version-controlled alongside the code.

**Two types of Jenkins Pipelines:**

**1. Declarative Pipeline** (recommended, simpler):
```groovy
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
        stage('Docker Build') {
            steps {
                sh 'docker build -t user-service:latest .'
            }
        }
    }
}
```

**2. Scripted Pipeline** (older, more flexible but complex):
```groovy
node {
    stage('Build') {
        sh 'mvn clean package'
    }
    stage('Test') {
        sh 'mvn test'
    }
}
```

**Key difference:** Declarative has a fixed, clean structure. Scripted gives you full Groovy power but is harder to read.

**AzureShop context:**
AzureShop uses Azure Pipelines (YAML-based), which works similarly to Jenkins Declarative Pipelines. If AzureShop were using Jenkins, the Jenkinsfile would be stored in the `pipelines/` folder.

---

### Q7. What are Jenkins Agents and why do you need them?

**Answer:**

A **Jenkins Agent** (also called a node or slave) is a machine that actually runs the build jobs. The **Jenkins Master** (controller) manages and distributes work to agents.

**Why you need agents:**
- The master should only manage jobs, not run them (separation of concern)
- Multiple builds can run in parallel on different agents
- Different agents can have different tools installed (one for Java, one for Python, one for Docker)
- Agents can be scaled up/down based on demand

**Types of agents:**
- **Permanent agents** — always-on machines registered in Jenkins
- **Dynamic agents** — created on-demand in Kubernetes or cloud (ephemeral)

**Example Jenkinsfile with agent:**
```groovy
pipeline {
    agent {
        kubernetes {
            yaml '''
            apiVersion: v1
            kind: Pod
            spec:
              containers:
              - name: docker
                image: docker:24-dind
              - name: maven
                image: maven:3.9-eclipse-temurin-17
            '''
        }
    }
    stages {
        stage('Build') {
            steps {
                container('maven') {
                    sh 'mvn clean package'
                }
            }
        }
    }
}
```

**AzureShop context:**
AzureShop uses Microsoft-hosted agents (`ubuntu-latest`) in Azure Pipelines. These are equivalent to dynamic Jenkins agents — a fresh virtual machine spins up for each build and is destroyed after.

---

### Q8. How do you handle secrets in Jenkins?

**Answer:**

You should **never hardcode secrets** (passwords, API keys, tokens) in a Jenkinsfile. Jenkins provides a **Credentials Store** to manage secrets safely.

**Ways to store and use secrets:**

1. **Jenkins Credentials Store** — Store secrets in Jenkins UI (Manage Jenkins → Credentials)
2. **Environment variables** — Inject secrets as env vars at runtime
3. **Vault integration** — Pull secrets from HashiCorp Vault or Azure Key Vault

**Example:**
```groovy
pipeline {
    agent any
    environment {
        ACR_PASSWORD = credentials('acr-password-secret')
    }
    stages {
        stage('Push to ACR') {
            steps {
                sh '''
                    docker login acrazureshop.azurecr.io \
                      -u azureshop -p $ACR_PASSWORD
                    docker push acrazureshop.azurecr.io/user-service:latest
                '''
            }
        }
    }
}
```

The `credentials('acr-password-secret')` pulls the secret from Jenkins' credential store — it is never visible in logs (masked automatically).

**AzureShop equivalent:**
In AzureShop's Azure Pipelines, secrets are stored in **Variable Groups** linked to **Azure Key Vault**. The pipeline pulls secrets at runtime the same way.

---

### Q9. What is a shared library in Jenkins?

**Answer:**

A **Shared Library** is a reusable collection of Groovy code that multiple Jenkinsfiles can use. Instead of copy-pasting the same pipeline steps across 8 services, you write them once in a shared library and call them from each Jenkinsfile.

**Why it matters:**
- No duplication — write once, use everywhere
- When you need to update a step (e.g., add a new security scan), update in one place
- Keeps individual Jenkinsfiles small and readable

**Structure of a shared library:**
```
jenkins-shared-lib/
├── vars/
│   ├── buildDocker.groovy      # function to build Docker image
│   ├── pushToACR.groovy        # function to push to registry
│   └── deployToKubernetes.groovy # function to deploy via Helm
├── src/
│   └── com/azureshop/Utils.groovy
└── resources/
    └── helm-values-template.yaml
```

**Usage in Jenkinsfile:**
```groovy
@Library('azureshop-shared-lib') _

pipeline {
    agent any
    stages {
        stage('Build') { steps { buildDocker('user-service') } }
        stage('Push')  { steps { pushToACR('user-service') } }
        stage('Deploy'){ steps { deployToKubernetes('user-service', 'dev') } }
    }
}
```

**AzureShop equivalent:**
AzureShop uses **pipeline templates** in `pipelines/templates/` — `build-template.yaml`, `deploy-template.yaml`. These serve the same purpose as Jenkins Shared Libraries.

---

## 3. Docker & Containerization

---

### Q10. What is Docker and why do we use it?

**Answer:**

Docker is a tool that lets you **package your application and all its dependencies into a single portable unit called a container**.

**The problem Docker solves:**
Before Docker, developers would say "it works on my machine" but it would fail in production because:
- Different OS versions
- Different library versions
- Different environment variables

Docker solves this by packaging everything the app needs — the code, runtime, libraries, and config — into one image that runs the same everywhere.

**Simple analogy:**
A Docker container is like a **shipping container**. A shipping container can carry any goods (your app), and any ship (server), crane (Kubernetes), or truck (your laptop) can handle it — because the container is standardized.

**AzureShop Example:**

The `user-service` in AzureShop runs on Node.js. Its Dockerfile packages:
- Node.js runtime
- npm dependencies
- The application code

That same Docker image runs identically on a developer's Mac, on the CI/CD agent, and on AKS in production.

---

### Q11. What is a multi-stage Dockerfile and why is it important?

**Answer:**

A **multi-stage Dockerfile** uses multiple `FROM` statements. Each stage does a different job, and only the final stage becomes the actual image. This keeps the final image small and clean.

**Why it matters:**
- Without multi-stage: The image contains build tools (compilers, package managers) which are not needed at runtime — making the image large and insecure.
- With multi-stage: Build tools are used in stage 1, only the compiled output is copied to stage 2 (the runtime image).

**AzureShop Example — user-service Dockerfile:**

```dockerfile
# ===== Stage 1: Build =====
FROM node:20-alpine AS builder
WORKDIR /app

# Copy package files and install ALL dependencies (including dev)
COPY package*.json ./
RUN npm ci

# Copy source code
COPY . .

# ===== Stage 2: Runtime =====
FROM node:20-alpine AS runtime
WORKDIR /app

# Only copy production dependencies
COPY package*.json ./
RUN npm ci --only=production

# Copy built app from builder stage
COPY --from=builder /app/src ./src

# Run as non-root user for security
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

EXPOSE 3001
CMD ["node", "src/index.js"]
```

**Result:**
- Stage 1 image: ~400MB (has all dev tools)
- Final image: ~120MB (only runtime — 3x smaller)
- Smaller image = faster pulls, less attack surface, cheaper storage

---

### Q12. What is the difference between a Docker image and a Docker container?

**Answer:**

| | Docker Image | Docker Container |
|---|---|---|
| What is it? | A read-only blueprint/template | A running instance of an image |
| Analogy | A cake recipe | The actual cake |
| Stored where? | In a registry (ACR, Docker Hub) | On the host machine (running) |
| Can you run it? | No, you build it | Yes, it runs your application |
| How many? | One image | Many containers from one image |

**AzureShop Example:**

1. Azure Pipeline builds `user-service` Docker **image** and pushes it to ACR:
   `acrazureshop.azurecr.io/user-service:build-42`

2. AKS pulls that image and runs 3 **containers** (replicas) of it simultaneously across different nodes.

So: **one image, three running containers**.

---

### Q13. What is Docker Compose and when do you use it?

**Answer:**

Docker Compose is a tool that lets you **define and run multiple containers together** using a single YAML file. It's used for local development to spin up all services at once.

**When to use it:**
- Local development — run all microservices + databases on your laptop
- Integration testing — spin up a full environment temporarily
- **NOT for production** — use Kubernetes for production

**AzureShop docker-compose.yml example:**

```yaml
version: '3.8'

services:
  user-service:
    build: ./services/user-service
    ports:
      - "3001:3001"
    environment:
      - DB_CONNECTION_STRING=Server=sql-server;Database=db-users;...
    depends_on:
      - sql-server

  product-service:
    build: ./services/product-service
    ports:
      - "3002:3002"
    depends_on:
      - cosmos-emulator

  api-gateway:
    build: ./services/api-gateway
    ports:
      - "8080:8080"
    depends_on:
      - user-service
      - product-service

  sql-server:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      - SA_PASSWORD=YourPassword123!
      - ACCEPT_EULA=Y
    ports:
      - "1433:1433"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
```

**Usage:**
```bash
docker-compose up       # Start all services
docker-compose down     # Stop all services
docker-compose logs -f  # View all logs
```

---

### Q14. What is a .dockerignore file?

**Answer:**

`.dockerignore` tells Docker **which files to exclude** when building an image. Just like `.gitignore` tells Git what not to track.

**Why it matters:**
- Keeps images small (don't copy `node_modules`, test files, `.git` into image)
- Faster build times (less data to send to Docker daemon)
- Keeps secrets out of images (never copy `.env` files)

**AzureShop `.dockerignore` for user-service:**

```
node_modules
npm-debug.log
.git
.gitignore
*.md
.env
.env.*
coverage/
tests/
Dockerfile
.dockerignore
```

---

## 4. Kubernetes & AKS

---

### Q15. What is Kubernetes and why do we need it?

**Answer:**

Kubernetes (K8s) is a system that **automatically manages, scales, and heals your Docker containers** in production.

**The problem Kubernetes solves:**

Imagine you have 8 microservices in AzureShop, each needing 3 replicas = 24 containers. Now:
- What if one container crashes? → Kubernetes restarts it automatically
- What if traffic increases? → Kubernetes scales up containers automatically
- What if a node (server) dies? → Kubernetes moves containers to healthy nodes automatically
- How do you update without downtime? → Kubernetes does rolling updates

Without Kubernetes, you'd need a team of people doing this manually 24/7.

**Simple analogy:**
Kubernetes is like an **air traffic controller**. There are hundreds of planes (containers) in the sky. The controller automatically assigns runways (nodes), reroutes planes when weather is bad (node failure), and lands/launches planes as demand changes.

---

### Q16. Explain the key Kubernetes objects — Pod, Deployment, Service, Ingress.

**Answer:**

**Pod:**
- The smallest unit in Kubernetes
- A Pod contains one or more containers
- Pods are temporary — if a pod dies, Kubernetes creates a new one

```yaml
# A Pod running user-service
apiVersion: v1
kind: Pod
metadata:
  name: user-service-pod
spec:
  containers:
  - name: user-service
    image: acrazureshop.azurecr.io/user-service:v1.0.0
    ports:
    - containerPort: 3001
```

**Deployment:**
- Manages multiple replicas of Pods
- Handles rolling updates and rollbacks
- Always maintains the desired number of running pods

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
spec:
  replicas: 3          # Run 3 copies always
  selector:
    matchLabels:
      app: user-service
  template:
    spec:
      containers:
      - name: user-service
        image: acrazureshop.azurecr.io/user-service:v1.0.0
```

**Service:**
- Gives your Pods a stable network address (IP + DNS name)
- Pods die and get new IPs, but Service IP stays the same
- Acts as a load balancer between Pods

```yaml
apiVersion: v1
kind: Service
metadata:
  name: user-service
spec:
  selector:
    app: user-service
  ports:
  - port: 3001
    targetPort: 3001
  type: ClusterIP      # Only accessible inside the cluster
```

**Ingress:**
- Routes external HTTP/HTTPS traffic to Services inside the cluster
- One entry point that routes to many services based on URL path

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: azureshop-ingress
spec:
  rules:
  - host: azureshop.com
    http:
      paths:
      - path: /api/users
        pathType: Prefix
        backend:
          service:
            name: user-service
            port:
              number: 3001
      - path: /api/products
        pathType: Prefix
        backend:
          service:
            name: product-service
            port:
              number: 3002
```

---

### Q17. What is a Horizontal Pod Autoscaler (HPA)?

**Answer:**

HPA automatically **scales the number of Pods up or down** based on CPU usage, memory, or custom metrics.

**How it works:**
- You define a target (e.g., keep CPU below 70%)
- HPA checks metrics every 15 seconds
- If CPU is above 70%, it adds more Pods
- If CPU drops, it removes Pods (down to the minimum)

**AzureShop Example:**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: user-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: user-service
  minReplicas: 2      # Never go below 2
  maxReplicas: 10     # Never go above 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70   # Scale when CPU > 70%
```

**Real scenario:**
During a sale on AzureShop, traffic to `user-service` spikes. HPA detects CPU crossing 70% and automatically adds pods (2 → 5 → 8). When the sale ends, traffic drops and HPA scales back down to 2.

---

### Q18. What is a Liveness Probe and Readiness Probe?

**Answer:**

These are health checks Kubernetes runs on your containers.

**Liveness Probe — "Is this container alive?"**
- If the liveness probe fails, Kubernetes **restarts** the container
- Use it to detect if your app is stuck (deadlock, OOM) and needs a restart

**Readiness Probe — "Is this container ready to receive traffic?"**
- If the readiness probe fails, Kubernetes **removes the pod from the Service** (no traffic sent to it)
- Use it so traffic never goes to a pod that's still starting up or temporarily busy

**AzureShop Example:**

```yaml
containers:
- name: user-service
  image: acrazureshop.azurecr.io/user-service:v1.0.0
  
  livenessProbe:
    httpGet:
      path: /health      # Calls GET /health endpoint
      port: 3001
    initialDelaySeconds: 30   # Wait 30s before first check
    periodSeconds: 10         # Check every 10s
    failureThreshold: 3       # Restart after 3 consecutive failures
    
  readinessProbe:
    httpGet:
      path: /health
      port: 3001
    initialDelaySeconds: 10   # Start checking after 10s
    periodSeconds: 5          # Check every 5s
    failureThreshold: 3       # Remove from service after 3 failures
```

**Simple analogy:**
- Liveness = "Are you alive?" → No → Restart you
- Readiness = "Are you ready to work?" → No → Don't send customers to you yet

---

### Q19. What are Namespaces in Kubernetes?

**Answer:**

Namespaces are **virtual clusters within your Kubernetes cluster**. They separate resources belonging to different environments or teams.

**Why use namespaces:**
- Isolate dev, staging, prod environments in the same cluster
- Apply resource quotas per namespace (dev gets fewer resources than prod)
- Control access — developers can access `dev` namespace but not `prod`

**AzureShop namespaces:**

```
aks-azureshop cluster
├── namespace: dev          ← development environment
├── namespace: staging      ← QA/testing environment
├── namespace: prod         ← production environment
└── namespace: monitoring   ← Grafana, Prometheus tools
```

```bash
# Create namespaces
kubectl create namespace dev
kubectl create namespace staging
kubectl create namespace prod
kubectl create namespace monitoring

# Deploy user-service to dev namespace
kubectl apply -f user-service-deployment.yaml -n dev

# See all pods in dev
kubectl get pods -n dev
```

---

## 5. Helm Charts

---

### Q20. What is Helm and why do we use it?

**Answer:**

Helm is the **package manager for Kubernetes**. Just like `apt` installs software on Ubuntu or `npm` installs Node.js packages, Helm installs and manages applications on Kubernetes.

**The problem Helm solves:**

AzureShop has 8 microservices. Each service needs:
- Deployment YAML
- Service YAML
- HPA YAML
- ConfigMap YAML
- NetworkPolicy YAML

That's 40+ YAML files. And if you need to deploy to 3 environments (dev/staging/prod), you need 120 YAML files — most of which are identical except for a few values.

Helm lets you write the YAML once as a **template** and inject different values for each environment.

---

### Q21. What is the structure of a Helm chart?

**Answer:**

```
helm/charts/user-service/
├── Chart.yaml          # Chart metadata (name, version, description)
├── values.yaml         # Default configuration values
├── values-dev.yaml     # Dev-specific overrides
├── values-prod.yaml    # Prod-specific overrides
└── templates/
    ├── deployment.yaml     # Deployment template
    ├── service.yaml        # Service template
    ├── hpa.yaml            # HPA template
    ├── configmap.yaml      # ConfigMap template
    ├── networkpolicy.yaml  # NetworkPolicy template
    └── _helpers.tpl        # Reusable template snippets
```

**Chart.yaml:**
```yaml
apiVersion: v2
name: user-service
description: AzureShop User Service Helm Chart
version: 1.0.0
appVersion: "1.0.0"
```

**values.yaml (defaults):**
```yaml
replicaCount: 2
image:
  repository: acrazureshop.azurecr.io/user-service
  tag: "latest"
  pullPolicy: IfNotPresent
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
hpa:
  minReplicas: 2
  maxReplicas: 10
  targetCPU: 70
```

**templates/deployment.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-user-service
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
      - name: user-service
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        resources:
          requests:
            cpu: {{ .Values.resources.requests.cpu }}
            memory: {{ .Values.resources.requests.memory }}
```

**Deploy with Helm:**
```bash
# Deploy to dev with dev values
helm upgrade --install user-service ./helm/charts/user-service \
  -f helm/values/dev.yaml \
  --namespace dev \
  --set image.tag=build-42

# Deploy to prod with prod values
helm upgrade --install user-service ./helm/charts/user-service \
  -f helm/values/prod.yaml \
  --namespace prod \
  --set image.tag=build-42
```

---

### Q22. What is `helm upgrade --install` and how does rollback work?

**Answer:**

**`helm upgrade --install`** means:
- If the release doesn't exist → **install** it
- If the release exists → **upgrade** it

This single command handles both first-time install and updates, which is perfect for CI/CD pipelines.

**Rollback:**

Helm keeps a history of every deployment (called a revision). If something goes wrong, you can roll back to any previous revision instantly.

```bash
# See release history
helm history user-service -n prod
# REVISION  STATUS     CHART              DESCRIPTION
# 1         superseded user-service-1.0.0 Install complete
# 2         superseded user-service-1.0.1 Upgrade complete
# 3         deployed   user-service-1.0.2 Upgrade complete
# 4         failed     user-service-1.0.3 Upgrade failed

# Rollback to revision 3 (previous working version)
helm rollback user-service 3 -n prod
```

**AzureShop pipeline rollback step:**
```yaml
- task: HelmDeploy@0
  inputs:
    command: rollback
    releaseName: user-service
    arguments: "3"
    namespace: prod
```

---

## 6. Infrastructure as Code — Terraform

---

### Q23. What is Infrastructure as Code (IaC) and why is it important?

**Answer:**

Infrastructure as Code means **writing code to create and manage infrastructure** (servers, databases, networks) instead of clicking buttons in a portal manually.

**Why it matters:**

| Manual (ClickOps) | Infrastructure as Code (Terraform) |
|---|---|
| Click through Azure portal | Write `.tf` files |
| Hard to reproduce | Run `terraform apply` — identical infra every time |
| No audit trail | Git history shows every change |
| Error-prone | Code is reviewed, tested, version-controlled |
| Takes hours | Takes minutes |

**AzureShop Example:**

Instead of manually creating the AKS cluster through the Azure portal, we wrote this Terraform code:

```hcl
resource "azurerm_kubernetes_cluster" "aks" {
  name                = "aks-azureshop-dev"
  location            = "eastus"
  resource_group_name = "rg-azureshop-dev"
  dns_prefix          = "azureshop-dev"

  default_node_pool {
    name       = "system"
    node_count = 2
    vm_size    = "Standard_D2s_v3"
  }

  identity {
    type = "SystemAssigned"
  }

  tags = {
    Environment = "dev"
    Project     = "AzureShop"
    ManagedBy   = "Terraform"
  }
}
```

Run `terraform apply` and the AKS cluster is created in minutes. Run it again — nothing changes (idempotent). Destroy it with `terraform destroy`.

---

### Q24. What is Terraform state and why is remote state important?

**Answer:**

**Terraform state** is a file (`terraform.tfstate`) that records what infrastructure Terraform has created. It's how Terraform knows:
- What resources exist
- What changes need to be made
- What to delete when you run `terraform destroy`

**Why remote state:**

If state is stored locally:
- Only the person who ran Terraform has the state file
- Team members can't collaborate
- If the laptop dies, state is lost

**Remote state** stores the state file in a shared location (like Azure Blob Storage) that the whole team and CI/CD pipelines can access.

**AzureShop Terraform backend configuration:**

```hcl
# backend.tf
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-azureshop-dev"
    storage_account_name = "myprojectazshoptfstate"
    container_name       = "tfstate"
    key                  = "dev.tfstate"   # Separate state per environment
  }
}
```

**State locking:** When one person or pipeline runs Terraform, the state file is locked so nobody else can run it simultaneously and cause conflicts. Azure Blob Storage provides this automatically.

---

### Q25. What is a Terraform module and why do you use it?

**Answer:**

A Terraform module is a **reusable, self-contained block of Terraform code** that creates a specific piece of infrastructure.

**Why modules:**
- Don't repeat yourself — write VNet configuration once, reuse for dev/staging/prod
- Consistent infrastructure — everyone uses the same tested module
- Easy to update — change the module, all users get the update

**AzureShop module structure:**

```
infra/
├── modules/
│   ├── networking/     ← Creates VNet, subnets, NSGs
│   ├── aks/            ← Creates AKS cluster
│   ├── acr/            ← Creates Container Registry
│   ├── databases/      ← Creates SQL, CosmosDB, Redis
│   ├── keyvault/       ← Creates Key Vault
│   └── monitoring/     ← Creates Log Analytics, Grafana
└── environments/
    ├── dev/
    │   └── main.tf     ← Calls modules with dev values
    └── prod/
        └── main.tf     ← Calls modules with prod values
```

**Calling a module:**
```hcl
# environments/dev/main.tf
module "networking" {
  source              = "../../modules/networking"
  environment         = "dev"
  vnet_address_space  = ["10.0.0.0/8"]
  location            = "eastus"
  resource_group_name = azurerm_resource_group.main.name
}

module "aks" {
  source              = "../../modules/aks"
  environment         = "dev"
  subnet_id           = module.networking.aks_subnet_id
  log_analytics_id    = module.monitoring.workspace_id
}
```

---

## 7. Scripting — Python, Groovy, Shell

---

### Q26. Write a shell script to check if a Kubernetes pod is running and restart it if not.

**Answer:**

```bash
#!/bin/bash
# check-pod-health.sh — Check pod health in AzureShop

NAMESPACE="dev"
SERVICE="user-service"
EXPECTED_REPLICAS=2

echo "Checking health of $SERVICE in namespace $NAMESPACE..."

# Get the number of running replicas
RUNNING=$(kubectl get deployment $SERVICE -n $NAMESPACE \
  -o jsonpath='{.status.readyReplicas}' 2>/dev/null)

# If no output, deployment might not exist
if [ -z "$RUNNING" ]; then
    echo "ERROR: Deployment $SERVICE not found in $NAMESPACE"
    exit 1
fi

echo "Running replicas: $RUNNING / $EXPECTED_REPLICAS"

# Check if replicas are below expected
if [ "$RUNNING" -lt "$EXPECTED_REPLICAS" ]; then
    echo "WARNING: Only $RUNNING/$EXPECTED_REPLICAS pods running. Restarting..."
    kubectl rollout restart deployment/$SERVICE -n $NAMESPACE
    
    # Wait for rollout to complete
    kubectl rollout status deployment/$SERVICE -n $NAMESPACE --timeout=120s
    
    if [ $? -eq 0 ]; then
        echo "SUCCESS: $SERVICE restarted successfully"
    else
        echo "ERROR: Restart failed. Investigate manually."
        exit 1
    fi
else
    echo "OK: All $EXPECTED_REPLICAS replicas are healthy"
fi
```

---

### Q27. Write a Python script to check Azure resource health and send an alert.

**Answer:**

```python
#!/usr/bin/env python3
# resource-health-check.py — Check AzureShop resource health

import subprocess
import json
import sys
from datetime import datetime

RESOURCE_GROUP = "rg-azureshop-dev"
SERVICES = [
    {"name": "aks-azureshop", "type": "AKS"},
    {"name": "acrazureshop", "type": "ACR"},
    {"name": "kv-azureshop-6a6c", "type": "KeyVault"},
]

def run_az_command(command):
    """Run an Azure CLI command and return parsed JSON output."""
    result = subprocess.run(
        command, shell=True, capture_output=True, text=True
    )
    if result.returncode != 0:
        print(f"ERROR: {result.stderr}")
        return None
    return json.loads(result.stdout)

def check_aks_health(cluster_name):
    """Check if AKS cluster is running and nodes are ready."""
    nodes = run_az_command(
        f"az aks show -g {RESOURCE_GROUP} -n {cluster_name} "
        f"--query 'powerState.code' -o json"
    )
    return nodes == "Running"

def check_acr_health(acr_name):
    """Check if ACR is accessible."""
    result = run_az_command(
        f"az acr show -g {RESOURCE_GROUP} -n {acr_name} "
        f"--query 'provisioningState' -o json"
    )
    return result == "Succeeded"

def main():
    print(f"\n{'='*50}")
    print(f"AzureShop Health Check — {datetime.now().strftime('%Y-%m-%d %H:%M')}")
    print(f"{'='*50}")

    all_healthy = True
    results = []

    for service in SERVICES:
        name = service["name"]
        svc_type = service["type"]

        if svc_type == "AKS":
            healthy = check_aks_health(name)
        elif svc_type == "ACR":
            healthy = check_acr_health(name)
        else:
            healthy = True  # Simplified for other types

        status = "HEALTHY" if healthy else "UNHEALTHY"
        results.append({"service": name, "type": svc_type, "status": status})

        if not healthy:
            all_healthy = False
        print(f"[{status}] {svc_type}: {name}")

    print(f"\nOverall Status: {'ALL SYSTEMS HEALTHY' if all_healthy else 'ISSUES DETECTED'}")
    sys.exit(0 if all_healthy else 1)

if __name__ == "__main__":
    main()
```

---

### Q28. What is Groovy and how is it used in Jenkins?

**Answer:**

Groovy is a scripting language that runs on the Java Virtual Machine (JVM). Jenkins uses Groovy as its scripting language for:
- Writing Jenkinsfiles (pipeline definitions)
- Writing Shared Libraries
- Jenkins System Groovy Scripts (admin automation)

**Key Groovy concepts for Jenkins:**

```groovy
// Variables
def serviceName = "user-service"
def buildId = env.BUILD_ID

// If/else
if (env.BRANCH_NAME == "main") {
    deployToProd()
} else {
    deployToDev()
}

// Loop through services
def services = ["user-service", "product-service", "cart-service"]
services.each { service ->
    sh "docker build -t ${service}:${buildId} ./services/${service}"
}

// Try/catch for error handling
try {
    sh "helm upgrade --install ${serviceName} ./helm/charts/${serviceName}"
    echo "Deployment successful"
} catch (Exception e) {
    echo "Deployment failed: ${e.message}"
    currentBuild.result = 'FAILURE'
    throw e
}

// Parallel execution
parallel (
    "Build user-service": {
        sh "docker build -t user-service ./services/user-service"
    },
    "Build product-service": {
        sh "docker build -t product-service ./services/product-service"
    }
)
```

---

## 8. Git & GitHub Enterprise

---

### Q29. What is GitFlow and how do you use it in a DevOps project?

**Answer:**

GitFlow is a **branching strategy** — a set of rules about how and when to create branches in Git. It keeps code organized and release-ready.

**AzureShop branch strategy:**

```
main        ← Production-only. Only release/* and hotfix/* merge here.
dev         ← Integration branch. All features merge here first.
feature/*   ← Individual features (e.g., feature/add-cart-endpoint)
release/*   ← Release candidates (e.g., release/v1.2.0)
hotfix/*    ← Emergency fixes for production (e.g., hotfix/fix-login-bug)
```

**Workflow example:**

```bash
# Developer starts a new feature
git checkout dev
git pull origin dev
git checkout -b feature/add-payment-gateway

# Work on the feature...
git add .
git commit -m "Add Stripe payment gateway integration"
git push origin feature/add-payment-gateway

# Open a Pull Request: feature/add-payment-gateway → dev
# CI pipeline runs automatically on the PR
# After review and CI pass → merge to dev

# When ready for release
git checkout -b release/v1.2.0 from dev
# Final testing on release branch
# Merge release/v1.2.0 → main AND back into dev
```

---

### Q30. What is a Pull Request and why is it important in DevOps?

**Answer:**

A Pull Request (PR) is a request to merge code from one branch into another. It's a **code review checkpoint** before code reaches shared branches.

**Why PRs are critical in DevOps:**
1. **Code Review** — Other engineers review the code for bugs, security issues, code quality
2. **Automated CI** — The CI pipeline runs automatically on every PR (tests, security scans, lint checks)
3. **Discussion** — Team discusses the approach before it's merged
4. **Audit trail** — Every change has a documented reason and reviewer

**AzureShop branch policies (configured in Azure DevOps):**
- Minimum 1 reviewer must approve
- CI pipeline must pass (all tests green)
- No direct pushes to `main` or `dev` — PRs only
- Linked work item required (connects code to a ticket)

---

### Q31. What is git rebase vs git merge and when to use which?

**Answer:**

Both combine changes from two branches, but differently.

**git merge** — Creates a **merge commit** that ties two branches together. History shows exactly when and how branches diverged and merged.

```bash
git checkout dev
git merge feature/add-cart
# Result: a merge commit is created showing the merge happened
```

**git rebase** — Replays your commits on top of another branch as if you had started there. **History is linear** (cleaner), but rewrite of history.

```bash
git checkout feature/add-cart
git rebase dev
# Result: your feature commits appear as if they were written after the latest dev
```

**When to use:**
- **merge** → For merging feature branches into `dev` or `main` (preserve the history of what happened)
- **rebase** → For updating your local feature branch with latest `dev` changes (keep your branch up to date without cluttering history)

**Golden rule: Never rebase branches that others are working on.**

---

## 9. Monitoring & Observability

---

### Q32. What is Observability and what are the three pillars?

**Answer:**

Observability means being able to understand **what is happening inside your system** by looking at data it produces from the outside — without having to guess or dig into code.

**The Three Pillars:**

**1. Metrics** — Numbers that measure your system over time
- Examples: CPU usage, request count per second, error rate, response time
- Tool in AzureShop: Azure Monitor, Grafana

**2. Logs** — Text records of events that happened
- Examples: "User login failed", "Order placed: order-123", "DB connection timeout"
- Tool in AzureShop: Azure Log Analytics (KQL queries)

**3. Traces** — Track a single request as it flows through multiple services
- Examples: A user places an order → trace shows it went through api-gateway → order-service → payment-service → notification-service with timing for each hop
- Tool in AzureShop: Application Insights (distributed tracing)

**Simple analogy:**
If your app is a car:
- Metrics = speedometer, fuel gauge, temperature (overall numbers)
- Logs = dashboard warning lights with messages ("Check Engine at 3:42 PM")
- Traces = GPS showing exactly which roads you took and how long each road took

---

### Q33. What are SLI, SLO, and SLA?

**Answer:**

**SLI (Service Level Indicator)** — A specific metric that measures service performance.
- Example: "What % of requests returned a successful response in under 500ms?"

**SLO (Service Level Objective)** — A target value for an SLI. An internal goal.
- Example: "99.9% of requests should succeed with response time < 500ms"

**SLA (Service Level Agreement)** — A contract with customers about service availability. If violated, there are financial penalties.
- Example: "We guarantee 99.5% uptime. If we miss it, we give you a 10% credit."

**Relationship:** SLI measures it → SLO is the internal target → SLA is the external commitment

**AzureShop example:**
- SLI: % of successful `/api/orders` responses (HTTP 200) measured by Application Insights
- SLO: 99.9% success rate, P95 response time < 300ms
- SLA: 99.5% availability guaranteed to WSI business users

**Error Budget:**
SLO of 99.9% means you can be down for 8.7 hours per year. This is your "error budget". If the budget is running low, you stop deploying new features and focus on reliability.

---

### Q34. What KQL queries do you use to monitor AzureShop?

**Answer:**

KQL (Kusto Query Language) is used to query Azure Log Analytics.

**Query 1: Error rate per service (last 24 hours)**
```kql
requests
| where timestamp > ago(24h)
| summarize
    total = count(),
    errors = countif(resultCode >= 400)
    by cloud_RoleName
| extend error_rate = round(errors * 100.0 / total, 2)
| order by error_rate desc
```

**Query 2: Top 5 slowest API endpoints**
```kql
requests
| where timestamp > ago(1h)
| summarize avg_duration = avg(duration) by name
| top 5 by avg_duration desc
```

**Query 3: Pod restarts in the last 1 hour**
```kql
KubePodInventory
| where TimeGenerated > ago(1h)
| where RestartCount > 0
| summarize max(RestartCount) by Name, Namespace
| order by max_RestartCount desc
```

**Query 4: Failed login attempts**
```kql
traces
| where timestamp > ago(1h)
| where message contains "login failed"
| summarize count() by bin(timestamp, 5m), cloud_RoleName
| render timechart
```

---

## 10. Security & DevSecOps

---

### Q35. What is DevSecOps and how do you implement shift-left security?

**Answer:**

**DevSecOps** = Development + Security + Operations. It means **embedding security at every stage** of the software development lifecycle, not just at the end before release.

**Shift-Left Security** means moving security checks as early as possible (to the "left" of the timeline — during development and CI, not after deployment).

**Old way (Shift Right):**
```
Code → Build → Test → Deploy → Security Review ← too late!
```

**Shift-Left (DevSecOps):**
```
Code         → Pre-commit hooks: secret scanning (gitleaks)
↓
CI Pipeline  → SAST: SonarCloud scans for vulnerabilities in code
↓
Docker Build → Container scan: Trivy checks image for CVEs
↓
Deploy       → DAST: OWASP ZAP tests running application
↓
Production   → Runtime: Microsoft Defender monitors continuously
```

**AzureShop DevSecOps implementation:**

1. **Pre-commit (gitleaks)** — Blocks commits with hardcoded secrets
2. **CI: npm audit** — Checks Node.js dependencies for known vulnerabilities
3. **CI: SonarCloud** — Scans for SQL injection, XSS, insecure code patterns
4. **CI: Trivy** — Scans Docker images for OS and library CVEs
5. **AKS: Azure Policy** — Blocks privileged containers from running
6. **Production: Defender for Cloud** — Detects threats in real-time

---

### Q36. What is Azure Key Vault and how do you integrate it with Kubernetes?

**Answer:**

**Azure Key Vault** is a cloud service that securely stores secrets (passwords, API keys, certificates) and controls who can access them.

**Why Key Vault:**
- Secrets never stored in code, config files, or Kubernetes manifests
- Full audit log of who accessed which secret and when
- Automatic rotation of secrets
- RBAC controls who can read/write secrets

**Integration with AKS using CSI Driver:**

The **Secrets Store CSI Driver** allows pods in AKS to mount Key Vault secrets as files or environment variables automatically.

**Step 1: Create a SecretProviderClass**
```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: user-service-secrets
  namespace: dev
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    clientID: "managed-identity-client-id"  # Workload Identity
    keyvaultName: "kv-azureshop-6a6c"
    tenantID: "4c135936-7e4d-4ea6-9816-7d696b51923d"
    objects: |
      array:
        - |
          objectName: sql-admin-password
          objectType: secret
        - |
          objectName: cosmos-primary-key
          objectType: secret
```

**Step 2: Mount in Pod**
```yaml
spec:
  containers:
  - name: user-service
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: user-service-secrets-sync
          key: sql-admin-password
  volumes:
  - name: secrets-store
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: user-service-secrets
```

The pod automatically gets the secret from Key Vault — no secret ever touches a YAML file or environment variable hardcoded anywhere.

---

### Q37. What is RBAC in Kubernetes?

**Answer:**

RBAC (Role-Based Access Control) in Kubernetes controls **who can do what** on which resources.

**Core components:**

- **Role** — Defines permissions (what actions are allowed on what resources) within a namespace
- **ClusterRole** — Same but cluster-wide
- **RoleBinding** — Assigns a Role to a user/service account
- **ClusterRoleBinding** — Assigns a ClusterRole cluster-wide

**AzureShop example:**

```yaml
# Role: Allow reading pods and deployments in dev namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: dev-readonly
  namespace: dev
rules:
- apiGroups: ["apps", ""]
  resources: ["pods", "deployments", "services"]
  verbs: ["get", "list", "watch"]  # Read only — no create/delete

---
# RoleBinding: Give this role to the azureshop-developers group
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-readonly-binding
  namespace: dev
subjects:
- kind: Group
  name: azureshop-developers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: dev-readonly
  apiGroup: rbac.authorization.k8s.io
```

**Result:** Developers can view pods and deployments in `dev` but cannot delete them or access `prod`.

---

## 11. Microservices Architecture

---

### Q38. What is a microservices architecture and how does it differ from monolithic?

**Answer:**

**Monolithic Architecture:**
All features (user management, product catalog, cart, orders, payments) are in one big application. One codebase, one database, one deployment.

**Microservices Architecture:**
Each feature is a separate, independent service. Each service has its own codebase, its own database, and is deployed independently.

| | Monolith | Microservices |
|---|---|---|
| Deployment | Deploy everything at once | Deploy each service independently |
| Scaling | Scale the entire app | Scale only the service that needs it |
| Technology | One language/framework | Each service can use different tech |
| Failure | One bug can take down everything | One service fails, others keep working |
| Team | One large team | Small teams per service |
| Complexity | Simpler to start | Complex to manage |

**AzureShop example:**

AzureShop uses microservices:
- `user-service` (Node.js) → Azure SQL
- `product-service` (Python/FastAPI) → Cosmos DB
- `cart-service` (Node.js) → Redis
- `order-service` (Node.js) → Azure SQL + Service Bus

Each service is independently deployable. If we need to update the product catalog, we only redeploy `product-service` — all other services keep running untouched.

---

### Q39. What is an API Gateway and what does it do?

**Answer:**

An API Gateway is the **single entry point** for all client requests. Instead of clients calling each microservice directly, they call the API Gateway, and it routes the request to the correct service.

**What API Gateway does:**
1. **Routing** — Routes `/api/users` to `user-service`, `/api/products` to `product-service`
2. **Authentication** — Checks JWT tokens before forwarding requests
3. **Rate Limiting** — Blocks clients making too many requests (prevents abuse)
4. **SSL Termination** — Handles HTTPS, forwards plain HTTP internally
5. **Load Balancing** — Distributes traffic across multiple service instances

**AzureShop NGINX API Gateway config:**

```nginx
# Rate limiting zones
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=60r/m;
limit_req_zone $binary_remote_addr zone=auth_limit:10m rate=10r/m;

server {
    listen 8080;

    # Auth endpoints — stricter rate limit
    location /api/users/auth/ {
        limit_req zone=auth_limit burst=5;
        proxy_pass http://user-service:3001;
    }

    # General API
    location /api/users/ {
        limit_req zone=api_limit burst=20;
        proxy_pass http://user-service:3001;
    }

    location /api/products/ {
        limit_req zone=api_limit burst=20;
        proxy_pass http://product-service:3002;
    }

    location /api/cart/ {
        proxy_pass http://cart-service:3003;
    }

    location /api/orders/ {
        proxy_pass http://order-service:3004;
    }

    # Frontend
    location / {
        proxy_pass http://frontend:3000;
    }
}
```

---

### Q40. What is the Service Bus pattern and why does AzureShop use it?

**Answer:**

Azure Service Bus is a **message broker** — it enables services to communicate **asynchronously** (without waiting for each other).

**Why async communication:**

When a user places an order:
1. `order-service` saves the order to the database
2. `payment-service` needs to process payment
3. `notification-service` needs to send a confirmation email

**Synchronous (bad):** order-service calls payment-service, waits for response, then calls notification-service. If payment-service is slow or down, the whole order fails.

**Asynchronous with Service Bus (good):** order-service saves the order and publishes an `order.placed` message to Service Bus. It immediately returns success to the user. Payment and notification services independently pick up the message and process it in their own time.

**AzureShop flow:**
```
User places order
    ↓
order-service saves to Azure SQL
    ↓
order-service publishes "order.placed" to Service Bus topic
    ↓
payment-service (subscriber) picks it up → processes payment
notification-service (subscriber) picks it up → sends email
```

Both payment and notification can fail and retry without affecting the order itself.

---

## 12. Cloud — Azure

---

### Q41. What is AKS and what are its main components?

**Answer:**

AKS (Azure Kubernetes Service) is Microsoft Azure's **managed Kubernetes service**. Azure handles the Kubernetes control plane (master nodes), and you just manage the worker nodes and your applications.

**Main components of AKS:**

| Component | What it does |
|---|---|
| Control Plane | Managed by Azure — API server, scheduler, etcd |
| Node Pools | VMs that run your pods |
| System Node Pool | Runs Kubernetes system pods (DNS, monitoring agents) |
| User Node Pool | Runs your application pods |
| Azure CNI | Networking — pods get real VNet IP addresses |
| Managed Identity | AKS authenticates to Azure services without passwords |
| OIDC Issuer | Enables Workload Identity for pods |
| Container Insights | Sends AKS metrics and logs to Log Analytics |

**AzureShop AKS configuration:**

```hcl
resource "azurerm_kubernetes_cluster" "aks" {
  name = "aks-azureshop-dev"

  # System node pool — critical K8s components
  default_node_pool {
    name                = "system"
    node_count          = 2
    vm_size             = "Standard_D2s_v3"
    only_critical_addons_enabled = true
    zones               = ["1", "2"]
  }
}

# User node pool — runs AzureShop services
resource "azurerm_kubernetes_cluster_node_pool" "user" {
  name                  = "user"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.aks.id
  vm_size               = "Standard_D4s_v3"
  enable_auto_scaling   = true
  min_count             = 2
  max_count             = 10
  zones                 = ["1", "2", "3"]
}
```

---

### Q42. What is Azure Container Registry (ACR) and how does AKS pull images from it?

**Answer:**

ACR is Azure's private Docker registry. Instead of using public Docker Hub, you push your images to ACR — a private, secure, Azure-native registry.

**AKS pulling from ACR:**

AKS uses a **Managed Identity** (not a username/password) to authenticate to ACR. You grant the AKS identity the `AcrPull` role on ACR.

```hcl
# Grant AKS the AcrPull role on ACR
resource "azurerm_role_assignment" "aks_acr_pull" {
  scope                = azurerm_container_registry.acr.id
  role_definition_name = "AcrPull"
  principal_id         = azurerm_kubernetes_cluster.aks.kubelet_identity[0].object_id
}
```

Now AKS can pull any image from ACR without any credential configuration in pod specs.

**ACR image naming:**
```
acrazureshop.azurecr.io/user-service:build-42
^login server           ^repository   ^tag
```

---

## 13. Maven & Build Tools

---

### Q43. What is Maven and what are its key concepts?

**Answer:**

Maven is a **build automation tool** primarily for Java projects. It manages dependencies, compiles code, runs tests, and packages applications.

**Key concepts:**

**pom.xml** (Project Object Model) — The configuration file that defines everything about the project:
```xml
<project>
    <groupId>com.azureshop</groupId>
    <artifactId>order-service</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>

    <dependencies>
        <!-- Spring Boot -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <version>3.2.0</version>
        </dependency>
        <!-- SQL Server driver -->
        <dependency>
            <groupId>com.microsoft.sqlserver</groupId>
            <artifactId>mssql-jdbc</artifactId>
            <version>12.4.0.jre11</version>
        </dependency>
    </dependencies>
</project>
```

**Maven Build Lifecycle:**
```bash
mvn validate    # Validates project is correct
mvn compile     # Compiles source code
mvn test        # Runs unit tests
mvn package     # Creates JAR/WAR file
mvn verify      # Runs integration tests
mvn install     # Installs artifact to local repo
mvn deploy      # Deploys to remote repository
```

**In CI/CD pipeline:**
```yaml
# Jenkins/Azure Pipeline step for Java service
- script: |
    mvn clean package -DskipTests=false
    docker build -t order-service:$(Build.BuildId) .
```

---

## 14. Node.js & npm

---

### Q44. What is npm and how do you manage Node.js dependencies in a DevOps context?

**Answer:**

npm (Node Package Manager) is the package manager for Node.js — like pip for Python or Maven for Java. It installs and manages the JavaScript libraries your application needs.

**Key commands:**

```bash
npm install           # Install all dependencies from package.json
npm ci               # Clean install (faster, used in CI/CD — uses package-lock.json exactly)
npm test             # Run tests
npm run build        # Build the application
npm audit            # Check for security vulnerabilities
npm audit --fix      # Auto-fix vulnerabilities
```

**Why `npm ci` instead of `npm install` in pipelines:**
- `npm install` can update `package-lock.json` and change dependency versions
- `npm ci` always installs exactly what's in `package-lock.json` — deterministic, reproducible builds

**package.json example for user-service:**
```json
{
  "name": "user-service",
  "version": "1.0.0",
  "scripts": {
    "start": "node src/index.js",
    "test": "jest --coverage",
    "lint": "eslint src/"
  },
  "dependencies": {
    "express": "^4.18.2",
    "mssql": "^10.0.0",
    "jsonwebtoken": "^9.0.0",
    "bcryptjs": "^2.4.3"
  },
  "devDependencies": {
    "jest": "^29.0.0",
    "supertest": "^6.3.0"
  }
}
```

**In AzureShop CI pipeline:**
```yaml
- script: |
    npm ci                    # Install exact versions
    npm test                  # Run tests
    npm audit --audit-level=high  # Fail if HIGH or CRITICAL vulnerabilities found
  displayName: 'Test and Security Check'
```

---

## 15. Site Reliability Engineering (SRE)

---

### Q45. What is SRE and how does it relate to DevOps?

**Answer:**

SRE (Site Reliability Engineering) was invented at Google. It applies software engineering principles to operations problems.

**DevOps vs SRE:**
- **DevOps** is a culture and set of practices for collaboration between dev and ops
- **SRE** is a specific implementation of DevOps with concrete practices and tools

**Core SRE principles:**

1. **Error Budgets** — If SLO is 99.9%, you have 0.1% "budget" to fail. When budget runs low, stop new releases and focus on reliability.

2. **Toil reduction** — Toil = repetitive, manual, automatable work. SREs aim to automate away toil. Target: toil < 50% of working time.

3. **Blameless Post-mortems** — When an incident happens, write a post-mortem that focuses on **what failed** (process/system), not **who failed** (person). Goal: learn and prevent recurrence.

4. **Capacity Planning** — Predict and prepare for traffic growth before it happens.

**AzureShop SRE practices:**
- Automated alerts when error rate exceeds budget
- Runbooks for common failure scenarios
- HPA auto-scales before pods get overloaded
- Blue/green deployments eliminate deployment-related downtime

---

## 16. Incident Management & Troubleshooting

---

### Q46. Walk me through how you would troubleshoot a pod that is stuck in CrashLoopBackOff.

**Answer:**

`CrashLoopBackOff` means a pod starts, crashes, Kubernetes restarts it, it crashes again — in a loop. Here is the step-by-step investigation:

**Step 1: Check what's happening**
```bash
kubectl get pods -n dev
# NAME                           READY   STATUS             RESTARTS
# user-service-7d4b9c8f6-xk9p2  0/1     CrashLoopBackOff   5
```

**Step 2: Look at the logs (what did the crash say?)**
```bash
kubectl logs user-service-7d4b9c8f6-xk9p2 -n dev
# If the container already died, check previous container logs:
kubectl logs user-service-7d4b9c8f6-xk9p2 -n dev --previous
```

**Step 3: Describe the pod (events section often has the root cause)**
```bash
kubectl describe pod user-service-7d4b9c8f6-xk9p2 -n dev
# Look at the Events section at the bottom
```

**Step 4: Common causes and fixes**

| Symptom in logs | Root cause | Fix |
|---|---|---|
| `Cannot connect to database` | Wrong DB connection string | Check secret/env var |
| `Error: Cannot find module` | Missing npm package | Check Dockerfile |
| `OOMKilled` | Container exceeded memory limit | Increase memory limit |
| `Permission denied` | Non-root user can't access file | Fix file permissions in Dockerfile |
| `Port already in use` | Two processes on same port | Fix app code |
| Secret/env var missing | Key Vault CSI not mounted | Check SecretProviderClass |

**Step 5: Fix and redeploy**
```bash
# Example: Update the deployment image after fixing the bug
kubectl set image deployment/user-service \
  user-service=acrazureshop.azurecr.io/user-service:build-43 \
  -n dev

# Watch rollout
kubectl rollout status deployment/user-service -n dev
```

---

### Q47. How do you do a rolling update and rollback in Kubernetes?

**Answer:**

**Rolling Update:**
Kubernetes updates pods one at a time (or in small batches) so the application never has zero running pods during an update.

```yaml
# Deployment rolling update strategy
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Can have 1 extra pod during update
      maxUnavailable: 0  # Zero pods can be down (zero-downtime)
```

**How it works:**
- Current: 3 pods running v1
- Start update to v2: Start 1 new v2 pod (now 4 pods: 3xv1 + 1xv2)
- v2 pod passes readiness probe → remove 1 v1 pod
- Repeat until all 3 pods are v2
- Zero downtime throughout

**Trigger an update:**
```bash
# Update image tag in the deployment
kubectl set image deployment/user-service \
  user-service=acrazureshop.azurecr.io/user-service:build-43 \
  -n prod

# Monitor the rollout
kubectl rollout status deployment/user-service -n prod
```

**Rollback:**
```bash
# Rollback to previous version immediately
kubectl rollout undo deployment/user-service -n prod

# Rollback to a specific version
kubectl rollout history deployment/user-service -n prod
kubectl rollout undo deployment/user-service --to-revision=3 -n prod
```

---

## 17. Leadership, Mentoring & Team Management

---

### Q48. How do you handle a situation where a junior engineer's PR has major issues?

**Answer:**

As a DevOps Lead, code reviews are as much about mentoring as they are about code quality. Here's my approach:

1. **Be specific, not vague** — Instead of "this is wrong", explain exactly what's wrong and why.
   - Bad: "This pipeline is not good."
   - Good: "This pipeline doesn't have a rollback step. If the prod deployment fails, we'll be stuck with a broken version. Here's how to add it: [example]."

2. **Explain the WHY** — Engineers who understand the reason learn faster and don't repeat the same mistake.

3. **Suggest, don't dictate** — "Have you considered using Helm templates here instead of duplicating YAML? It would make it easier to manage across environments."

4. **Acknowledge the good** — Start with what they did right before pointing out issues. This keeps morale up.

5. **Pair program if needed** — For complex issues, I'll sit with them (or video call) and work through it together rather than just leaving comments.

6. **Follow up** — After they fix the PR, review it again and acknowledge the improvement.

**AzureShop example:**
A junior engineer wrote a pipeline that stores the ACR password as a plain text environment variable in the YAML. Instead of just rejecting it, I'd:
- Approve the overall structure
- Flag the security issue clearly
- Explain Azure Key Vault and Variable Groups
- Point them to the existing pipeline that does it correctly
- Review the fix and merge it together

---

### Q49. How do you plan and estimate work for a DevOps team?

**Answer:**

**Work planning approach:**

1. **Break down epics into tasks** — A large goal like "Set up CI/CD for all 8 services" breaks into:
   - Write shared pipeline template (3 days)
   - Create CI pipeline for user-service (1 day)
   - Create CI pipelines for remaining 7 services (1 day each)
   - Create CD pipelines for dev/staging/prod (2 days)
   - Test end-to-end (1 day)

2. **Identify dependencies** — CD pipeline cannot be built before CI pipeline is done. Templates must be ready before individual service pipelines.

3. **Estimate with buffer** — Always add 20-30% buffer for unknowns (infrastructure issues, access problems, integration surprises).

4. **Use Azure Boards / Jira** — Each task is a ticket with story points, assignee, and acceptance criteria.

5. **Daily standups** — 15 minutes: What did I do? What will I do? Any blockers?

6. **Anticipate risks** — "We're relying on network access to ACR from AKS. If RBAC role assignment takes time to propagate, CI will fail. Mitigation: set up ACR access on day 1 before we need it."

---

## 18. Scenario-Based Questions

---

### Q50. Production is down. Users cannot place orders. What do you do?

**Answer:**

This is a high-pressure incident. Here's my structured response:

**Step 1: Don't panic. Start with data.**
```bash
# Check if pods are running
kubectl get pods -n prod

# Check recent events
kubectl get events -n prod --sort-by='.lastTimestamp'

# Check ingress / application gateway
kubectl get ingress -n prod
```

**Step 2: Check if it's a deployment issue**
```bash
# Was there a recent deployment?
kubectl rollout history deployment/order-service -n prod

# If yes — rollback immediately
kubectl rollout undo deployment/order-service -n prod
```

**Step 3: Check application logs**
```bash
kubectl logs deployment/order-service -n prod --tail=100
```

**Step 4: Check Azure Monitor alerts**
- Open Azure Monitor → Alerts → What fired in the last 30 minutes?
- Check Application Insights for error spikes
- Check SQL database DTU — is it maxed out?

**Step 5: Communicate**
- Notify stakeholders immediately: "We are aware of the order placement issue. Investigating. ETA for update: 15 minutes."
- Post in incident channel
- Never go silent during an incident

**Step 6: Fix and verify**
- Apply fix or rollback
- Verify `/health` endpoints return 200
- Place a test order
- Confirm with monitoring that error rate is back to normal

**Step 7: Post-mortem**
- After recovery, write a blameless post-mortem
- Root cause, timeline, impact, resolution, action items to prevent recurrence

---

### Q51. How do you onboard a new application team to your CI/CD platform?

**Answer:**

Onboarding a new team is a structured process:

**Step 1: Discovery meeting**
- What language/framework? (Node.js, Java, Python?)
- What are the test commands?
- What environments do they need? (dev, staging, prod)
- What secrets do they need? (DB credentials, API keys)
- What is their deployment strategy?

**Step 2: Prepare the platform**
- Create Variable Groups in Azure DevOps for their secrets
- Store secrets in Azure Key Vault
- Create Environments in Azure Pipelines (dev/staging/prod)
- Set up branch policies on their repo

**Step 3: Create pipeline from template**
```yaml
# New service pipeline — uses shared templates
trigger:
  branches:
    include: [dev, feature/*]

extends:
  template: templates/build-template.yaml@templates-repo
  parameters:
    serviceName: 'new-service'
    dockerfilePath: './services/new-service/Dockerfile'
    imageRepository: 'new-service'
```

**Step 4: First run together**
- Walk through the first pipeline run with the team
- Explain each stage and what it does
- Show them how to read logs when a stage fails

**Step 5: Documentation**
- Share a runbook: "How to deploy your service", "How to rollback"
- Add their service to the monitoring dashboard

**Step 6: Ongoing support**
- Be available for first 2 weeks to answer questions
- Set up a shared channel for DevOps support

---

### Q52. How would you reduce a 45-minute build time to under 10 minutes?

**Answer:**

A 45-minute build is painful. Here's how I would systematically reduce it:

**Step 1: Profile — find what's slow**
Look at each stage's duration in the pipeline logs. Identify the top 3 slowest steps.

**Step 2: Common fixes:**

**Parallelize independent stages:**
```yaml
# Instead of running these sequentially:
# Build user-service (10m) → Build product-service (10m) → Build cart-service (10m) = 30m

# Run them in parallel:
jobs:
- job: BuildUserService     # 10m
- job: BuildProductService  # 10m  ← runs at the same time
- job: BuildCartService     # 10m  ← runs at the same time
# Total: 10m instead of 30m
```

**Cache dependencies:**
```yaml
# Cache node_modules between builds
- task: Cache@2
  inputs:
    key: 'npm | "$(Agent.OS)" | package-lock.json'
    restoreKeys: 'npm | "$(Agent.OS)"'
    path: $(npm_config_cache)
```

**Use Docker layer caching:**
- Put frequently-changing layers (your app code) LAST in Dockerfile
- Put rarely-changing layers (npm install) FIRST — they get cached

**Only build changed services:**
```yaml
# Check which services changed in this commit
# Only trigger CI for services that have file changes
trigger:
  paths:
    include:
    - services/user-service/**   # Only triggers if user-service changed
```

**Result in AzureShop:**
- Parallelizing 8 service builds: 80 minutes → 10 minutes
- Dependency caching: saves 3-5 minutes per build
- Docker layer caching: saves 2-3 minutes per build

---

*Last Updated: 2026-07-10*
*Author: AzureShop DevOps Interview Prep*
*Based on: DevOps Lead JD — 12 Years Experience — Pune*
