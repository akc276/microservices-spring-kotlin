# ArgoCD GitOps Deployment Setup

This directory contains the ArgoCD declarative `Application` manifests for continuous GitOps deployment of `consumer-api-gateway` and `processor-service`.

---

## 🚀 Step-by-Step Setup Guide

### Step 1: Install ArgoCD into Kubernetes Cluster
Create the `argocd` namespace and apply the official ArgoCD manifests:

```bash
# Create dedicated namespace for ArgoCD controller
kubectl create namespace argocd

# Install ArgoCD controller and server components
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

---

### Step 2: Access ArgoCD Web Dashboard

1. **Port-forward the ArgoCD Server service to localhost:**
   ```bash
   kubectl port-forward svc/argocd-server -n argocd 8080:443
   ```
   Open browser at: `https://localhost:8080`

2. **Retrieve Initial Admin Password:**
   ```bash
   kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
   ```
   - **Username**: `admin`
   - **Password**: *(output from command above)*

---

### Step 3: Apply GitOps Application Manifests

Apply the declarative ArgoCD application definitions:

```bash
kubectl apply -f argocd/applications.yaml
```

This registers two applications in ArgoCD:
1. **`consumer-api-gateway`**: Points to `https://github.com/akc276/customer-api-gateway.git` (`path: helm/consumer-api-gateway`)
2. **`processor-service`**: Points to `https://github.com/akc276/processor-service.git` (`path: helm/processor-service`)

---

### Step 4: Verify Automated Sync & Self-Healing

ArgoCD automatically monitors both Git repositories:
- `syncPolicy.automated.prune: true` ensures deleted Helm resources in Git are pruned in cluster.
- `syncPolicy.automated.selfHeal: true` automatically corrects manual configuration drift.
- Any commit pushed to `main` in either microservice repository triggers an automatic Helm deployment into the cluster.
