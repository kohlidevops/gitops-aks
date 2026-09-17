# Lesson 06 — What is Argo CD?

> **GitOps Learning Path — Azure AKS + GitHub + Argo CD**

---

## 1. What is Argo CD?

**Argo CD is a GitOps Continuous Delivery tool for Kubernetes.**

Its main responsibility is:

> **Continuously compare what is defined in Git with what is actually running in Kubernetes, and reconcile the difference.**

The basic flow is:

```text
GitHub
   ↓
Desired State
   ↓
Argo CD
   ↓
Reconciliation
   ↓
Kubernetes API
   ↓
AKS
   ↓
Application
```

### Simple mental model

```text
GitHub = What I WANT
Argo CD = Makes it match
AKS     = What is ACTUALLY running
```

Remember:

> **Git stores what I want. Argo CD makes Kubernetes match it. Kubernetes runs it.**

---

# 2. Why do we need Argo CD?

In Lesson 05, we deployed an application manually.

The flow was:

```text
Developer
   ↓
Docker Build
   ↓
Azure Container Registry
   ↓
kubectl apply
   ↓
AKS
```

For example:

```bash
kubectl apply -f deployment.yaml
```

The problem is that Kubernetes configuration can be changed manually.

For example:

```bash
kubectl scale deployment nginx --replicas=1 -n gitops-demo
```

But Git may say:

```yaml
replicas: 3
```

Now there is a difference.

```text
Git:
replicas = 3

AKS:
replicas = 1
```

This difference is called **drift**.

Argo CD continuously detects this difference and can bring Kubernetes back to the state defined in Git.

---

# 3. Traditional Deployment vs GitOps

## Traditional Kubernetes Deployment

```text
Developer
    ↓
GitHub
    ↓
CI Pipeline
    ↓
kubectl apply
    ↓
AKS
```

The pipeline actively pushes changes into Kubernetes.

---

## GitOps Deployment

```text
Developer
    ↓
GitHub
    ↓
Argo CD
    ↓
Kubernetes API
    ↓
AKS
```

Argo CD watches Git and continuously reconciles Kubernetes.

---

# 4. Desired State vs Actual State

This is one of the most important concepts in GitOps.

## Desired State

Desired state means:

> **How we want Kubernetes to look.**

Example:

```yaml
spec:
  replicas: 3
```

Git says:

```text
Application should have 3 Pods.
```

---

## Actual State

Actual state means:

> **What is currently running in Kubernetes.**

For example:

```text
AKS currently has 2 Pods.
```

So:

```text
Desired State = 3 Pods
Actual State  = 2 Pods
```

Argo CD detects:

```text
Desired != Actual
```

Therefore:

```text
OutOfSync
```

Argo CD can then reconcile the cluster.

After reconciliation:

```text
Desired State = 3 Pods
Actual State  = 3 Pods
```

Therefore:

```text
Synced
```

---

# 5. What is Reconciliation?

**Reconciliation** means:

> Continuously checking the desired state against the actual state and correcting differences.

Example:

```text
Git
replicas: 3
   │
   │
   ▼
Argo CD
   │
   │ Compare
   ▼
AKS
replicas: 2
```

Argo CD detects:

```text
3 != 2
```

Then it reconciles:

```text
AKS → replicas: 3
```

Final state:

```text
Git = 3
AKS = 3
```

---

# 6. Argo CD Architecture

The important Argo CD components are:

```text
                 GitHub
                    │
                    ▼
            Repository Server
                    │
                    ▼
          Application Controller
                    │
                    ▼
             Kubernetes API
                    │
                    ▼
                   AKS
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
      Deployment          Service
          │
          ▼
       ReplicaSet
          │
          ▼
         Pods
```

Other important components:

```text
User
 │
 ▼
Argo CD API Server
 │
 ├── Web UI
 └── Argo CD CLI


Argo CD components
        │
        ▼
      Redis
```

---

# 7. Argo CD API Server

The **API Server** is the interface through which users and tools communicate with Argo CD.

It handles things such as:

- Authentication
- API requests
- Application information
- Application operations
- UI communication
- CLI communication

Example:

```text
Developer
    │
    ├── Argo CD UI
    │
    └── Argo CD CLI
             │
             ▼
       Argo CD API Server
```

Think:

> **API Server = Front door of Argo CD**

---

# 8. Repository Server

The **Repository Server** works with Git repositories.

Its job includes:

- Connecting to Git repositories
- Fetching repository content
- Getting Kubernetes manifests
- Rendering manifests when required

Later we will use:

```text
Plain YAML
Helm
Kustomize
```

For example:

```text
GitHub
  │
  │ git repository
  ▼
Repository Server
  │
  │ manifests
  ▼
Application Controller
```

Think:

> **Repository Server = Gets the desired state from Git**

---

# 9. Application Controller

This is one of the most important Argo CD components.

The **Application Controller** continuously monitors:

```text
Desired State
     vs
Actual State
```

It communicates with the Kubernetes API.

Example:

```text
Git
replicas: 3
    │
    ▼
Repository Server
    │
    ▼
Application Controller
    │
    │ Compare
    ▼
Kubernetes API
    │
    ▼
AKS
replicas: 2
```

The controller detects the difference.

```text
Desired = 3
Actual  = 2
```

Then reconciliation can change the cluster to:

```text
Actual = 3
```

Remember:

> **Application Controller = Brain of GitOps reconciliation**

---

# 10. Redis

Argo CD uses **Redis for caching/internal data**.

It is not where our application data is stored.

Do not think:

```text
Application → Redis
```

Instead:

```text
Argo CD
   │
   ▼
 Redis
```

Redis helps Argo CD internally with cached information and improves efficiency.

For our GitOps mental model, remember:

> **Redis = Argo CD internal cache**

---

# 11. Kubernetes API Server

Argo CD does not directly manipulate Pods.

It communicates with the Kubernetes API.

```text
Argo CD
   │
   ▼
Kubernetes API Server
   │
   ▼
Kubernetes Controllers
   │
   ▼
Deployment
   │
   ▼
ReplicaSet
   │
   ▼
Pods
```

This is important for interviews.

Argo CD tells Kubernetes what should exist.

Kubernetes then handles the normal Kubernetes control loop.

---

# 12. Argo CD Application

An **Argo CD Application** is an Argo CD resource that defines:

```text
Where is my Git repository?
        +
Which path contains my manifests?
        +
Which Kubernetes cluster should receive them?
        +
Which namespace should be used?
```

Conceptually:

```text
Argo CD Application
        │
        ├── Git Repository
        │
        ├── Path
        │
        ├── Destination Cluster
        │
        └── Namespace
```

Example:

```text
Git Repository:
https://github.com/example/gitops-aks-demo

Path:
manifests/

Destination:
AKS

Namespace:
gitops-demo
```

Important:

> **Argo CD Application is NOT the same thing as a Kubernetes Deployment or Pod.**

---

# 13. GitOps Example

Suppose Git contains:

```yaml
apiVersion: apps/v1
kind: Deployment

spec:
  replicas: 3
```

Git says:

```text
3 replicas
```

But AKS currently has:

```text
2 replicas
```

Argo CD sees:

```text
Desired State
replicas = 3

Actual State
replicas = 2
```

Therefore:

```text
OutOfSync
```

After reconciliation:

```text
Desired State
replicas = 3

Actual State
replicas = 3
```

Now:

```text
Synced
```

---

# 14. Complete Argo CD Architecture

Keep this architecture in your mind:

```text
                    Developer
                        │
                        ▼
                     GitHub
                        │
                 Desired State
                        │
                        ▼
              ┌─────────────────┐
              │ Repository      │
              │ Server          │
              └────────┬────────┘
                       │
                       ▼
              ┌─────────────────┐
              │ Application     │
              │ Controller      │
              └────────┬────────┘
                       │
                       ▼
              Kubernetes API
                       │
                       ▼
                      AKS
                       │
                       ▼
                  Deployment
                       │
                       ▼
                   ReplicaSet
                       │
                       ▼
                     Pods
                       │
                       ▼
                  Application
```

User interaction:

```text
Developer
   │
   ├──────────────► Argo CD UI
   │
   └──────────────► Argo CD CLI
                         │
                         ▼
                  Argo CD API Server
```

Internal cache:

```text
Argo CD
   │
   ▼
 Redis
```

---

# 15. CI vs GitOps CD

In our learning architecture:

```text
GitHub Actions
      │
      ▼
     CI
      │
      ├── Build
      ├── Test
      ├── Security Scan
      └── Push Image to ACR
```

Then:

```text
Git
 │
 ▼
Argo CD
 │
 ▼
AKS
```

So our mental model is:

```text
GitHub Actions = CI

Argo CD = GitOps CD + Reconciliation

AKS = Runtime
```

GitHub Actions can technically perform CD as well, but in our GitOps architecture Argo CD is responsible for Kubernetes deployment and reconciliation.

---

# 16. Lesson 05 vs Lesson 06

## Lesson 05

We manually deployed to AKS:

```text
Docker
   ↓
ACR
   ↓
kubectl
   ↓
AKS
```

## Lesson 06

We are introducing GitOps:

```text
GitHub
   ↓
Argo CD
   ↓
Kubernetes API
   ↓
AKS
```

The major change is:

```text
Before:

Developer/Pipeline → kubectl → AKS


After:

Developer → Git → Argo CD → AKS
```

---

# 17. Hands-On Practice

For Lesson 06, we are **not installing Argo CD yet**.

Argo CD installation will happen in:

```text
Lesson 07 — Install Argo CD on AKS
```

The purpose of this practice is to understand the difference between:

```text
Git = Desired State

AKS = Actual State
```

---

## Step 1 — Verify AKS

Run:

```bash
kubectl get nodes
```

Expected:

```text
NAME                                STATUS   ROLES
aks-nodepool1-xxxx-vmss000000       Ready    <none>
```

The important part is:

```text
STATUS = Ready
```

---

## Step 2 — Verify Kubernetes API

Run:

```bash
kubectl cluster-info
```

This confirms that `kubectl` can communicate with the Kubernetes cluster.

---

## Step 3 — Check namespaces

Run:

```bash
kubectl get namespaces
```

You should see:

```text
gitops-demo
```

because we created this namespace in the previous lessons.

---

## Step 4 — Check current resources

Run:

```bash
kubectl get all -n gitops-demo
```

We intentionally removed the old manual Nginx Deployment and Service from Lesson 05.

The namespace can remain empty.

This is useful for our Lesson 06 experiment.

---

# 18. Create the Git Desired State

Create a folder:

```text
lesson6-gitops/
└── manifests/
    └── deployment.yaml
```

Create:

```text
manifests/deployment.yaml
```

with:

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

---

# 19. Important — Do NOT Run kubectl apply

Do **not** run:

```bash
kubectl apply -f deployment.yaml
```

Why?

Because we want to create this situation:

```text
Git:
gitops-nginx Deployment
replicas = 2
```

But:

```text
AKS:
gitops-nginx does NOT exist
```

Therefore:

```text
Git = Desired State
AKS = Actual State
```

They are different.

This is exactly the type of difference Argo CD will detect.

---

# 20. Push the Manifest to GitHub

Put the file in our GitHub repository:

```text
gitops-aks-demo
```

Recommended structure:

```text
gitops-aks-demo/
│
├── manifests/
│   ├── namespace.yaml
│   ├── deployment.yaml
│   └── service.yaml
│
└── ...
```

Commit and push:

```bash
git add manifests/deployment.yaml
git commit -m "Add GitOps nginx deployment"
git push
```

Now GitHub contains the desired state.

---

# 21. Verify the Difference

Check Git:

```text
Deployment:
gitops-nginx
replicas: 2
```

Check AKS:

```bash
kubectl get deployment -n gitops-demo
```

The `gitops-nginx` Deployment should NOT exist yet.

This is intentional.

We have created:

```text
             GitHub
               │
               │
        Desired State
               │
               X
               │
        Actual State
               │
              AKS
```

Argo CD will eventually sit between these two.

---

# 22. What Will Happen in Lesson 07?

In Lesson 07 we will install Argo CD into the existing AKS cluster.

Then the architecture becomes real:

```text
GitHub
   │
   ▼
Argo CD
   │
   ▼
AKS
```

Argo CD will read the Deployment from Git.

It will notice:

```text
Git:
gitops-nginx exists

AKS:
gitops-nginx does not exist
```

Therefore:

```text
OutOfSync
```

After synchronization:

```text
Git:
gitops-nginx exists

AKS:
gitops-nginx exists
```

Then Kubernetes creates:

```text
Deployment
    ↓
ReplicaSet
    ↓
Pods
```

---

# 23. Troubleshooting Commands

## Check nodes

```bash
kubectl get nodes -o wide
```

## Check namespaces

```bash
kubectl get namespaces
```

## Check resources

```bash
kubectl get all -n gitops-demo
```

## Check deployments

```bash
kubectl get deployments -n gitops-demo
```

## Check Pods

```bash
kubectl get pods -n gitops-demo
```

## Check events

```bash
kubectl get events -n gitops-demo
```

---

# 24. Interview Questions

## Q1. What is Argo CD?

**Answer:**

Argo CD is a GitOps continuous delivery tool for Kubernetes. It continuously compares the desired state stored in Git with the actual state in Kubernetes and reconciles differences.

---

## Q2. What is GitOps?

**Answer:**

GitOps is an operational model where Git stores the desired state of infrastructure or applications, and an automated controller continuously reconciles the Kubernetes environment to match that state.

---

## Q3. What is reconciliation?

**Answer:**

Reconciliation is the continuous process of comparing the desired state with the actual Kubernetes state and correcting differences.

---

## Q4. What is drift?

**Answer:**

Drift occurs when the actual Kubernetes state differs from the desired state stored in Git.

Example:

```text
Git:
replicas = 3

AKS:
replicas = 2
```

This is drift.

---

## Q5. What does the Argo CD Application Controller do?

**Answer:**

The Application Controller continuously compares the desired state from Git with the actual state in Kubernetes and performs reconciliation when differences are detected.

---

## Q6. What does the Repository Server do?

**Answer:**

The Repository Server connects to Git repositories, retrieves application configuration and manifests, and prepares the desired state for Argo CD.

---

## Q7. What is the Argo CD API Server?

**Answer:**

The API Server provides the interface for the Argo CD UI, CLI, and API clients. It handles authentication and application operations.

---

## Q8. Why does Argo CD use Redis?

**Answer:**

Redis is used by Argo CD for caching and internal data. It is not the application's database.

---

## Q9. Does Argo CD directly create Pods?

**Answer:**

No. Argo CD communicates with the Kubernetes API. Kubernetes controllers then manage Deployments, ReplicaSets, and Pods.

---

## Q10. What is an Argo CD Application?

**Answer:**

An Argo CD Application defines the relationship between a Git repository/path and a destination Kubernetes cluster/namespace.

---

## Q11. What is the difference between Git and AKS in GitOps?

**Answer:**

Git represents the desired state, while AKS represents the actual runtime state.

---

## Q12. What happens if someone manually changes Kubernetes?

Example:

```text
Git:
replicas = 3
```

Someone runs:

```bash
kubectl scale deployment gitops-nginx --replicas=1 -n gitops-demo
```

Then:

```text
Desired = 3
Actual  = 1
```

Argo CD detects the drift.

Depending on the application's sync configuration, Argo CD can reconcile the cluster back to the Git-defined state.

---

# 25. Important Terms to Remember

| Term | Meaning |
|---|---|
| GitOps | Git-based operational model |
| Desired State | What we want Kubernetes to have |
| Actual State | What is currently running |
| Drift | Difference between desired and actual state |
| Reconciliation | Bringing actual state toward desired state |
| Argo CD | GitOps CD tool for Kubernetes |
| API Server | Argo CD interface |
| Repository Server | Retrieves/renders Git manifests |
| Application Controller | Compares and reconciles state |
| Redis | Internal cache |
| Argo CD Application | Defines Git source and Kubernetes destination |
| Synced | Desired and actual state match |
| OutOfSync | Desired and actual state differ |

---

# 26. Lesson 06 Checklist

## Concepts

- [ ] Understand what Argo CD is
- [ ] Understand GitOps
- [ ] Understand desired state
- [ ] Understand actual state
- [ ] Understand drift
- [ ] Understand reconciliation
- [ ] Understand Argo CD Application
- [ ] Understand Argo CD API Server
- [ ] Understand Repository Server
- [ ] Understand Application Controller
- [ ] Understand Redis
- [ ] Understand Kubernetes API interaction

## Practice

- [ ] Run `kubectl get nodes`
- [ ] Run `kubectl cluster-info`
- [ ] Run `kubectl get namespaces`
- [ ] Run `kubectl get all -n gitops-demo`
- [ ] Create `manifests/deployment.yaml`
- [ ] Push the Deployment manifest to GitHub
- [ ] Do NOT run `kubectl apply`
- [ ] Confirm the Deployment is in Git but not yet in AKS

---

# 27. Final Mental Model

The most important picture from this lesson:

```text
                 GitHub
                    │
                    │
              Desired State
                    │
                    ▼
                Argo CD
                    │
             Reconciliation
                    │
                    ▼
           Kubernetes API
                    │
                    ▼
                   AKS
                    │
                    ▼
               Deployment
                    │
                    ▼
                ReplicaSet
                    │
                    ▼
                  Pods
                    │
                    ▼
              Application
```

Remember this sentence:

> **Git stores what I want. Argo CD makes Kubernetes match it. Kubernetes runs it.**

---


Then we will see the real GitOps architecture running inside AKS.
