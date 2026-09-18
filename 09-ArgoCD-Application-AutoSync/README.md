# Lesson 09 — Automatic Sync

## 1. What is Automatic Sync?

Automatic Sync means Argo CD does not wait for us to click the Sync button after detecting a Git change.

Flow:

```text
Developer
    |
    | git push
    v
GitHub
    |
    v
Argo CD detects change
    |
    v
Argo CD automatically synchronizes
    |
    v
AKS
```

---

# 2. Manual Sync vs Automatic Sync

## Manual Sync

```text
Git change
    |
    v
Argo CD detects change
    |
    v
OutOfSync
    |
    v
Human clicks Sync
    |
    v
AKS updated
```

## Automatic Sync

```text
Git change
    |
    v
Argo CD detects change
    |
    v
OutOfSync
    |
    v
Argo CD automatically syncs
    |
    v
AKS updated
```

The important difference:

```text
Manual Sync     = Human triggers deployment

Automatic Sync  = Argo CD triggers deployment
```

---

# 3. Enable Automatic Sync

For the existing Argo CD Application:

```text
gitops-nginx
```

enable:

```text
Automated Sync
```

For this lesson also enable:

```text
Prune
Self Heal
```

Conceptually:

```text
Automated Sync
      |
      +---- Git changes
      |       |
      |       v
      |   Automatically sync
      |
      +---- Prune
      |       |
      |       v
      |   Remove resources
      |   deleted from Git
      |
      +---- Self Heal
              |
              v
          Correct live-state
          drift
```

---

# 4. Automated Sync Experiment

Initially:

```yaml
replicas: 2
```

Change Git to:

```yaml
replicas: 3
```

Commit and push:

```bash
git add .
git commit -m "Scale nginx to 3 replicas"
git push
```

IMPORTANT:

```text
DO NOT click Sync.
```

Argo CD should detect the Git change and automatically synchronize.

Verify:

```bash
kubectl get deployment gitops-nginx -n gitops-demo
```

Expected:

```text
NAME           READY
gitops-nginx   3/3
```

Verify Pods:

```bash
kubectl get pods -n gitops-demo
```

There should be 3 Pods.

---

# 5. What Happened Behind the Scenes?

Git:

```text
replicas: 3
```

AKS initially:

```text
replicas: 2
```

Argo CD detects:

```text
Desired State != Actual State
```

Therefore:

```text
OutOfSync
```

Because Automatic Sync is enabled:

```text
OutOfSync
    |
    v
Automatic Sync
    |
    v
Kubernetes Deployment updated
    |
    v
3 Pods created
    |
    v
Synced
```

---

# 6. Self-Heal

Self-Heal handles drift caused by someone changing Kubernetes directly.

Suppose Git says:

```yaml
replicas: 3
```

But someone runs:

```bash
kubectl scale deployment gitops-nginx \
  --replicas=1 \
  -n gitops-demo
```

Now:

```text
Git Desired State = 3

AKS Actual State = 1
```

This is called:

```text
Drift
```

Argo CD detects the difference.

With Self-Heal enabled:

```text
Git
 |
 | replicas = 3
 v
Argo CD
 |
 | detects drift
 v
AKS
 |
 | replicas = 1
 v
Argo CD reconciles
 |
 v
AKS
 |
 | replicas = 3
```

Verify:

```bash
kubectl get deployment gitops-nginx -n gitops-demo -w
```

The Deployment should return to:

```text
3/3
```

---

# 7. Automatic Sync vs Self-Heal

These are related but different concepts.

### Automatic Sync

Primarily handles changes detected in Git.

Example:

```text
Git:
replicas 2 → 3

Argo CD:
automatically deploys 3
```

### Self-Heal

Handles drift in the live Kubernetes environment.

Example:

```text
Git:
replicas = 3

Someone:
kubectl scale → 1

Argo CD:
detects drift

Argo CD:
restores → 3
```

Mental model:

```text
              Git
               |
               | desired state
               v
           Argo CD
          /        \
         /          \
        v            v
 Git changes      Live drift
    |                 |
    v                 v
Auto Sync          Self Heal
```

---

# 8. Prune

Suppose Git contains:

```text
deployment.yaml
service.yaml
configmap.yaml
```

Later, we delete:

```text
configmap.yaml
```

from Git.

Without Prune:

```text
Git:
ConfigMap does not exist

AKS:
ConfigMap still exists
```

With:

```text
Prune = enabled
```

Argo CD can remove the resource that is no longer part of the desired Git state.

Conceptually:

```text
Git resource removed
        |
        v
Argo CD detects it
        |
        v
Prune
        |
        v
Resource removed from AKS
```

---

# 9. Three Important Auto-Sync Features

| Feature | Purpose |
|---|---|
| Automated Sync | Automatically deploy Git changes |
| Self Heal | Correct live Kubernetes drift |
| Prune | Remove resources deleted from Git |

Remember:

```text
Automated Sync = Deploy Git changes automatically

Self Heal = Fix Kubernetes drift

Prune = Remove resources no longer declared in Git
```

---

# 10. Important GitOps Principle

With GitOps:

```text
Git is the desired state.
```

If someone manually changes Kubernetes:

```bash
kubectl scale ...
kubectl edit ...
kubectl patch ...
```

they are changing the live state outside Git.

That can create:

```text
Desired State != Actual State
```

Argo CD detects and reconciles the difference.

Therefore:

```text
Git
 |
 | Source of Truth
 v
Argo CD
 |
 | Reconciliation
 v
Kubernetes
```

---

# 11. Useful Commands

Check Application:

```bash
kubectl get applications -n argocd
```

Check Application details:

```bash
kubectl describe application gitops-nginx -n argocd
```

Check Deployment:

```bash
kubectl get deployment gitops-nginx -n gitops-demo
```

Watch Deployment:

```bash
kubectl get deployment gitops-nginx -n gitops-demo -w
```

Check Pods:

```bash
kubectl get pods -n gitops-demo
```

Check Service:

```bash
kubectl get svc -n gitops-demo
```

Check all resources:

```bash
kubectl get all -n gitops-demo
```

---

# 12. Interview Discussion

### Q1. What is Automatic Sync in Argo CD?

Automatic Sync allows Argo CD to automatically synchronize the Kubernetes cluster when it detects a difference between the desired state stored in Git and the live state.

---

### Q2. What is Self-Heal?

Self-Heal allows Argo CD to automatically correct live-state drift.

For example:

```text
Git = 3 replicas
Kubernetes = 1 replica
```

Argo CD detects the drift and reconciles Kubernetes back toward the Git-defined state.

---

### Q3. What is Prune?

Prune removes resources from Kubernetes when those resources are no longer defined in the Git desired state and are managed by the Argo CD Application.

---

### Q4. What is the difference between Auto-Sync and Self-Heal?

```text
Auto-Sync:
Git change → automatically synchronize

Self-Heal:
Live Kubernetes drift → automatically reconcile
```

---

### Q5. Why is Git important in GitOps?

Git stores the declarative desired state of the application and infrastructure configuration.

It provides:

- Version history
- Auditability
- Review through pull requests
- Rollback capability
- A source of desired state for Argo CD

---

### Q6. What happens if someone manually changes a Deployment in AKS?

If the change creates drift from Git and Self-Heal is enabled, Argo CD detects the difference and reconciles the Deployment back toward the desired state defined in Git.

---

### Q7. What happens when a developer changes replicas from 2 to 3 in Git?

With Automatic Sync:

```text
Developer
   ↓
Git push
   ↓
Argo CD detects change
   ↓
Auto Sync
   ↓
Kubernetes Deployment updated
   ↓
3 Pods
```

No manual Sync button is required.

---

# 13. Real Production Scenario

Suppose a developer submits a pull request:

```yaml
replicas: 2
```

changes to:

```yaml
replicas: 5
```

After the PR is reviewed and merged:

```text
GitHub
   |
   | merge
   v
main branch
   |
   v
Argo CD
   |
   | detects desired-state change
   v
Automatic Sync
   |
   v
AKS
   |
   v
5 replicas
```

The deployment process becomes:

```text
Code
 ↓
Pull Request
 ↓
Review
 ↓
Merge
 ↓
Git desired state
 ↓
Argo CD
 ↓
AKS
```

This is the core GitOps deployment model.

---

# 14. Lesson 08 Final Architecture

```text
                 Developer
                     |
                     | git push
                     v
              ┌──────────────┐
              │    GitHub    │
              │              │
              │ manifests/   │
              │ deployment   │
              │ service      │
              └──────┬───────┘
                     |
                     | Desired State
                     v
              ┌──────────────┐
              │   Argo CD    │
              │              │
              │ Application  │
              │ Controller   │
              └──────┬───────┘
                     |
                     | Reconciliation
                     v
              ┌──────────────┐
              │     AKS      │
              │              │
              │ Deployment   │
              │      ↓       │
              │ ReplicaSet   │
              │      ↓       │
              │    Pods      │
              │      ↓       │
              │ Application  │
              └──────────────┘

Git = Desired State
AKS = Actual State
Argo CD = Reconciliation
```

---

# 15. Key Takeaways

```text
1. Argo CD connects Git with Kubernetes.

2. Git contains the desired state.

3. Kubernetes contains the actual state.

4. Argo CD compares desired and actual state.

5. OutOfSync means desired and actual state differ.

6. Missing means a desired resource is absent from Kubernetes.

7. Synced means desired and actual state are synchronized.

8. Healthy describes the health of the deployed resources.

9. Manual Sync requires a human to trigger synchronization.

10. Automatic Sync allows Argo CD to synchronize automatically.

11. Self-Heal corrects live-state drift.

12. Prune removes managed resources that are deleted from Git.

13. GitOps reduces direct manual changes to Kubernetes.

14. In GitOps, Git is the source of the desired state.
```

---

# 16. Lesson 08 Checklist

- [x] Created Argo CD Application
- [x] Connected Argo CD to GitHub
- [x] Configured repository and path
- [x] Configured AKS destination
- [x] Understood OutOfSync
- [x] Understood Missing
- [x] Performed Manual Sync
- [x] Verified Deployment
- [x] Verified Pods
- [x] Verified Service
- [x] Verified Synced/Healthy
- [x] Enabled Automatic Sync
- [x] Changed replicas from 2 → 3 through Git
- [x] Observed automatic synchronization
- [ ] Practice Self-Heal
- [ ] Practice Prune

---

# 🎯 Final Mental Model

```text
                  GIT
             "What I want"
                  |
                  | GitOps
                  v
               ARGO CD
        "Make reality match Git"
                  |
                  v
                 AKS
             "What exists"
                  |
                  v
               Pods/App


Git Change
    ↓
Auto Sync
    ↓
AKS Updated

Live Kubernetes Drift
    ↓
Self Heal
    ↓
AKS Corrected

Git Resource Deleted
    ↓
Prune
    ↓
AKS Resource Removed
```

**Core sentence to remember:**

> Git stores what I want. Argo CD makes Kubernetes match it. Kubernetes runs it.
