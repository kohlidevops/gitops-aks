# Lesson 10 — Manual Sync vs Automatic Sync

## 🎯 Objective

In this lesson, we learned the difference between:

- Manual Sync
- Automatic Sync
- Prune
- Self-Heal
- Sync Options
- Production GitOps deployment flow

We also practiced how Argo CD reacts when the desired Kubernetes configuration changes in Git.

---

# 1. What is Sync?

In GitOps, Git contains the desired state.

Example:

```yaml
spec:
  replicas: 3
```

But Kubernetes currently has:

```text
replicas = 2
```

Argo CD compares:

```text
Git Desired State
        3
        |
        | compare
        v
Kubernetes Actual State
        2
```

Because they are different:

```text
OutOfSync
```

**Sync means:**

```text
Make Kubernetes match the desired state stored in Git.
```

---

# 2. Manual Sync

With Manual Sync:

```text
Developer
    |
    | git push
    v
GitHub
    |
    v
Argo CD
    |
    | detects difference
    v
OutOfSync
    |
    | waits
    v
Human clicks SYNC
    |
    v
Kubernetes / AKS updated
```

Example:

Git:

```yaml
replicas: 3
```

AKS:

```text
2 replicas
```

Argo CD:

```text
OutOfSync
```

The application is not changed until a user triggers:

```text
SYNC
```

---

# 3. Automatic Sync

With Automatic Sync:

```text
Developer
    |
    | git push
    v
GitHub
    |
    v
Argo CD
    |
    | detects difference
    v
OutOfSync
    |
    v
Automatic Sync
    |
    v
Kubernetes / AKS updated
```

There is no need for a user to click Sync.

Example:

Git changes:

```yaml
replicas: 2
```

to:

```yaml
replicas: 3
```

After Argo CD detects the Git change:

```text
Git Desired State = 3
AKS Actual State  = 2
```

Argo CD automatically synchronizes:

```text
AKS Actual State = 3
```

---

# 4. Manual Sync vs Automatic Sync

The key question is:

```text
Who triggers the synchronization?
```

| | Manual Sync | Automatic Sync |
|---|---|---|
| Git change detected | Yes | Yes |
| Application becomes OutOfSync | Yes | Yes |
| Human clicks Sync | Required | Not required |
| Kubernetes updated | After manual Sync | Automatically |
| Human-controlled deployment step | Possible | Not required for sync |

### Easy way to remember

```text
Manual Sync
→ Human triggers deployment

Automatic Sync
→ Argo CD triggers deployment
```

---

# 5. Self-Heal

Self-Heal handles **live Kubernetes drift**.

Suppose Git says:

```yaml
replicas: 3
```

But someone manually changes Kubernetes:

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

This is:

```text
Drift
```

With Self-Heal enabled:

```text
Git
Desired = 3
    |
    v
Argo CD
    |
    | detects drift
    v
AKS
Actual = 1
    |
    v
Argo CD reconciles
    |
    v
AKS
Actual = 3
```

### Definition

```text
Self-Heal
=
Automatically correct live Kubernetes drift
back toward the desired state in Git.
```

---

# 6. Auto Sync vs Self-Heal

This is important for interviews.

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

Argo CD automatically synchronizes.

---

## Self-Heal

Think:

```text
Kubernetes changed
```

Example:

```text
Git = 3 replicas

Someone:
kubectl scale → 1

Argo CD:
detects drift

Argo CD:
restores desired state → 3
```

### Memory trick

```text
Git change  → Auto Sync

Live drift  → Self-Heal
```

---

# 7. Prune

Suppose Git contains:

```text
deployment.yaml
service.yaml
configmap.yaml
```

Argo CD manages all three.

Now a developer removes:

```text
configmap.yaml
```

from Git.

Git now says:

```text
Deployment
Service
```

But AKS still contains:

```text
Deployment
Service
ConfigMap
```

There is now a difference.

With Prune enabled:

```text
Resource removed from Git
        |
        v
Argo CD detects it is no longer desired
        |
        v
Prune
        |
        v
Resource removed from Kubernetes
```

### Definition

```text
Prune
=
Remove managed Kubernetes resources
that are no longer defined in Git.
```

---

# 8. Auto Sync + Self-Heal + Prune

These features work together:

```text
                    Git
                     |
              Desired State
                     |
                     v
                  Argo CD
                     |
        ┌────────────┼────────────┐
        |            |            |
        v            v            v
    Auto Sync     Self-Heal     Prune
        |            |            |
        v            v            v
   Git changes    Fix drift    Remove
                               deleted
                               resources
        \            |            /
         \           |           /
          └──────────┼───────────┘
                     v
                    AKS
```

Remember:

```text
Auto Sync
→ Deploy Git changes automatically

Self-Heal
→ Fix live Kubernetes drift

Prune
→ Remove resources deleted from Git
```

---

# 9. Sync Options

Sync Options control **how Argo CD performs synchronization**.

They are different from:

```text
Manual Sync
Automatic Sync
```

Think:

```text
Sync Policy
=
When should synchronization happen?

Sync Options
=
How should synchronization happen?
```

Some commonly encountered options include:

```text
CreateNamespace
PruneLast
Replace
Force
ServerSideApply
Validate
```

---

# 10. CreateNamespace

Suppose the Argo CD Application destination is:

```text
Namespace = gitops-demo
```

but the namespace doesn't exist.

A sync option can allow Argo CD to create it:

```text
CreateNamespace=true
```

Conceptually:

```text
Argo CD
    |
    | Namespace doesn't exist
    v
Create Namespace
    |
    v
Deploy application
```

Our Git repository also contains:

```text
namespace.yaml
```

so the namespace can be managed declaratively through Git.

---

# 11. PruneLast

With:

```text
PruneLast=true
```

pruning can happen as a final synchronization step.

Conceptually:

```text
Create/update required resources
            |
            v
Application resources become ready
            |
            v
Prune old resources
```

This can be useful when resource ordering matters.

---

# 12. Replace

Normally Argo CD updates resources using normal apply-style behavior.

With:

```text
Replace=true
```

Argo CD uses a replacement operation instead of the normal apply approach.

This is an advanced option.

Do not enable it on the `gitops-nginx` application just for practice.

Some replacement operations can be disruptive depending on the resource.

---

# 13. Force

Another advanced option is:

```text
Force=true
```

It can force replacement behavior when a normal synchronization cannot update a resource as required.

This should be used carefully because forced replacement can be disruptive.

Do not enable it on our simple Nginx application just for practice.

---

# 14. Server-Side Apply

Kubernetes supports:

```text
Server-Side Apply
```

Argo CD can use server-side apply when Kubernetes field ownership and merge behavior are important.

Conceptually:

```text
Client
   |
   v
Kubernetes API
   |
   v
Server manages field ownership
```

This becomes more relevant when multiple controllers or tools manage different fields of a resource.

For our simple Nginx application, it is not required.

---

# 15. Important Production Principle

Do not say:

```text
"Enable every Sync Option in production."
```

Instead:

```text
Sync options should be selected according
to the application's deployment requirements.
```

Options such as:

```text
CreateNamespace
PruneLast
Replace
Force
ServerSideApply
```

change synchronization behavior and should therefore be used deliberately.

---

# 16. Production Scenario

### Scenario

A developer changes a Kubernetes Deployment configuration in Git.

Example:

```diff
- replicas: 3
+ replicas: 5
```

---

## Step 1 — Developer Changes Git

```text
Developer
    |
    | Modify deployment.yaml
    v
GitHub Pull Request
```

---

# 17. Step 2 — Pull Request Review

A production GitOps workflow commonly uses:

```text
Developer
    |
    v
Pull Request
    |
    v
Code Review
    |
    v
Merge
```

The important point is that Argo CD normally watches the configured Git repository, revision, and path.

Once the change is merged into the branch/revision Argo CD monitors, Argo CD can detect the new desired state.

---

# 18. Step 3 — Argo CD Detects the Change

Git now contains:

```text
replicas = 5
```

AKS currently contains:

```text
replicas = 3
```

Argo CD compares:

```text
Desired State = 5
Actual State  = 3
```

Therefore:

```text
OutOfSync
```

---

# 19. Step 4 — Synchronization Depends on Policy

## Manual Sync

```text
OutOfSync
    |
    v
Wait for human
    |
    v
Click Sync
    |
    v
Kubernetes updated
```

## Automatic Sync

```text
OutOfSync
    |
    v
Argo CD automatically synchronizes
    |
    v
Kubernetes updated
```

---

# 20. Step 5 — Kubernetes Performs the Deployment

Argo CD does not act as the Kubernetes Deployment Controller.

Argo CD sends the desired resources through the Kubernetes API.

Then Kubernetes controllers perform the workload changes.

```text
Argo CD
    |
    v
Kubernetes API
    |
    v
Deployment Controller
    |
    v
ReplicaSet
    |
    v
Pods
```

For:

```text
3 replicas → 5 replicas
```

Kubernetes creates the additional Pods.

---

# 21. Complete Production Flow

```text
Developer
    |
    | Change YAML
    v
GitHub
    |
    | Pull Request
    v
Code Review
    |
    | Merge
    v
Git Branch monitored by Argo CD
    |
    v
Argo CD
    |
    | Detect Git change
    v
Compare Desired vs Actual
    |
    v
OutOfSync
    |
    ├───────────────┐
    |               |
 Manual          Automatic
 Sync              Sync
    |               |
 Human             Argo CD
    |               |
    └───────┬───────┘
            v
     Kubernetes API
            |
            v
        Deployment
            |
            v
        ReplicaSet
            |
            v
           Pods
```

---

# 22. Hands-On Practice

Use the existing application:

```text
Argo CD Application:
gitops-nginx

Namespace:
gitops-demo

AKS:
aks-gitops-lab
```

No new Azure resources are required.

---

## Practice 1 — Automatic Sync

Make sure Git contains:

```yaml
replicas: 3
```

Check:

```bash
kubectl get deployment gitops-nginx -n gitops-demo
```

Expected:

```text
NAME           READY   UP-TO-DATE   AVAILABLE
gitops-nginx   3/3     3            3
```

Change Git:

```yaml
replicas: 4
```

Commit and push:

```bash
git add .
git commit -m "Scale nginx to 4 replicas"
git push
```

Do not click Sync.

Watch:

```bash
kubectl get deployment gitops-nginx -n gitops-demo -w
```

Expected:

```text
3/3
 ↓
4/4
```

This demonstrates:

```text
Git Change
    ↓
Argo CD detects change
    ↓
Automatic Sync
    ↓
AKS updated
```

---

# 23. Practice 2 — Self-Heal

After Git contains:

```yaml
replicas: 4
```

deliberately change Kubernetes:

```bash
kubectl scale deployment gitops-nginx \
  --replicas=2 \
  -n gitops-demo
```

Watch:

```bash
kubectl get deployment gitops-nginx -n gitops-demo -w
```

Because Git says:

```text
4 replicas
```

and Kubernetes was manually changed to:

```text
2 replicas
```

Argo CD should detect the drift and reconcile the Deployment back toward:

```text
4 replicas
```

This demonstrates:

```text
Live Kubernetes Drift
        ↓
Argo CD detects drift
        ↓
Self-Heal
        ↓
Desired state restored
```

---

# 24. Practice 3 — Inspect Managed Resources

Run:

```bash
kubectl get application gitops-nginx -n argocd -o yaml
```

Look under:

```text
status:
  resources:
```

Argo CD should show resources managed by the Application.

For example:

```text
Namespace
Deployment
Service
```

This helps understand the relationship:

```text
Git resource
      ↓
Argo CD Application manages it
      ↓
Kubernetes resource
```

Do not delete a Git manifest yet for the Prune experiment.

---

# 25. Useful Commands

Check Application:

```bash
kubectl get applications -n argocd
```

Describe Application:

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

# 26. Interview Questions

## Q1. What is Manual Sync?

Manual Sync means Argo CD detects a difference between Git and Kubernetes but waits for a user to trigger synchronization.

---

## Q2. What is Automatic Sync?

Automatic Sync allows Argo CD to synchronize the Kubernetes cluster automatically when it detects a desired-state change in Git.

---

## Q3. What is Self-Heal?

Self-Heal allows Argo CD to automatically correct live Kubernetes drift so that the actual state moves back toward the desired state defined in Git.

---

## Q4. What is Prune?

Prune removes managed Kubernetes resources that are no longer defined in the Git desired state.

---

## Q5. What is the difference between Auto Sync and Self-Heal?

```text
Auto Sync
→ Git desired state changes.

Self-Heal
→ Live Kubernetes state changes outside Git.
```

---

## Q6. What are Sync Options?

Sync Options control how Argo CD performs synchronization.

Examples include:

```text
CreateNamespace
PruneLast
Replace
Force
ServerSideApply
Validate
```

---

## Q7. What happens when a developer changes a Deployment configuration in Git?

A strong interview answer:

```text
The developer modifies the Kubernetes manifest and raises
a pull request. After review and merge into the Git revision
monitored by Argo CD, Argo CD detects that the desired state
has changed.

It compares the desired state with the live state in Kubernetes
and marks the Application OutOfSync.

If Manual Sync is configured, an operator must trigger the sync.

If Automatic Sync is enabled, Argo CD automatically applies
the desired state through the Kubernetes API.

The Kubernetes controllers then reconcile the resources,
and the application moves toward the desired state.
```

---

# 27. Quick Interview Memory

Remember these:

```text
Manual Sync
= Human triggers synchronization

Auto Sync
= Argo CD automatically synchronizes Git changes

Self-Heal
= Argo CD fixes live-state drift

Prune
= Argo CD removes managed resources deleted from Git

Sync Options
= Control how synchronization is performed
```

---

# 28. Final Mental Model

```text
                         GIT
                   Desired State
                         |
                         |
                         v
                     ARGO CD
                         |
                Compare Desired
                 vs Actual State
                         |
                         v
                     OutOfSync
                         |
              ┌──────────┴──────────┐
              |                     |
        Manual Sync             Auto Sync
              |                     |
        Human clicks          Argo CD syncs
              |                     |
              └──────────┬──────────┘
                         |
                         v
                  Kubernetes API
                         |
                         v
                    Deployment
                         |
                         v
                     ReplicaSet
                         |
                         v
                        Pods
```

For drift:

```text
Git = 4 replicas
        |
        v
      Argo CD
        |
        | Self-Heal
        v
AKS = 2 replicas
        |
        v
Restored toward 4
```

For deleted resources:

```text
Resource exists in Git
        |
        v
Resource exists in AKS

Delete resource from Git
        |
        v
Argo CD detects difference
        |
        v
Prune enabled
        |
        v
Resource removed from AKS
```

---

# 🎯 Lesson 10 Key Takeaways

```text
1. Sync means making Kubernetes match Git.

2. Manual Sync requires a human to trigger synchronization.

3. Automatic Sync allows Argo CD to synchronize automatically.

4. Auto Sync primarily handles desired-state changes detected from Git.

5. Self-Heal handles drift in the live Kubernetes environment.

6. Prune removes managed resources that are no longer desired in Git.

7. Sync Options control how synchronization is performed.

8. CreateNamespace can allow Argo CD to create the destination namespace.

9. Replace and Force are advanced options and should be used carefully.

10. GitOps production flow commonly includes Git change,
    pull request, review, merge, Argo CD reconciliation,
    and Kubernetes deployment.

11. Argo CD communicates with the Kubernetes API;
    Kubernetes controllers create/update the workload resources.

12. The core GitOps model is:

    Git = Desired State
    Argo CD = Reconciliation
    Kubernetes = Actual Runtime State
```

---

# ✅ Lesson 10 Checklist

- [x] Understand Manual Sync
- [x] Understand Automatic Sync
- [x] Understand OutOfSync
- [x] Understand Self-Heal
- [x] Understand Prune
- [x] Understand Sync Options
- [x] Understand CreateNamespace
- [x] Understand PruneLast
- [x] Understand Replace
- [x] Understand Force
- [x] Understand Server-Side Apply
- [x] Practice Automatic Sync
- [x] Practice Self-Heal
- [ ] Practice Prune
- [ ] Practice production scenario explanation

---

# 🧠 One Sentence to Remember

> **Manual Sync waits for a human, Auto Sync reacts to Git changes automatically, Self-Heal corrects live drift, and Prune removes managed resources that are no longer defined in Git.**
