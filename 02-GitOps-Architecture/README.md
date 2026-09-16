# Lesson 02 — GitOps Architecture

## 📚 Topics

- Git repository
- Kubernetes cluster
- GitOps controller
- Reconciliation
- Desired state
- Actual state
- Drift
- GitOps architecture
- Hands-on: Create a simple GitOps repository

---

# 1. GitOps Architecture

The basic GitOps architecture is:

```text
Developer
    ↓
Git Repository
    ↓
GitOps Tool
    ↓
Kubernetes
    ↓
Application
```

For our Azure implementation:

```text
Developer
    ↓
GitHub
    ↓
Argo CD
    ↓
Azure AKS
    ↓
Application
```

The main components are:

```text
1. Git Repository
2. GitOps Controller
3. Kubernetes Cluster
4. Application
```

---

# 2. Git Repository

The Git repository stores the **desired state** of the Kubernetes environment.

Example repository:

```text
gitops-aks-demo
│
└── manifests
    ├── namespace.yaml
    ├── deployment.yaml
    └── service.yaml
```

Example `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: nginx

spec:
  replicas: 3

  selector:
    matchLabels:
      app: nginx

  template:
    metadata:
      labels:
        app: nginx

    spec:
      containers:
        - name: nginx
          image: nginx:1.27
```

This configuration represents:

```text
Application = nginx
Replicas    = 3
Image       = nginx:1.27
```

GitHub is storing this configuration.

GitHub itself is not running the application.

```text
GitHub
   ↓
Stores Desired State
```

---

# 3. Kubernetes Cluster

Kubernetes is where the application actually runs.

For our project:

```text
Azure
  ↓
AKS
  ↓
Kubernetes Cluster
```

Inside AKS:

```text
AKS
 |
 └── Deployment
       |
       └── ReplicaSet
              |
              ├── Pod
              ├── Pod
              └── Pod
```

The Pods run the actual application container.

For example:

```text
Pod 1 → nginx:1.27
Pod 2 → nginx:1.27
Pod 3 → nginx:1.27
```

Therefore:

```text
GitHub
   ↓
Desired State

AKS
   ↓
Actual Environment
```

---

# 4. GitOps Controller

A GitOps controller watches the desired configuration and the Kubernetes environment.

For our project, the GitOps controller is:

```text
Argo CD
```

Argo CD connects Git and Kubernetes:

```text
GitHub
   |
   | Desired State
   ↓
Argo CD
   |
   | Reconciliation
   ↓
AKS
```

Argo CD continuously checks:

```text
What does Git say?

        VS

What is running in Kubernetes?
```

---

# 5. What Does Controller Mean?

A controller continuously observes the current state and takes action to move it toward the desired state.

For Argo CD:

```text
Git
 ↓
Desired State
 ↓
Argo CD
 ↓
Compare
 ↓
Kubernetes
 ↓
Actual State
```

If the states don't match, Argo CD can reconcile the difference depending on its synchronization configuration.

---

# 6. Kubernetes API

Argo CD communicates with Kubernetes through the **Kubernetes API**.

For example:

```text
kubectl
   |
   ↓
Kubernetes API
   |
   ↓
AKS
```

Argo CD also communicates with Kubernetes through the Kubernetes API:

```text
Argo CD
   |
   ↓
Kubernetes API
   |
   ↓
AKS
```

Conceptually:

```text
GitHub
   |
   ↓
Argo CD
   |
   ↓
Kubernetes API
   |
   ↓
AKS
   |
   ↓
Deployment
   |
   ↓
Pods
```

---

# 7. Desired State

**Desired state** means:

> What we want the Kubernetes environment to look like.

Example:

```text
Application = nginx
Replicas    = 3
Image       = nginx:1.27
Port        = 80
```

This desired state is represented by Kubernetes manifests stored in Git.

Example:

```yaml
spec:
  replicas: 3
```

Git therefore represents:

```text
Desired State
```

---

# 8. Actual State

**Actual state** means:

> What is currently running in Kubernetes.

For example:

```text
AKS

Application = nginx
Replicas    = 3
Image       = nginx:1.27
```

If Git and AKS match:

```text
Desired State
      =
Actual State
```

The application is in the expected state.

---

# 9. Reconciliation

**Reconciliation** means:

> Comparing the desired state with the actual state and taking action to make them match.

Example:

Git:

```text
replicas = 3
```

AKS:

```text
replicas = 1
```

Argo CD detects:

```text
3 ≠ 1
```

Then reconciliation can bring AKS toward the desired state.

```text
Git
replicas = 3
     |
     ↓
Argo CD
     |
     ↓
Kubernetes API
     |
     ↓
AKS
replicas = 1
     |
     ↓
Reconciliation
     |
     ↓
AKS
replicas = 3
```

Now:

```text
Desired State = Actual State
```

---

# 10. Drift

**Drift** occurs when the actual Kubernetes state differs from the desired state stored in Git.

Suppose Git contains:

```yaml
spec:
  replicas: 3
```

Git says:

```text
Desired = 3
```

Someone manually changes Kubernetes:

```bash
kubectl scale deployment nginx --replicas=1
```

Now:

```text
Git                  AKS

Desired               Actual

3 replicas             1 replica
     |                     |
     └─────────┬───────────┘
               ↓
              Drift
```

This is called:

```text
Configuration Drift
```

---

# 11. OutOfSync

Argo CD compares:

```text
Git
Desired = 3 replicas

       VS

AKS
Actual = 1 replica
```

Because:

```text
Desired ≠ Actual
```

Argo CD can report:

```text
OutOfSync
```

When the states match:

```text
Desired = Actual
```

Argo CD can report:

```text
Synced
```

Remember:

```text
Synced
   ↓
Desired State = Actual State
```

```text
OutOfSync
   ↓
Desired State ≠ Actual State
```

---

# 12. Complete Technical Example

Suppose our Git repository contains:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: nginx

spec:
  replicas: 2

  template:
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
```

The desired state is:

```text
Application = nginx
Replicas    = 2
Image       = nginx:1.27
```

Argo CD reads the repository.

AKS currently has:

```text
Application = nginx
Replicas    = 2
Image       = nginx:1.27
```

Therefore:

```text
Desired = Actual
```

Argo CD:

```text
Synced
```

---

# 13. Example — Changing the Desired State

Developer changes:

```yaml
replicas: 2
```

to:

```yaml
replicas: 3
```

Then:

```bash
git add .
git commit -m "Scale nginx to 3 replicas"
git push
```

Now:

```text
Git
Desired = 3

AKS
Actual = 2
```

Argo CD detects:

```text
Desired ≠ Actual
```

Application becomes:

```text
OutOfSync
```

After synchronization:

```text
Git
Desired = 3

AKS
Actual = 3
```

Now:

```text
Synced
```

---

# 14. Example — Manual Kubernetes Change

Git contains:

```text
replicas = 3
```

An engineer executes:

```bash
kubectl scale deployment nginx --replicas=1
```

Now:

```text
Git:
3 replicas

AKS:
1 replica
```

This creates:

```text
Drift
```

Argo CD detects:

```text
OutOfSync
```

If automatic synchronization/self-healing is configured appropriately, Argo CD can reconcile the cluster back toward:

```text
3 replicas
```

---

# 15. Why Use GitOps?

## Version Control

Git stores the history of configuration changes.

```text
Commit 1 → nginx v1.0
Commit 2 → nginx v1.1
Commit 3 → nginx v1.2
Commit 4 → nginx v2.0
```

We can determine:

- Who changed it
- What changed
- When it changed
- Previous configuration

---

## Pull Requests and Code Review

Instead of:

```text
Engineer
   ↓
kubectl
   ↓
Production
```

we can use:

```text
Engineer
   ↓
Git
   ↓
Pull Request
   ↓
Code Review
   ↓
Merge
   ↓
Argo CD
   ↓
Production
```

---

## Rollback

If a deployment causes a problem:

```text
Current
v2.0
  ↓
Git revert
  ↓
v1.2
  ↓
Argo CD
  ↓
AKS
```

Argo CD can reconcile Kubernetes to the reverted desired configuration.

---

## Drift Detection

GitOps allows us to identify when:

```text
Git State ≠ Kubernetes State
```

Argo CD can detect this difference.

---

# 16. Does GitHub Deploy the Application?

No.

GitHub stores the desired configuration.

For example:

```text
GitHub
   |
   └── deployment.yaml
```

Argo CD reads that configuration:

```text
GitHub
   |
   ↓
Argo CD
```

Argo CD communicates with Kubernetes:

```text
Argo CD
   |
   ↓
Kubernetes API
   |
   ↓
AKS
```

Kubernetes then creates/manages the workload:

```text
AKS
   |
   ↓
Deployment
   |
   ↓
ReplicaSet
   |
   ↓
Pods
   |
   ↓
Nginx
```

---

# 17. GitOps Controller vs Kubernetes Controllers

Kubernetes already has controllers.

For example:

```text
Deployment Controller
ReplicaSet Controller
Node Controller
Job Controller
```

These controllers manage Kubernetes resources.

Argo CD is a GitOps controller that manages the relationship between:

```text
Git
 ↓
Desired Kubernetes State
```

and:

```text
Kubernetes
 ↓
Actual State
```

Conceptually:

```text
GitHub
   |
   ↓
Argo CD
   |
   ↓
Kubernetes API
   |
   ↓
Kubernetes Controllers
   |
   ↓
Pods / Services / Deployments
```

---

# 18. GitOps Reconciliation Loop

Conceptually:

```text
             ┌─────────────────────┐
             │                     │
             ↓                     │
          Read Git                │
             │                     │
             ↓                     │
       Desired State              │
             │                     │
             ↓                     │
    Check Kubernetes              │
             │                     │
             ↓                     │
      Compare States              │
             │                     │
             ↓                     │
         Match?                   │
          /   \                   │
        YES    NO                 │
         │      │                 │
         │      ↓                 │
         │   Reconcile            │
         │      │                 │
         │      ↓                 │
         │  Update Kubernetes     │
         │      │                 │
         └──────┴─────────────────┘
```

This continuous comparison is the foundation of GitOps.

---

# 19. Hands-on — Create GitOps Repository

## Goal

Create our first GitOps repository.

For Lesson 02 we don't need:

```text
❌ AKS
❌ Argo CD
❌ Azure resources
```

We only need:

```text
Git
GitHub
Local computer
```

---

## Step 1 — Create GitHub Repository

Create:

```text
gitops-aks-demo
```

---

## Step 2 — Clone the Repository

```bash
git clone https://github.com/<YOUR-USERNAME>/gitops-aks-demo.git
```

Then:

```bash
cd gitops-aks-demo
```

Check:

```bash
git status
```

---

## Step 3 — Create Directory

Create:

```text
manifests
```

Repository:

```text
gitops-aks-demo
│
└── manifests
```

---

## Step 4 — Create namespace.yaml

Create:

```text
manifests/namespace.yaml
```

Content:

```yaml
apiVersion: v1
kind: Namespace

metadata:
  name: gitops-demo
```

This represents the desired namespace:

```text
Desired State:
Create namespace gitops-demo
```

---

## Step 5 — Create deployment.yaml

Create:

```text
manifests/deployment.yaml
```

Content:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: nginx
  namespace: gitops-demo

spec:
  replicas: 2

  selector:
    matchLabels:
      app: nginx

  template:
    metadata:
      labels:
        app: nginx

    spec:
      containers:
        - name: nginx
          image: nginx:1.27

          ports:
            - containerPort: 80
```

Desired state:

```text
Namespace = gitops-demo
Application = nginx
Replicas = 2
Image = nginx:1.27
Port = 80
```

---

## Step 6 — Create service.yaml

Create:

```text
manifests/service.yaml
```

Content:

```yaml
apiVersion: v1
kind: Service

metadata:
  name: nginx
  namespace: gitops-demo

spec:
  selector:
    app: nginx

  ports:
    - port: 80
      targetPort: 80

  type: ClusterIP
```

---

# 20. Final Repository Structure

Your repository should look like:

```text
gitops-aks-demo
│
└── manifests
    ├── namespace.yaml
    ├── deployment.yaml
    └── service.yaml
```

---

# 21. Commit and Push

Check:

```bash
git status
```

Stage the files:

```bash
git add .
```

Commit:

```bash
git commit -m "Add initial GitOps manifests"
```

Push:

```bash
git push origin main
```

Verify the files in GitHub.

---

# 22. What We Have Done

We have created the **desired state** in Git.

```text
GitHub
   |
   └── Desired State
        |
        ├── Namespace
        ├── Deployment
        └── Service
```

We have NOT deployed anything yet.

```text
❌ No AKS
❌ No Argo CD
❌ No Pods
❌ No running Nginx
```

Later, we will connect:

```text
GitHub
   |
   ↓
Argo CD
   |
   ↓
AKS
   |
   ↓
Deployment
   |
   ↓
Pods
   |
   ↓
Nginx
```

---

# 23. 🎯 Interview Questions

## Q1. What are the main components of a GitOps architecture?

**Answer:**

> The main components are a Git repository that stores the desired state, a GitOps controller such as Argo CD, a Kubernetes cluster where the application runs, and the application itself.

---

## Q2. What is reconciliation?

**Answer:**

> Reconciliation is the process of comparing the desired state stored in Git with the actual state in Kubernetes and taking corrective action when they differ.

---

## Q3. What is drift?

**Answer:**

> Drift occurs when the actual state of the Kubernetes environment differs from the desired state stored in Git.

---

## Q4. Does GitHub deploy the application?

**Answer:**

> No. GitHub stores the desired configuration. Argo CD reads that configuration and communicates with the Kubernetes API to reconcile the Kubernetes cluster.

---

## Q5. What is the role of Argo CD?

**Answer:**

> Argo CD is a GitOps continuous delivery controller for Kubernetes. It monitors the desired state in Git, compares it with the actual Kubernetes state, and reconciles differences.

---

## Q6. Why does Argo CD use the Kubernetes API?

**Answer:**

> The Kubernetes API is the interface through which clients and controllers communicate with Kubernetes. Argo CD uses it to inspect and manage Kubernetes resources in the target cluster.

---

## Q7. What happens if Git says 3 replicas but Kubernetes has 1?

**Answer:**

> There is a difference between the desired and actual state, which is configuration drift. Argo CD detects the difference and marks the application OutOfSync. Depending on the synchronization configuration, it can reconcile Kubernetes back toward the desired state.

---

# 24. 🧠 Final Mental Model

Remember this architecture:

```text
                     Developer
                         |
                         | git push
                         ↓
                    GitHub Repo
                         |
                         | Desired State
                         ↓
                      Argo CD
                 GitOps Controller
                         |
                         | Reconciliation
                         ↓
                  Kubernetes API
                         |
                         ↓
                        AKS
                         |
                         ↓
                    Deployment
                         |
                         ↓
                     ReplicaSet
                         |
                  +------+------+
                  |      |      |
                  ↓      ↓      ↓
                 Pod    Pod    Pod
                  |      |      |
                  +------+------+
                         |
                         ↓
                    Application
```

### ⭐ One sentence to remember

> **Git stores the desired state, Argo CD reconciles the desired state with Kubernetes, and Kubernetes runs the application.**

### 🔑 Six words from Lesson 02

```text
Git Repository
      ↓
Desired State
      ↓
Argo CD
      ↓
Reconciliation
      ↓
Actual State
      ↓
Drift
```
