# Lesson 08 — First Argo CD Application + Automatic Sync

## 🎯 Objective

In this lesson, we learned how to:

- Create the first Argo CD Application
- Connect Argo CD to a GitHub repository
- Define Git as the desired state
- Deploy an application to AKS using Argo CD
- Understand `OutOfSync`, `Missing`, `Synced`, and `Healthy`
- Perform a manual synchronization
- Enable Automatic Sync
- Automatically deploy Git changes
- Understand Prune and Self-Heal
- Understand the difference between Manual Sync and Automatic Sync

---

# Lesson 08.1 — First Argo CD Application

## 1. GitOps Architecture

Our deployment flow:

```text
Developer
    |
    | git push
    v
GitHub Repository
    |
    | desired state
    v
Argo CD
    |
    | Kubernetes API
    v
AKS
    |
    v
Deployment
    |
    v
ReplicaSet
    |
    v
Pods
    |
    v
Application
```

The important idea:

```text
GitHub = Desired State

AKS = Actual State

Argo CD = Reconciliation Controller
```

Argo CD continuously compares the desired state in Git with the actual state in Kubernetes.

---

# 2. Example Git Repository

Repository:

```text
gitops-aks-demo
```

Directory:

```text
manifests/
├── namespace.yaml
├── deployment.yaml
└── service.yaml
```

---

# 3. Namespace

`namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: gitops-demo
```

The namespace is:

```text
gitops-demo
```

---

# 4. Deployment

`deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gitops-nginx
  namespace: gitops-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: gitops-nginx
  template:
    metadata:
      labels:
        app: gitops-nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
```

Desired state:

```text
Deployment name = gitops-nginx
Namespace        = gitops-demo
Replicas         = 2
Image            = nginx:1.27
Container port   = 80
```

---

# 5. Service

`service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: gitops-nginx
  namespace: gitops-demo
spec:
  selector:
    app: gitops-nginx
  ports:
    - port: 80
      targetPort: 80
  type: LoadBalancer
```

Because the Service is:

```yaml
type: LoadBalancer
```

AKS can provision/use an Azure Load Balancer frontend configuration for the Service.

---

# 6. Important GitOps Rule

For this exercise:

```text
DO NOT manually deploy the application with:

kubectl apply -f deployment.yaml
```

Instead:

```text
GitHub
   ↓
Argo CD
   ↓
AKS
```

Argo CD should be responsible for deploying the application.

---

# 7. Create Argo CD Application

In the Argo CD UI, create:

```text
Application Name:
gitops-nginx
```

Project:

```text
default
```

Repository:

```text
Your gitops-aks-demo GitHub repository
```

Path:

```text
manifests
```

Destination:

```text
https://kubernetes.default.svc
```

Namespace:

```text
gitops-demo
```

Initially, use:

```text
Sync Policy = Manual
```

---

# 8. Argo CD Application

The Argo CD Application is not the same thing as the Kubernetes application.

Argo CD Application means:

```text
"Deploy the Kubernetes manifests from this Git repository/path
to this Kubernetes cluster/namespace."
```

Example:

```text
Argo CD Application
        |
        +-- Repository
        |     GitHub
        |
        +-- Path
        |     manifests/
        |
        +-- Destination
        |     AKS
        |
        +-- Namespace
              gitops-demo
```

---

# 9. OutOfSync and Missing

If Git contains:

```text
Namespace
Deployment
Service
```

but AKS contains nothing:

```text
Git Desired State
       |
       | different
       v
AKS Actual State
```

Argo CD reports:

```text
SYNC STATUS: OutOfSync
HEALTH STATUS: Missing
```

### OutOfSync

Means:

```text
Desired State != Actual State
```

### Missing

Means the resource expected by Argo CD is not currently present in Kubernetes.

---

# 10. Manual Sync

Click:

```text
SYNC
```

Argo CD reads the manifests from Git and applies them to Kubernetes.

Flow:

```text
GitHub
   |
   | manifests
   v
Argo CD
   |
   | sync
   v
Kubernetes API
   |
   v
AKS
```

After successful synchronization:

```text
SYNC STATUS   = Synced
HEALTH STATUS = Healthy
```

---

# 11. Verify from Kubernetes

```bash
kubectl get all -n gitops-demo
```

Check Deployment:

```bash
kubectl get deployment gitops-nginx -n gitops-demo
```

Expected:

```text
NAME           READY   UP-TO-DATE   AVAILABLE
gitops-nginx   2/2     2            2
```

Check Pods:

```bash
kubectl get pods -n gitops-demo
```

Check Service:

```bash
kubectl get svc -n gitops-demo
```

Check Argo CD Application:

```bash
kubectl get applications -n argocd
```

Expected:

```text
NAME           SYNC STATUS   HEALTH STATUS
gitops-nginx   Synced        Healthy
```

---

# 12. Git Change with Manual Sync

Suppose Git contains:

```yaml
replicas: 2
```

Change it to:

```yaml
replicas: 3
```

Commit and push:

```bash
git add .
git commit -m "Scale nginx to 3 replicas"
git push
```

Argo CD detects that:

```text
Git Desired State = 3 replicas
AKS Actual State  = 2 replicas
```

Therefore:

```text
OutOfSync
```

With Manual Sync enabled, Argo CD waits for us.

We must click:

```text
SYNC
```

Then Kubernetes becomes:

```text
3 replicas
```

---

# Lesson 08.1 Mental Model

```text
Git
 |
 | "I want 3 replicas"
 |
 v
Argo CD
 |
 | "AKS currently has 2"
 |
 | OutOfSync
 |
 | Manual Sync
 |
 v
AKS
 |
 | "Now I have 3"
 v
Synced
```

---

