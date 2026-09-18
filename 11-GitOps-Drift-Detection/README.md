# Lesson 11 — GitOps Drift Detection

## 🎯 Objective

In this lesson, we learned:

- What configuration drift is
- Desired state vs actual state
- How Argo CD detects drift
- What `OutOfSync` means
- What `Synced` means
- Drift detection vs reconciliation
- How Self-Heal handles drift
- Different types of configuration drift
- Sync Status vs Health Status
- How to troubleshoot an OutOfSync application
- How GitOps helps prevent unmanaged configuration changes

> **Note:** The actual Self-Heal lab was already performed in the previous lessons, so this lesson focuses on deeper understanding and observation rather than repeating the same lab.

---

# 1. What is Configuration Drift?

Suppose Git contains:

```yaml
spec:
  replicas: 3
```

Git represents:

```text
Desired State = 3 replicas
```

But somebody directly changes Kubernetes:

```bash
kubectl scale deployment gitops-nginx \
  --replicas=1 \
  -n gitops-demo
```

Now:

```text
Git                  AKS
Desired State        Actual State

3 replicas           1 replica
     |                    |
     └────────┬───────────┘
              |
          DIFFERENCE
              |
              v
             DRIFT
```

This difference between the desired state and actual state is called:

> **Configuration Drift**

---

# 2. Desired State vs Actual State

This is the foundation of GitOps.

## Desired State

The configuration stored in Git.

Example:

```yaml
replicas: 3
image: nginx:1.27
```

Git says:

```text
I want:
3 replicas
nginx:1.27
```

---

## Actual State

What currently exists in Kubernetes.

For example:

```text
Deployment:
replicas = 1
image = nginx:1.27
```

Therefore:

```text
Desired State
      |
      | Git
      v
   Argo CD
      |
      | Compare
      v
Actual State
      |
      | Kubernetes API
      v
     AKS
```

---

# 3. Why Does Drift Happen?

Drift can happen when somebody changes Kubernetes directly.

Examples:

```bash
kubectl scale ...
```

```bash
kubectl edit deployment ...
```

```bash
kubectl patch deployment ...
```

For example:

```text
Git:
replicas = 3

Someone:
kubectl scale → 1

Result:
Git != Kubernetes
```

Another possibility is that another Kubernetes controller changes a resource.

The important GitOps principle is:

```text
Git
=
Desired Configuration
```

Direct changes to the cluster can therefore create drift.

---

# 4. Drift Is Not Only Replicas

Drift can involve many Kubernetes configuration fields.

## Example 1 — Replicas

Git:

```yaml
replicas: 3
```

AKS:

```text
replicas = 1
```

---

## Example 2 — Container Image

Git:

```yaml
image: nginx:1.27
```

AKS:

```text
image: nginx:1.26
```

---

## Example 3 — Environment Variable

Git:

```yaml
env:
  - name: APP_ENV
    value: production
```

AKS:

```text
APP_ENV=development
```

---

## Example 4 — Resource Limits

Git:

```yaml
resources:
  limits:
    cpu: "500m"
```

AKS:

```text
cpu limit = 1
```

---

## Example 5 — Service Configuration

Git:

```yaml
type: ClusterIP
```

AKS:

```text
type: LoadBalancer
```

All of these can represent differences between:

```text
Desired State
      ≠
Actual State
```

---

# 5. How Does Argo CD Detect Drift?

The Argo CD:

```text
Application Controller
```

works with:

```text
Git Desired State
```

and:

```text
Kubernetes Live State
```

Conceptually:

```text
                  Git
                   |
                   | Desired State
                   v
          ┌─────────────────┐
          │     Argo CD     │
          │                 │
          │ Application     │
          │ Controller      │
          └────────┬────────┘
                   |
                   | Kubernetes API
                   v
                  AKS
                   |
                   | Live State
                   v
               Resources
```

Argo CD compares the desired state with the live state.

If they differ:

```text
OutOfSync
```

---

# 6. What Does OutOfSync Mean?

Suppose:

```text
Git:
replicas = 3
```

and:

```text
AKS:
replicas = 1
```

Argo CD sees:

```text
Desired != Actual
```

Therefore:

```text
SYNC STATUS = OutOfSync
```

Think of it as:

```text
OutOfSync
=
Desired configuration and live configuration
do not match.
```

---

# 7. What Does Synced Mean?

Suppose:

```text
Git:
replicas = 3

AKS:
replicas = 3
```

Then:

```text
Desired == Actual
```

Argo CD reports:

```text
SYNC STATUS = Synced
```

Therefore:

```text
Synced
=
Desired state and live state are synchronized.
```

---

# 8. Drift Detection vs Correction

This is an important interview concept.

There are two separate ideas:

```text
Detection
```

and:

```text
Correction / Reconciliation
```

Example:

```text
Git = 3
AKS = 1
```

First:

```text
Argo CD detects:

"These configurations are different."
```

Then:

```text
Argo CD determines whether
the configured synchronization
behavior should reconcile the difference.
```

Therefore:

```text
Drift Detection
       ↓
OutOfSync
       ↓
Reconciliation
       ↓
Kubernetes moves toward
desired state
```

Do not say:

```text
OutOfSync = already fixed
```

Instead:

```text
OutOfSync
=
Argo CD detected a difference.
```

---

# 9. What Happens Without Self-Heal?

Suppose:

```text
Git = 3 replicas
AKS = 1 replica
```

and Self-Heal is disabled.

Argo CD can detect:

```text
OutOfSync
```

but live drift is not automatically corrected merely because it exists.

Conceptually:

```text
Git = 3
AKS = 1
     |
     v
Argo CD
     |
     v
OutOfSync
     |
     v
Wait for synchronization
```

The exact behavior depends on the application's synchronization configuration.

---

# 10. What Happens With Self-Heal?

Now:

```text
Git = 3
AKS = 1
```

and Self-Heal is enabled.

Argo CD can automatically reconcile the live state toward the desired state.

```text
Git
Desired = 3
    |
    v
Argo CD
    |
    | Detect drift
    v
AKS
Actual = 1
    |
    v
Self-Heal
    |
    v
AKS
Actual → 3
```

### Important definition

> **Self-Heal allows Argo CD to automatically reconcile live-state drift back toward the desired state stored in Git.**

---

# 11. Auto Sync vs Self-Heal

These concepts are closely related but different.

## Auto Sync

Think:

```text
Git changed
```

Example:

```text
Git:
replicas 2 → 3
```

Argo CD automatically synchronizes the new desired state.

---

## Self-Heal

Think:

```text
Kubernetes changed
```

Example:

```text
Git = 3

Someone:
kubectl scale → 1

Argo CD:
detects drift

Argo CD:
reconciles → 3
```

### Memory Trick

```text
Git Change
    ↓
Auto Sync

Live Drift
    ↓
Self-Heal
```

---

# 12. Sync Status vs Health Status

These are different concepts.

## Sync Status

Answers:

> Does the live configuration match the desired configuration?

Possible states include:

```text
Synced
OutOfSync
```

---

## Health Status

Answers:

> Are the deployed resources healthy?

Examples include:

```text
Healthy
Progressing
Degraded
Missing
Suspended
Unknown
```

Therefore:

```text
Synced ≠ Healthy
```

---

# 13. Example — Synced but Unhealthy

Suppose:

```text
Git:
replicas = 3
```

and Kubernetes also has:

```text
replicas = 3
```

Configuration matches.

Therefore:

```text
SYNC STATUS = Synced
```

But suppose all Pods are crashing.

The application may have:

```text
HEALTH STATUS = Degraded
```

So:

```text
Synced
=
Configuration matches

Healthy
=
Resources are healthy
```

These are separate concepts.

---

# 14. Example — OutOfSync but Application May Still Run

Suppose:

```text
Git:
replicas = 3

AKS:
replicas = 1
```

The one Pod is healthy.

Argo CD can still report:

```text
SYNC STATUS = OutOfSync
```

because:

```text
Desired configuration != Live configuration
```

`OutOfSync` does not automatically mean:

```text
Application is broken
```

It means:

```text
Configuration differs from desired state.
```

---

# 15. Real Production Scenario

Imagine a production application:

```text
production-api
```

Git contains:

```yaml
replicas: 10
```

An engineer directly runs:

```bash
kubectl scale deployment production-api \
  --replicas=5 \
  -n production
```

Now:

```text
Git:
10 replicas

AKS:
5 replicas
```

This is:

```text
Configuration Drift
```

Argo CD detects:

```text
OutOfSync
```

If Self-Heal is enabled:

```text
5 replicas
    ↓
Argo CD detects drift
    ↓
Reconciliation
    ↓
10 replicas
```

---

# 16. Why GitOps Cares About Drift

Imagine production is changed manually many times.

Git:

```text
replicas = 10
image = v1.5
CPU = 500m
```

Production:

```text
replicas = 7
image = v1.4
CPU = 1
```

Nobody knows why.

Now Git no longer accurately represents production.

This creates problems with:

- Auditing
- Troubleshooting
- Reproducibility
- Rollbacks
- Disaster recovery
- Environment consistency

GitOps aims to keep:

```text
Git
=
Desired Configuration
```

and:

```text
Argo CD
=
Reconciliation
```

---

# 17. GitOps and kubectl

GitOps does not mean:

```text
kubectl is forbidden
```

`kubectl` remains useful for:

```text
Troubleshooting
Inspection
Diagnostics
Emergency investigation
```

However, routine configuration changes should generally be made through Git so that Git remains the desired-state source.

Instead of:

```text
Developer
    |
    | kubectl edit
    v
AKS
```

the GitOps workflow is:

```text
Developer
    |
    | Pull Request
    v
GitHub
    |
    | Approved + Merged
    v
Argo CD
    |
    | Reconciliation
    v
AKS
```

---

# 18. Practice — Observation Only

The actual Self-Heal experiment was already performed in previous lessons.

Therefore, we do not need to repeat:

```bash
kubectl scale ...
```

Instead, inspect the current Application.

Run:

```bash
kubectl get application gitops-nginx -n argocd
```

Then:

```bash
kubectl describe application gitops-nginx -n argocd
```

Look for:

```text
Sync Status
Health Status
Conditions
Resources
```

Then check the Deployment:

```bash
kubectl get deployment gitops-nginx -n gitops-demo
```

And Pods:

```bash
kubectl get pods -n gitops-demo
```

Now answer these questions from the current state:

```text
1. What is the desired state?

2. What is the current AKS state?

3. Is there any drift?

4. What Sync Status should Argo CD show?

5. If drift occurs and Self-Heal is enabled,
   what should Argo CD do?
```

---

# 19. Practice — Scenario Table

Try answering these without looking at the answers.

| Git Desired State | AKS Actual State | What is happening? |
|---|---|---|
| replicas = 3 | replicas = 1 | ? |
| replicas = 3 | replicas = 3 | ? |
| Resource exists in Git | Resource missing in AKS | ? |
| Resource deleted from Git | Resource still exists in AKS | ? |
| Git = 3 | AKS = 1 + Self-Heal enabled | ? |

### Answers

| Scenario | Answer |
|---|---|
| Git = 3, AKS = 1 | Drift / OutOfSync |
| Git = 3, AKS = 3 | Synced |
| Resource exists in Git but missing in AKS | Resource is missing / OutOfSync |
| Resource deleted from Git but still in AKS | Potential Prune scenario |
| Git = 3, AKS = 1, Self-Heal enabled | Argo CD reconciles toward 3 |

---

# 20. Production Troubleshooting Flow

Suppose someone says:

> "Argo CD says my application is OutOfSync."

Do not immediately click Sync.

First:

```text
Application OutOfSync
        |
        v
Check Application
        |
        v
Identify affected resource
        |
        v
Compare Git configuration
with live Kubernetes configuration
        |
        v
Understand why drift happened
        |
        v
Determine the appropriate
reconciliation behavior
```

Useful commands:

```bash
kubectl get application gitops-nginx -n argocd
```

```bash
kubectl describe application gitops-nginx -n argocd
```

```bash
kubectl get deployment gitops-nginx \
  -n gitops-demo \
  -o yaml
```

---

# 21. Interview Questions

## Q1. What is configuration drift?

> Configuration drift occurs when the actual configuration of a Kubernetes resource differs from the desired configuration defined in Git.

---

## Q2. How does Argo CD detect drift?

> Argo CD compares the desired state obtained from the configured Git repository with the live state of resources in Kubernetes. When they differ, the Application can become OutOfSync.

---

## Q3. What does OutOfSync mean?

> OutOfSync means the desired state and live state do not match.

---

## Q4. Does OutOfSync mean the application is unhealthy?

> No. OutOfSync describes a configuration difference between desired and live state. Application health is represented separately by the health status.

---

## Q5. What is Self-Heal?

> Self-Heal allows Argo CD to automatically reconcile live-state drift back toward the desired state stored in Git.

---

## Q6. What happens if someone runs kubectl scale on a GitOps-managed Deployment?

> If the scale operation causes the live state to differ from Git, Argo CD detects the drift and marks the Application OutOfSync. If Self-Heal is enabled, Argo CD can automatically reconcile the Deployment back toward the Git-defined state.

---

## Q7. Is kubectl forbidden in GitOps?

> No. kubectl is useful for troubleshooting, inspection, and diagnostics. However, routine configuration changes should generally be made through Git so Git remains the desired-state source.

---

## Q8. What is the difference between Sync Status and Health Status?

```text
Sync Status
→ Does live configuration match Git?

Health Status
→ Are the deployed resources healthy?
```

---

# 22. Production Interview Scenario

### Interviewer:

> A developer says the Argo CD application is OutOfSync. What would you check?

### Strong Answer

```text
First, I would inspect the Argo CD Application status
and identify which resources are OutOfSync.

Then I would compare the desired configuration in Git
with the live resource in Kubernetes.

I would determine whether the difference was caused by
a Git change, a manual Kubernetes change, or another
controller.

If the application uses automatic synchronization and
Self-Heal, I would verify whether Argo CD is reconciling
the difference or whether there is a synchronization error.
```

The important troubleshooting approach is:

```text
Detect
  ↓
Identify affected resource
  ↓
Compare Git vs Kubernetes
  ↓
Understand the cause
  ↓
Reconcile appropriately
```

---

# 23. Final Mental Model

```text
                 Git
           Desired State
                 |
                 v
              Argo CD
                 |
                 | Compare
                 v
             Kubernetes
            Actual State
                 |
          ┌──────┴──────┐
          |             |
        Match        Different
          |             |
          v             v
       Synced        OutOfSync
                        |
                        v
                       Drift
                        |
                        v
                Self-Heal enabled?
                    /       \
                  Yes        No
                   |          |
                   v          v
             Reconcile     Wait for
             automatically synchronization
```

---

# 24. Complete GitOps Drift Flow

```text
                         Git
                    Desired State
                         |
                         v
                     Argo CD
                         |
                    Compare State
                         |
              ┌──────────┴──────────┐
              |                     |
           Matches               Different
              |                     |
              v                     v
           Synced               OutOfSync
                                    |
                                    v
                                  Drift
                                    |
                                    v
                              Self-Heal?
                              /        \
                            Yes         No
                             |           |
                             v           v
                       Reconcile     Remains
                       automatically OutOfSync
                             |
                             v
                           AKS
```

---

# 🧠 Key Memory

```text
Desired State
= What Git says should exist

Actual State
= What Kubernetes currently has

Drift
= Desired State != Actual State

OutOfSync
= Argo CD detected a difference

Synced
= Desired State == Live State

Self-Heal
= Automatically reconcile live drift

Auto Sync
= Automatically synchronize Git changes

Prune
= Remove managed resources no longer desired in Git
```

### One sentence to remember:

> **Drift occurs when Kubernetes no longer matches the desired state in Git; Argo CD detects the difference as OutOfSync, and with Self-Heal enabled it can automatically reconcile the live state toward Git.**

---

# ✅ Lesson 11 Checklist

- [x] Understand Desired State
- [x] Understand Actual State
- [x] Understand Configuration Drift
- [x] Understand why drift happens
- [x] Understand drift beyond replica changes
- [x] Understand how Argo CD detects drift
- [x] Understand OutOfSync
- [x] Understand Synced
- [x] Understand Drift Detection vs Reconciliation
- [x] Understand Self-Heal
- [x] Understand Auto Sync vs Self-Heal
- [x] Understand Sync Status vs Health Status
- [x] Understand GitOps and kubectl
- [x] Understand production troubleshooting flow
- [x] Self-Heal lab already completed in previous lessons
- [x] Perform observation-based practice
- [ ] Explain the production scenario confidently in an interview
