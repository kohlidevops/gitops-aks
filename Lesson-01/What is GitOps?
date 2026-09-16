````markdown
# Lesson 01 — What is GitOps?

## 📚 Topics

- What is GitOps?
- Traditional deployment vs GitOps
- Git as the source of truth
- Declarative configuration
- Desired state vs actual state
- GitOps workflow
- Why companies use GitOps
- GitOps vs CI/CD
- Where Argo CD fits

---

# 1. What is GitOps?

**GitOps** is a way of managing and deploying applications where **Git is the source of truth** for the desired state of the application and infrastructure.

A GitOps tool such as **Argo CD** continuously compares:

```text
Desired State → What we want
Actual State  → What is currently running
```

and makes Kubernetes match the desired state stored in Git.

### Simple mental model

```text
             Git
              │
              │ Desired State
              ↓
           Argo CD
              │
              │ Reconciliation
              ↓
        Kubernetes / AKS
              │
              ↓
         Application
```

### Easy way to remember

> Git says what I want.  
> Argo CD makes Kubernetes match it.  
> Kubernetes runs it.

---

# 2. Traditional Deployment vs GitOps

## Traditional Deployment

A developer or DevOps engineer may directly change Kubernetes:

```text
Developer
    ↓
kubectl
    ↓
Kubernetes
    ↓
Application
```

Example:

```bash
kubectl scale deployment myapp --replicas=3
```

The change happens directly in the Kubernetes cluster.

### Problem

Someone may later manually change the cluster:

```bash
kubectl scale deployment myapp --replicas=1
```

Now the Kubernetes environment may no longer match the documented configuration.

---

## GitOps Deployment

With GitOps:

```text
Developer
    ↓
Git
    ↓
Argo CD
    ↓
Kubernetes
    ↓
Application
```

The developer changes the desired configuration in Git.

Example:

```yaml
spec:
  replicas: 3
```

Argo CD detects the Git change and synchronizes Kubernetes.

---

# 3. Git as the Source of Truth

**Source of truth** means:

> The place we trust to define what the system should look like.

In GitOps:

```text
Git
 │
 └── Source of Truth
        │
        ↓
      Argo CD
        │
        ↓
      AKS
```

For example, Git may contain:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: myapp

spec:
  replicas: 3
```

Git says:

```text
I want 3 replicas.
```

Argo CD uses this desired configuration to keep Kubernetes aligned.

### Important

Git normally contains the **desired state**, not necessarily the current running state.

---

# 4. Declarative Configuration

There are two common ways to tell a system what to do.

## Imperative

Imperative means:

> Tell the system HOW to do something.

Example:

```bash
kubectl create deployment myapp --image=myapp:2.0

kubectl scale deployment myapp --replicas=3
```

We are giving Kubernetes individual instructions.

---

## Declarative

Declarative means:

> Tell the system WHAT the final state should be.

Example:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: myapp

spec:
  replicas: 3

  selector:
    matchLabels:
      app: myapp

  template:
    metadata:
      labels:
        app: myapp

    spec:
      containers:
        - name: myapp
          image: myapp:2.0
```

We are saying:

```text
I want:
- 3 replicas
- myapp version 2.0
```

Kubernetes determines how to achieve that state.

### GitOps uses declarative configuration.

---

# 5. Desired State vs Actual State

This is one of the most important GitOps concepts.

## Desired State

The state we **want**.

Usually defined in Git:

```text
Replicas = 3
Image = myapp:2.0
```

## Actual State

The state that is **currently running** in Kubernetes:

```text
Replicas = 3
Image = myapp:2.0
```

Everything is correct:

```text
Desired State
      =
Actual State
```

---

## What is Drift?

Suppose somebody manually changes Kubernetes:

```bash
kubectl scale deployment myapp --replicas=1
```

Now:

```text
Git                     AKS
Desired State            Actual State

3 replicas               1 replica
      │                      │
      └──────────┬───────────┘
                 ↓
               MISMATCH
                 ↓
                Drift
```

This difference is called **configuration drift**.

Argo CD can detect this mismatch.

If automatic synchronization/self-healing is enabled, Argo CD can reconcile the cluster back to the desired state.

---

# 6. GitOps Workflow

Suppose a developer wants to deploy application version `2.0`.

## Step 1 — Developer changes Git

For example:

```yaml
image: myapp:2.0
```

Then:

```bash
git add .
git commit -m "Deploy myapp v2.0"
git push
```

---

## Step 2 — Git contains the new desired state

```text
Git
 │
 └── myapp
      └── image: myapp:2.0
```

---

## Step 3 — Argo CD detects the change

Argo CD compares:

```text
Git = myapp:2.0
AKS = myapp:1.0
```

Therefore:

```text
OutOfSync
```

---

## Step 4 — Argo CD synchronizes

Argo CD applies the desired configuration to Kubernetes.

---

## Step 5 — Kubernetes performs the deployment

Kubernetes changes:

```text
myapp:1.0
     ↓
myapp:2.0
```

### Complete workflow

```text
Developer
    │
    │ git push
    ↓
Git Repository
    │
    │ Desired State
    ↓
Argo CD
    │
    │ Reconciliation
    ↓
Kubernetes / AKS
    │
    ↓
Application
```

---

# 7. Why Companies Use GitOps

## 7.1 Version Control

Every configuration change can be tracked in Git.

```text
Commit 1 → v1.0
Commit 2 → v1.1
Commit 3 → v1.2
Commit 4 → v2.0
```

We can see:

- Who changed it?
- What changed?
- When was it changed?
- What was the previous configuration?

---

## 7.2 Easy Rollback

Suppose version `2.0` has a problem.

Git contains the previous configuration.

We can revert the Git change:

```text
Git
 ↓
Revert
 ↓
Previous version
 ↓
Argo CD
 ↓
AKS
```

Argo CD then reconciles Kubernetes to the reverted desired state.

---

## 7.3 Auditability

Instead of making changes directly in production:

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

This provides a clear change history.

---

## 7.4 Reduced Manual Changes

GitOps reduces the need for engineers to manually modify production Kubernetes resources.

Instead of:

```text
Engineer → kubectl → Production
```

we prefer:

```text
Engineer → Git → Argo CD → Production
```

---

## 7.5 Drift Detection

If someone manually changes Kubernetes:

```text
Git
Desired = 3 replicas

        ≠

AKS
Actual = 1 replica
```

Argo CD can detect the mismatch.

---

# 8. GitOps vs CI/CD

This is an important concept.

## CI — Continuous Integration

CI generally handles:

- Build
- Unit tests
- Code quality checks
- Security scanning
- Packaging
- Container image creation

For example:

```text
Developer
    ↓
GitHub
    ↓
GitHub Actions
    ↓
Build
    ↓
Test
    ↓
Security Scan
    ↓
Docker Image
    ↓
Azure Container Registry
```

So for our learning:

```text
GitHub Actions = Mainly CI
```

---

# 9. CD — Continuous Delivery / Deployment

CD is concerned with getting the application into the target environment.

With GitOps:

```text
Git
 ↓
Argo CD
 ↓
AKS
 ↓
Application
```

So for our learning:

```text
Argo CD = GitOps-based CD
```

### Important clarification

GitHub Actions is not limited to CI.

GitHub Actions can also perform CD.

For example:

```text
GitHub Actions
     ↓
kubectl apply
     ↓
AKS
```

However, in a GitOps architecture, we normally separate responsibilities:

```text
GitHub Actions
      ↓
      CI
      ↓
Build / Test / Scan / Push Image
```

and:

```text
Argo CD
      ↓
     CD
      ↓
Deploy / Reconcile Kubernetes
```

---

# 10. GitHub Actions + Argo CD

Eventually we will build this architecture.

```text
                    CI
                     │
Developer             │
   │                  ↓
   └──────────────→ GitHub
                      │
                      ↓
                GitHub Actions
                      │
              ┌───────┼────────┐
              ↓       ↓        ↓
            Build    Test    Scan
                              │
                              ↓
                             ACR
                              │
                              │
                    Update GitOps Repo
                              │
                              ↓
                         Git Repository
                              │
                              ↓
                           Argo CD
                              │
                              ↓
                             AKS
                              │
                              ↓
                         Application
```

### Simple interpretation

```text
GitHub Actions
      ↓
Build and test the software

Git
      ↓
Store desired deployment configuration

Argo CD
      ↓
Deploy/reconcile desired state

AKS
      ↓
Run the application
```

---

# 11. Where Argo CD Fits

Argo CD is a **GitOps continuous delivery tool for Kubernetes**.

Think of it this way:

```text
GitOps
   ↓
Methodology / Operating Model

Argo CD
   ↓
Tool that implements GitOps

AKS
   ↓
Kubernetes environment
```

Argo CD continuously compares:

```text
Git
 ↓
Desired State

       VS

AKS
 ↓
Actual State
```

If they match:

```text
Synced
```

If they don't match:

```text
OutOfSync
```

---

# 12. Important GitOps Terms

| Term | Meaning |
|---|---|
| GitOps | Method of managing deployments using Git as the source of truth |
| Source of Truth | Location that defines the desired state |
| Desired State | What we want the environment to look like |
| Actual State | What is currently running |
| Drift | Difference between desired and actual state |
| Reconciliation | Process of making actual state match desired state |
| Declarative | Define what the final state should be |
| Argo CD | GitOps continuous delivery tool for Kubernetes |
| Sync | Applying desired state to Kubernetes |
| OutOfSync | Desired and actual states don't match |
| Synced | Desired and actual states match |

---

# 13. 🧠 Easy Way to Remember

Remember this:

```text
GIT
 ↓
WHAT I WANT

ARGO CD
 ↓
MAKE KUBERNETES MATCH

KUBERNETES / AKS
 ↓
RUN IT
```

And:

```text
Desired State
      │
      │ stored in
      ↓
     Git
      │
      │ reconciled by
      ↓
   Argo CD
      │
      ↓
Actual State
      │
      ↓
 Kubernetes / AKS
```

---

# 14. 🎯 Interview Questions

## Q1. What is GitOps?

### Answer

> GitOps is a deployment and operational model where Git acts as the source of truth for the desired state of an application or infrastructure. A GitOps controller such as Argo CD continuously compares the desired state in Git with the actual state in Kubernetes and reconciles differences.

---

## Q2. Why is Git called the source of truth?

### Answer

> Git contains the declarative configuration that defines the desired state of the application or environment. Therefore, the configuration in Git is treated as the authoritative definition of what should be running.

---

## Q3. What is desired state?

### Answer

> Desired state is the configuration we want the environment to have, such as the number of replicas, container image version, services and other Kubernetes resources. In GitOps, this desired state is stored in Git.

---

## Q4. What is drift?

### Answer

> Drift occurs when the actual state of the Kubernetes environment differs from the desired state stored in Git.

Example:

```text
Git = 3 replicas
AKS = 1 replica
```

This is configuration drift.

---

## Q5. What happens when someone manually changes Kubernetes?

### Answer

> A manual change can create drift because the actual Kubernetes state may no longer match the desired state in Git. Argo CD detects the difference and marks the application OutOfSync. If self-healing is enabled, Argo CD can reconcile the cluster back to the desired state.

---

## Q6. Is GitOps the same as CI/CD?

### Answer

> No. CI/CD and GitOps are related but not the same. CI focuses on building, testing and validating software. GitOps focuses on managing the desired deployment state through Git. Argo CD can provide the continuous delivery part of a GitOps workflow.

---

## Q7. What is the role of GitHub Actions and Argo CD?

### Answer

> GitHub Actions can handle CI activities such as building, testing, scanning and pushing container images. Argo CD handles GitOps-based continuous delivery by monitoring the desired state in Git and reconciling Kubernetes to that state.

---

# 15. 🧪 Hands-on

No Azure AKS is required for Lesson 01.

We are only building the mental model.

### Remember this architecture:

```text
Developer
    ↓
GitHub
    ↓
GitHub Actions
    ↓
Build / Test / Scan / Push Image
    ↓
Azure Container Registry
    ↓
GitOps Repository
    ↓
Argo CD
    ↓
Azure AKS
    ↓
Application
```

We will build this architecture step-by-step in later lessons.

---

# 16. ✅ Lesson 01 Checklist

Before moving to Lesson 02, make sure you can explain:

- [ ] What is GitOps?
- [ ] What is the source of truth?
- [ ] What is declarative configuration?
- [ ] What is desired state?
- [ ] What is actual state?
- [ ] What is drift?
- [ ] What is reconciliation?
- [ ] Why do companies use GitOps?
- [ ] GitOps vs CI/CD
- [ ] Role of GitHub Actions
- [ ] Role of Argo CD
- [ ] Where Argo CD fits in the architecture

---

# ⭐ Final Mental Model

```text
                 DEVELOPER
                     │
                     ↓
                  GITHUB
                     │
                     ↓
              GITHUB ACTIONS
                     │
                CI: Build
                Test
                Scan
                     │
                     ↓
                    ACR
                     │
                     ↓
              GITOPS REPOSITORY
                     │
               Desired State
                     │
                     ↓
                  ARGO CD
                     │
              Reconciliation
                     │
                     ↓
                    AKS
                     │
                Actual State
                     │
                     ↓
                APPLICATION
```

### One-line memory trick

> **GitHub Actions builds the software. Git stores what should be deployed. Argo CD makes Kubernetes match Git. AKS runs it.**
````
