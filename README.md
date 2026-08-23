# Consumer API Gateway & Processor Service Architecture

A production-ready distributed event-driven microservices architecture built with **Spring Boot 3+**, **Kotlin 2+**, **Apache Kafka / Azure EventHubs**, **Redis**, **MongoDB**, **Helm 3**, and **ArgoCD GitOps**.

Features **zero-code automatic distributed tracing** across HTTP and Kafka event streams via **Micrometer Tracing (W3C standard)**.

---

## 🏗️ Architecture Flow & End-to-End Distributed Tracing

```text
[ HTTP Client ] ──( GET /hello with Trace ID: <traceId> )──► [ consumer-api-gateway ]
                                                                     │
                                      Produces to `gateway-requests` │ (W3C traceparent header)
                                                                     ▼
                                                   [ Azure Event Hubs Simulator ]
                                                                     │
                                    Consumes from `gateway-requests` │ (W3C traceparent header)
                                                                     ▼
                                                          [ processor-service ]
                                                                     │
                                     Produces to `service-responses` │ (W3C traceparent header)
                                                                     ▼
                                                   [ Azure Event Hubs Simulator ]
                                                                     │
                                   Consumes from `service-responses` │ (W3C traceparent header)
                                                                     ▼
                                                         [ consumer-api-gateway ]
                                                         (Logs response with traceId)
```

### Verified Log Output Across Microservices (`Trace ID: 6a8a72131eb3e771a29055dcc2f745bf`)

```text
1. Gateway HTTP Entry & Producer:
   2026-08-23T04:07:47.416Z INFO 1 --- [consumer-api-gateway] [main] [6a8a72131eb3e771a29055dcc2f745bf-732c20585fd313da] KafkaProducer : Instantiated an idempotent producer.

2. Processor Service Consumer & Producer:
   2026-08-23T04:07:48.102Z INFO 1 --- [processor-service]   [container#0-0-C-1] [6a8a72131eb3e771a29055dcc2f745bf-810a42b109f5e08c] RequestEventListener : Received request event from consumer-api-gateway

3. Gateway Response Event Listener:
   2026-08-23T04:07:48.620Z INFO 1 --- [consumer-api-gateway] [container#0-0-C-1] [6a8a72131eb3e771a29055dcc2f745bf-732c20585fd313da] ResponseEventListener : Received response message from processor-service
```

---

## 📦 Microservice Repositories & Submodules

Clone the root repository recursively to automatically initialize all submodules:

```zsh
# Clone root orchestrator repository with all submodules
git clone --recursive https://github.com/akc276/microservices-spring-kotlin.git
cd microservices-spring-kotlin
```

### Repository Links:
| Microservice | Submodule Path | GitHub Repository Link | Responsibilities |
| :--- | :--- | :--- | :--- |
| **Consumer API Gateway** | `consumer-api-gateway` | [akc276/customer-api-gateway](https://github.com/akc276/customer-api-gateway) | REST API Gateway, HTTP Correlation Filter, Event Publisher, Response Listener, Scheduled Cron Poller |
| **Processor Service** | `processor-service` | [akc276/processor-service](https://github.com/akc276/processor-service) | Background Event Processor, Kafka Consumer, Response Publisher |

---

## Option 1: Fast Local Setup with Docker Compose (Recommended)

### Step 1: Launch all infrastructure & microservices
```zsh
# Run from root microservices-spring-kotlin directory
docker compose up -d --build
```

### Step 2: Verify container statuses
```zsh
docker compose ps
```

### Step 3: Trigger Hello World EventHub Workflow
```zsh
curl -i http://localhost:8080/hello
```

### Step 4: Continuous Live Log Streaming Across Microservices

#### 1. Stream continuous live logs from BOTH services simultaneously in real-time:
```zsh
docker compose logs -f gateway processor-service
```

#### 2. Stream live logs filtered by a specific Trace ID across both services:
```zsh
docker compose logs -f gateway processor-service | grep --line-buffered "6a8a72131eb3e771a29055dcc2f745bf"
```

#### 3. Inspect individual service logs:
```zsh
docker logs -f gateway-api
docker logs -f processor-service-api
```

---

## Option 2: Kubernetes Deployment with Helm & ArgoCD

### Step 1: Ensure local k3d Kubernetes cluster is active
```zsh
k3d cluster start gateway-cluster || k3d cluster create gateway-cluster
```

### Step 2: Install ArgoCD into Kubernetes
```zsh
kubectl create namespace argocd || true
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Step 3: Export and format k3d kubeconfig
```zsh
k3d kubeconfig get gateway-cluster > ./k3d-kubeconfig.yaml
sed -i '' 's/0.0.0.0/host.docker.internal/g; s/127.0.0.1/host.docker.internal/g' ./k3d-kubeconfig.yaml
sed -i '' 's/certificate-authority-data:.*/insecure-skip-tls-verify: true/g' ./k3d-kubeconfig.yaml
```

### Step 4: Build and Import local microservice Docker images into k3d
```zsh
# 1. Build Gateway image & import into k3d
docker build -t consumer-api-gateway:local ./consumer-api-gateway
k3d image import consumer-api-gateway:local -c gateway-cluster

# 2. Build Processor Service image & import into k3d
docker build -t processor-service:local ./processor-service
k3d image import processor-service:local -c gateway-cluster
```

### Step 5: Register Both Applications in ArgoCD GitOps
```zsh
# Deploys both consumer-api-gateway and processor-service Applications into ArgoCD
kubectl apply -f argocd/applications.yaml
```

### Step 6: Access ArgoCD Web Dashboard (at http://localhost:9091)
```zsh
# 1. Launch ArgoCD UI port forward in background
kubectl port-forward svc/argocd-server -n argocd 9091:80 &

# 2. Retrieve initial admin password (Username: admin)
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

### Step 7: Deploy Helm Charts Directly via Helm CLI (Alternative to GitOps Sync)
```zsh
# Install consumer-api-gateway Helm chart
helm upgrade --install gateway ./consumer-api-gateway/helm/consumer-api-gateway

# Install processor-service Helm chart
helm upgrade --install processor ./processor-service/helm/processor-service
```

### Step 8: Verify Kubernetes Deployments & Services
```zsh
kubectl get pods -A
kubectl get svc -A
```

---

## ⏱️ Scheduled Polling Service

The Gateway includes a periodic scheduled polling service ([ScheduledPublisherService.kt](consumer-api-gateway/src/main/kotlin/com/example/gateway/scheduler/ScheduledPublisherService.kt)):
- **Immediate Startup Trigger**: Uses `@EventListener(ApplicationReadyEvent::class)` to publish an initial heartbeat upon container startup.
- **Periodic Cron Schedule**: Uses `@Scheduled(cron = "0 */5 * * * *")` to execute every 5 minutes.
- **Trace Context**: Wrapped in `Observation.createNotStarted("scheduled.heartbeat", observationRegistry)` to ensure a root W3C Trace ID is generated from the first line of execution.

---

## 🔄 How to Update Microservices Code After Edits

### Fast Reload via Docker Compose:
```zsh
# Rebuild and restart consumer-api-gateway container
docker compose up -d --build --force-recreate gateway

# Rebuild and restart processor-service container
docker compose up -d --build --force-recreate processor-service
```

### Update via Kubernetes & ArgoCD:
```zsh
# Rebuild image & re-import into k3d
docker build -t consumer-api-gateway:local ./consumer-api-gateway
k3d image import consumer-api-gateway:local -c gateway-cluster
kubectl rollout restart deployment consumer-api-gateway

# Rebuild processor-service & re-import into k3d
docker build -t processor-service:local ./processor-service
k3d image import processor-service:local -c gateway-cluster
kubectl rollout restart deployment processor-service
```
