# Lesson 13 — Deploy Helm Application Using Argo CD

## 🎯 Goal

In this lesson, we connect **Helm + GitHub + Argo CD + AKS**.

By the end of this lesson, we will understand and practice:

- Creating a Helm Chart
- Helm Chart structure
- `Chart.yaml`
- `values.yaml`
- Helm templates
- `helm lint`
- `helm template`
- Pushing a Helm Chart to GitHub
- Creating an Argo CD Application for a Helm Chart
- Deploying the Helm application to AKS
- Argo CD Sync
- Argo CD Application status
- Synced vs OutOfSync
- Healthy vs Degraded
- Changing Helm values
- Application upgrade
- Chart version upgrade
- Why chart version change alone may not cause OutOfSync
- Helm Release vs Argo CD Application
- GitOps upgrade
- GitOps downgrade / rollback
- Drift detection
- Manual vs automated synchronization
- Troubleshooting Helm + Argo CD

---

# 1. Architecture

## Lesson 12 — Direct Helm

With direct Helm:

```text
Developer
    |
    | helm install / helm upgrade
    ↓
Kubernetes API
    ↓
AKS
    ↓
Deployment
    ↓
Pods
```

The developer directly tells Helm to install or upgrade the application.

---

# 2. Lesson 13 — Helm + Argo CD

Now Git becomes the source of truth:

```text
Developer
    |
    | git push
    ↓
GitHub
    |
    | Desired State
    ↓
Argo CD
    |
    | Helm rendering
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

The important difference is:

```text
Direct Helm:

helm install / helm upgrade
        ↓
Kubernetes


GitOps:

Git
 ↓
Argo CD
 ↓
Kubernetes
```

---

# 3. Important Helm + Argo CD Concept

Do not assume that an Argo CD Helm application is the same thing as a normal Helm CLI release.

With direct Helm:

```bash
helm install helm-nginx ./gitops-nginx
```

Helm creates and manages a Helm release.

With Argo CD:

```text
Git
 ↓
Argo CD Application
 ↓
Helm Chart
 ↓
Rendered Kubernetes manifests
 ↓
Kubernetes
```

The primary GitOps lifecycle object is:

```text
Argo CD Application
```

Argo CD uses Helm to render the chart into Kubernetes manifests and then manages the resulting Kubernetes resources.

---

# 4. Our Application

We continue using our Nginx application.

```text
Application:
gitops-nginx

Namespace:
gitops-demo

Image:
nginx:1.27
```

For this lesson, create a separate Argo CD Application:

```text
gitops-nginx-helm
```

This prevents us from interfering with the existing `gitops-nginx` application from earlier lessons.

---

# 5. Create the Helm Chart

On the Ubuntu VM:

```bash
mkdir -p ~/lesson13
cd ~/lesson13
```

Create the chart:

```bash
helm create gitops-nginx
```

Remove the default templates:

```bash
rm -rf gitops-nginx/templates/*
```

Check the structure:

```bash
find gitops-nginx -maxdepth 2 -type f
```

We will create:

```text
gitops-nginx/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    └── service.yaml
```

---

# 6. Chart.yaml

Edit:

```bash
vi gitops-nginx/Chart.yaml
```

Use:

```yaml
apiVersion: v2
name: gitops-nginx
description: Nginx Helm application deployed using Argo CD
type: application
version: 0.1.0
appVersion: "1.27"
```

Important:

```text
version
    ↓
Helm Chart version

appVersion
    ↓
Application version
```

For example:

```yaml
version: 0.1.0
appVersion: "1.27"
```

means:

```text
Chart version:
0.1.0

Application version:
1.27
```

These are different concepts.

---

# 7. values.yaml

Edit:

```bash
vi gitops-nginx/values.yaml
```

Use:

```yaml
replicaCount: 2

image:
  repository: nginx
  tag: "1.27"

service:
  type: ClusterIP
  port: 80
```

This represents our desired configuration:

```text
Replicas = 2
Image = nginx:1.27
Service = ClusterIP
Port = 80
```

---

# 8. Deployment Template

Create:

```bash
vi gitops-nginx/templates/deployment.yaml
```

Use:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
  namespace: {{ .Release.Namespace }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
        - name: nginx
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: 80
```

Important Helm expressions:

```text
.Release.Name
.Release.Namespace
.Values.replicaCount
.Values.image.repository
.Values.image.tag
```

---

# 9. Service Template

Create:

```bash
vi gitops-nginx/templates/service.yaml
```

Use:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}
  namespace: {{ .Release.Namespace }}
spec:
  selector:
    app: {{ .Release.Name }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: 80
  type: {{ .Values.service.type }}
```

---

# 10. Validate the Chart

Run:

```bash
helm lint ./gitops-nginx
```

This validates the Helm Chart.

Expected result should indicate that the chart is valid.

---

# 11. Render the Chart

Run:

```bash
helm template gitops-nginx ./gitops-nginx
```

Helm converts:

```text
values.yaml
+
templates/
        ↓
      Helm
        ↓
Kubernetes YAML
```

For example:

```yaml
replicas: {{ .Values.replicaCount }}
```

becomes:

```yaml
replicas: 2
```

And:

```yaml
image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

becomes:

```yaml
image: "nginx:1.27"
```

Important:

```bash
helm template
```

does not deploy anything.

It only renders the manifests.

---

# 12. Test Value Overrides

Run:

```bash
helm template gitops-nginx ./gitops-nginx --set replicaCount=3
```

The generated Deployment should contain:

```yaml
replicas: 3
```

Test an image override:

```bash
helm template gitops-nginx ./gitops-nginx --set image.tag=1.28
```

The generated Deployment should contain:

```yaml
image: "nginx:1.28"
```

---

# 13. Push the Helm Chart to GitHub

Our repository should look like:

```text
gitops-aks-demo/
│
├── manifests/
│   └── ...
│
└── helm/
    └── gitops-nginx/
        ├── Chart.yaml
        ├── values.yaml
        └── templates/
            ├── deployment.yaml
            └── service.yaml
```

Create the Helm directory:

```bash
mkdir -p ~/gitops-aks-demo/helm
```

Copy the chart:

```bash
cp -r ~/lesson13/gitops-nginx ~/gitops-aks-demo/helm/
```

Go to the repository:

```bash
cd ~/gitops-aks-demo
```

Check:

```bash
git status
```

Add:

```bash
git add helm/gitops-nginx
```

Commit:

```bash
git commit -m "Add Helm chart for nginx"
```

Push:

```bash
git push
```

Now GitHub contains the desired Helm configuration.

---

# 14. Git Is the Source of Truth

Our desired application now exists in Git:

```text
GitHub
  |
  └── helm/gitops-nginx/
          |
          ├── Chart.yaml
          ├── values.yaml
          └── templates/
```

Argo CD reads this configuration.

---

# 15. Create the Argo CD Application

Open the Argo CD UI.

Create:

```text
New Application
```

Use:

```text
Application Name:
gitops-nginx-helm

Project:
default
```

Repository:

```text
YOUR_GITHUB_REPOSITORY_URL
```

Path:

```text
helm/gitops-nginx
```

Destination:

```text
https://kubernetes.default.svc
```

Namespace:

```text
gitops-demo
```

Sync Policy:

```text
Manual
```

For this lesson, keep Manual Sync so that we can clearly observe the GitOps workflow.

---

# 16. How Argo CD Detects Helm

Argo CD examines:

```text
helm/gitops-nginx/
```

and finds:

```text
Chart.yaml
```

Therefore it recognizes the source as a Helm Chart.

Conceptually:

```text
Argo CD
   ↓
Git Repository
   ↓
helm/gitops-nginx
   ↓
Chart.yaml
   ↓
Helm Chart
```

---

# 17. Before the First Sync

After creating the Application, check:

```text
SYNC STATUS
HEALTH STATUS
```

The Application may show:

```text
OutOfSync
```

because:

```text
Git desired state
        ≠
AKS actual state
```

The resources have not been created yet.

---

# 18. First Sync

Click:

```text
SYNC
```

Then:

```text
SYNCHRONIZE
```

The workflow is:

```text
GitHub
   ↓
Helm Chart
   ↓
Argo CD
   ↓
Helm rendering
   ↓
Kubernetes manifests
   ↓
Kubernetes API
   ↓
AKS
```

---

# 19. Verify Kubernetes Resources

Run:

```bash
kubectl get all -n gitops-demo
```

Check Deployment:

```bash
kubectl get deployment -n gitops-demo
```

Check Pods:

```bash
kubectl get pods -n gitops-demo
```

Check Service:

```bash
kubectl get svc -n gitops-demo
```

Expected resources include:

```text
deployment.apps/gitops-nginx-helm

pod/gitops-nginx-helm-xxxxx

service/gitops-nginx-helm
```

---

# 20. Check Argo CD Application Status

Run:

```bash
kubectl get application gitops-nginx-helm -n argocd
```

Expected state:

```text
SYNC STATUS   HEALTH STATUS
Synced        Healthy
```

Remember:

```text
Synced
    =
Git desired state matches live Kubernetes state

Healthy
    =
Application resources are considered healthy
```

These are different concepts.

---

# 21. Inspect the Application

Run:

```bash
kubectl describe application gitops-nginx-helm -n argocd
```

Look for:

```text
Source
Destination
Sync Status
Health Status
Resources
Revision
```

This command is very useful when troubleshooting.

---

# 22. Helm Release vs Argo CD Application

With direct Helm:

```bash
helm install helm-nginx ./gitops-nginx
```

you have a normal Helm release.

You can use:

```bash
helm list
helm status
helm history
helm rollback
```

With Argo CD + Helm:

```text
Git
 ↓
Argo CD Application
 ↓
Helm Chart
 ↓
Rendered Kubernetes manifests
 ↓
Kubernetes
```

The primary GitOps lifecycle object is:

```text
Argo CD Application
```

Do not assume that:

```bash
helm list
```

must show an ordinary Helm CLI release for the Argo CD-managed application.

---

# 23. Change Helm Values

Current:

```yaml
replicaCount: 2
```

Change to:

```yaml
replicaCount: 3
```

Edit:

```bash
vi helm/gitops-nginx/values.yaml
```

Change:

```yaml
replicaCount: 2
```

to:

```yaml
replicaCount: 3
```

Commit:

```bash
git add helm/gitops-nginx/values.yaml
git commit -m "Scale nginx to 3 replicas"
```

Push:

```bash
git push
```

---

# 24. What Happens After the Git Change?

Argo CD detects:

```text
Git:
replicas = 3

AKS:
replicas = 2
```

Therefore:

```text
Desired State != Actual State
```

Argo CD reports:

```text
OutOfSync
```

Because our Application uses Manual Sync, Argo CD waits for us to synchronize.

---

# 25. Sync the Upgrade

Click:

```text
SYNC
```

Then:

```text
SYNCHRONIZE
```

The flow is:

```text
values.yaml
replicaCount: 3
        ↓
Helm rendering
        ↓
Deployment replicas: 3
        ↓
Argo CD
        ↓
Kubernetes
```

Verify:

```bash
kubectl get deployment gitops-nginx-helm -n gitops-demo
```

Expected:

```text
READY
3/3
```

---

# 26. Upgrade the Application Image

Current:

```yaml
image:
  repository: nginx
  tag: "1.27"
```

Change:

```yaml
image:
  repository: nginx
  tag: "1.28"
```

Commit:

```bash
git add helm/gitops-nginx/values.yaml
git commit -m "Update nginx image"
git push
```

Argo CD should detect:

```text
Desired:
nginx:1.28

Actual:
nginx:1.27
```

Therefore:

```text
OutOfSync
```

Sync the Application.

Verify:

```bash
kubectl describe deployment gitops-nginx-helm -n gitops-demo
```

Check the image.

---

# 27. What Does "Upgrade" Mean in GitOps?

With traditional Helm:

```bash
helm upgrade
```

With GitOps:

```text
Change values.yaml
        ↓
git commit
        ↓
git push
        ↓
Argo CD detects change
        ↓
OutOfSync
        ↓
Sync
        ↓
Kubernetes updated
```

Therefore, we normally do not manually run:

```bash
helm upgrade
```

for the Argo CD-managed application.

The Git change represents the desired upgrade.

---

# 28. Chart Version Upgrade

In `Chart.yaml`:

```yaml
version: 0.1.0
```

Change to:

```yaml
version: 0.2.0
```

For example:

```yaml
apiVersion: v2
name: gitops-nginx
description: Nginx Helm application deployed using Argo CD
type: application
version: 0.2.0
appVersion: "1.27"
```

Commit:

```bash
git add helm/gitops-nginx/Chart.yaml
git commit -m "Bump Helm chart version to 0.2.0"
git push
```

---

# 29. Why Chart Version Change May NOT Cause OutOfSync

This is an important concept.

If we change only:

```yaml
version: 0.1.0
```

to:

```yaml
version: 0.2.0
```

but the rendered Kubernetes manifests remain exactly the same:

```text
replicas: 3
image: nginx:1.27
service: ClusterIP
```

then Argo CD may remain:

```text
Synced
```

This is expected.

Why?

Because the chart version is metadata.

Argo CD's reconciliation is concerned with the resulting Kubernetes desired resources.

Conceptually:

```text
Chart version:
0.1.0
      ↓
Rendered Kubernetes resources
      ↓
Deployment replicas: 3
Image: nginx:1.27


Chart version:
0.2.0
      ↓
Rendered Kubernetes resources
      ↓
Deployment replicas: 3
Image: nginx:1.27
```

The rendered Kubernetes state is unchanged.

Therefore:

```text
Desired Kubernetes state
        =
Live Kubernetes state
```

So:

```text
Synced
```

is expected.

---

# 30. Prove the Chart Version Concept

Run:

```bash
helm template gitops-nginx ./helm/gitops-nginx
```

Changing only:

```yaml
version: 0.1.0
```

to:

```yaml
version: 0.2.0
```

does not change the Deployment or Service output.

Now change:

```yaml
replicaCount: 3
```

to:

```yaml
replicaCount: 4
```

Run:

```bash
helm template gitops-nginx ./helm/gitops-nginx
```

Now the generated Deployment changes:

```yaml
replicas: 4
```

That type of change can cause:

```text
OutOfSync
```

because the desired Kubernetes state has changed.

---

# 31. Chart Version vs Application Version

Remember:

```yaml
version: 0.2.0
```

means:

```text
Helm Chart version
```

while:

```yaml
appVersion: "1.27"
```

means:

```text
Application version metadata
```

And:

```yaml
image:
  tag: "1.27"
```

controls the actual container image.

These are three related but different concepts:

```text
Chart version
    ↓
0.2.0

Application version metadata
    ↓
1.27

Container image
    ↓
nginx:1.27
```

---

# 32. Argo CD Status

Important states:

## Synced

```text
Git desired state
        =
Kubernetes live state
```

## OutOfSync

```text
Git desired state
        !=
Kubernetes live state
```

## Healthy

Resources are functioning according to Argo CD's health assessment.

## Degraded

One or more resources are not healthy.

## Missing

A resource expected by the Application does not exist.

---

# 33. Argo CD Revision

Argo CD tracks the Git revision it has deployed.

For example:

```text
Commit A
    ↓
Application deployed
```

Then:

```text
Commit B
    ↓
Application upgraded
```

Then:

```text
Commit C
    ↓
Application upgraded again
```

Think:

```text
Git commit
    =
Version of desired state
```

---

# 34. GitOps Downgrade / Rollback

Suppose:

```text
Version 1
nginx:1.27

Version 2
nginx:1.28
```

Version 2 causes a problem.

With direct Helm, you may use:

```bash
helm rollback
```

With GitOps, the preferred model is:

```text
Git
 ↓
Revert the bad commit
 ↓
Push
 ↓
Argo CD detects desired state
 ↓
Sync
 ↓
Previous configuration restored
```

---

# 35. GitOps Rollback Example

Suppose Git currently contains:

```yaml
image:
  tag: "1.28"
```

Change it back:

```yaml
image:
  tag: "1.27"
```

Commit:

```bash
git add helm/gitops-nginx/values.yaml
git commit -m "Rollback nginx image to 1.27"
```

Push:

```bash
git push
```

Argo CD detects the desired state change.

Sync.

Verify:

```bash
kubectl describe deployment gitops-nginx-helm -n gitops-demo
```

The image should return to:

```text
nginx:1.27
```

---

# 36. Git Revert

Instead of manually editing the file, Git can also revert the bad commit:

```bash
git revert <commit>
```

Then:

```bash
git push
```

Argo CD detects the new desired state and reconciles it.

The important principle:

```text
GitOps rollback
    =
Change Git desired state back
```

---

# 37. Traditional Helm Rollback vs GitOps Rollback

### Traditional Helm

```text
helm upgrade
      ↓
Revision 2
      ↓
Problem
      ↓
helm rollback
      ↓
Revision 1
```

### GitOps

```text
Git commit A
      ↓
Git commit B
      ↓
Problem
      ↓
git revert B
      ↓
Argo CD
      ↓
Sync
      ↓
Previous desired state
```

Remember:

```text
Traditional Helm:
Helm release revision rollback

GitOps:
Git history + Argo CD reconciliation
```

---

# 38. Drift Detection

Suppose Git says:

```yaml
replicaCount: 3
```

but someone changes the live Kubernetes Deployment:

```bash
kubectl scale deployment gitops-nginx-helm \
  -n gitops-demo \
  --replicas=1
```

Now:

```text
Git:
3 replicas

AKS:
1 replica
```

This is:

```text
Configuration Drift
```

Argo CD can detect:

```text
OutOfSync
```

---

# 39. Reconcile the Drift

If Self-Heal is enabled, Argo CD can automatically correct the live state.

If Self-Heal is not enabled, manually Sync the Application.

The important model:

```text
Git
replicas: 3
       ↓
Argo CD
       ↓
AKS
replicas: 3
```

If someone changes AKS:

```text
AKS
replicas: 1
```

Argo CD detects:

```text
Desired = 3
Actual = 1
```

and reconciliation can restore the desired state.

---

# 40. Why Manual kubectl Changes Are Dangerous

Suppose production Git says:

```yaml
replicaCount: 5
```

Someone runs:

```bash
kubectl scale deployment myapp --replicas=2
```

Now:

```text
Desired = 5
Actual = 2
```

Argo CD sees drift.

The correct long-term solution is:

```text
If 2 is correct:
    Change Git to 2

If 5 is correct:
    Let Argo CD restore 5
```

Permanent configuration should be represented in Git.

---

# 41. Helm Release Lifecycle vs GitOps Lifecycle

## Direct Helm

```text
helm install
      ↓
Release
      ↓
helm status
      ↓
helm upgrade
      ↓
New revision
      ↓
helm history
      ↓
helm rollback
```

Commands:

```bash
helm list
helm status
helm history
helm upgrade
helm rollback
```

---

## Argo CD + Helm

```text
Git commit
      ↓
Argo CD Application
      ↓
Helm rendering
      ↓
Kubernetes resources
      ↓
Argo CD Sync
      ↓
New Git revision
```

Commands:

```bash
kubectl get application gitops-nginx-helm -n argocd

kubectl describe application gitops-nginx-helm -n argocd
```

The Git history and Argo CD Application are central to the GitOps lifecycle.

---

# 42. Troubleshooting Flow

If Argo CD shows:

```text
OutOfSync
```

First check:

```bash
kubectl get application gitops-nginx-helm -n argocd
```

Then:

```bash
kubectl describe application gitops-nginx-helm -n argocd
```

Check Git:

```text
Did the commit reach GitHub?
```

Check the path:

```text
helm/gitops-nginx
```

Validate Helm locally:

```bash
helm lint ./helm/gitops-nginx
```

Render:

```bash
helm template gitops-nginx ./helm/gitops-nginx
```

Check Kubernetes:

```bash
kubectl get all -n gitops-demo
```

---

# 43. Troubleshooting Helm Rendering

If Argo CD cannot render the chart, reproduce the problem locally:

```bash
helm lint ./gitops-nginx
```

Then:

```bash
helm template gitops-nginx ./gitops-nginx
```

Look for:

```text
YAML syntax problems
Template syntax problems
Missing values
Incorrect indentation
Invalid Kubernetes fields
```

This is a useful production troubleshooting technique.

---

# 44. Troubleshooting Health

If Argo CD says:

```text
Synced
```

but:

```text
Degraded
```

remember:

```text
Synced != Healthy
```

Check:

```bash
kubectl get pods -n gitops-demo
```

Then:

```bash
kubectl describe pod <pod-name> -n gitops-demo
```

Logs:

```bash
kubectl logs <pod-name> -n gitops-demo
```

Deployment:

```bash
kubectl describe deployment gitops-nginx-helm -n gitops-demo
```

---

# 45. Complete Upgrade Lifecycle

```text
Developer
    ↓
Change values.yaml
    ↓
git commit
    ↓
git push
    ↓
GitHub
    ↓
Argo CD detects change
    ↓
OutOfSync
    ↓
Sync
    ↓
Helm renders new configuration
    ↓
Kubernetes updated
    ↓
Deployment changes
    ↓
Pods updated
    ↓
Synced + Healthy
```

---

# 46. Complete Downgrade Lifecycle

```text
Version 1
nginx:1.27
    ↓
Git commit A
    ↓
Argo CD Sync
    ↓
Running

Version 2
nginx:1.28
    ↓
Git commit B
    ↓
Argo CD Sync
    ↓
Running

Problem
    ↓
git revert commit B
    ↓
Git returns to Version 1
    ↓
Argo CD detects change
    ↓
Sync
    ↓
nginx:1.27
```

---

# 47. Complete Lesson 13 Architecture

```text
                    Developer
                        |
                        | git push
                        ↓
                     GitHub
                        |
                        |
                Helm Chart Repository
                        |
              ┌─────────┴─────────┐
              |                   |
         Chart.yaml          values.yaml
              |                   |
              └─────────┬─────────┘
                        ↓
                     Argo CD
                        |
                   Application
                        |
                        ↓
                  Helm Rendering
                        |
                        ↓
                 Kubernetes API
                        |
                        ↓
                       AKS
                        |
                 ┌──────┴──────┐
                 ↓             ↓
             Deployment      Service
                 ↓
                Pods
```

---

# 48. Interview Questions

## Q1. How do you deploy a Helm application using Argo CD?

Answer:

> I store the Helm chart in Git, create an Argo CD Application pointing to the chart path, configure the destination Kubernetes cluster and namespace, and let Argo CD render the Helm chart and reconcile the resulting Kubernetes resources with the desired state in Git.

---

## Q2. How do you upgrade a Helm application using Argo CD?

Answer:

> I normally don't run `helm upgrade` manually. I change the Helm values or chart in Git, commit and push the change, and Argo CD detects the desired-state change. With manual sync I trigger synchronization, while with automated sync Argo CD reconciles it automatically.

---

## Q3. How do you downgrade a Helm application using GitOps?

Answer:

> I revert the Git commit that introduced the bad configuration or change the Helm values back to the previous desired version. Argo CD then detects the desired-state change and reconciles Kubernetes back to the previous configuration.

---

## Q4. What happens when values.yaml changes?

```text
values.yaml changes
       ↓
Git commit
       ↓
GitHub
       ↓
Argo CD detects change
       ↓
OutOfSync
       ↓
Sync
       ↓
Helm renders new manifests
       ↓
Kubernetes updated
```

---

## Q5. Does changing Chart.yaml version always cause OutOfSync?

No.

For example:

```yaml
version: 0.1.0
```

changed to:

```yaml
version: 0.2.0
```

may not cause OutOfSync if the rendered Kubernetes manifests are unchanged.

The important concept is:

```text
Chart metadata changed
        !=
Kubernetes desired state changed
```

---

## Q6. What is the difference between Helm rollback and GitOps rollback?

Traditional Helm:

```bash
helm rollback <release> <revision>
```

GitOps:

```text
git revert
    ↓
Git push
    ↓
Argo CD
    ↓
Sync
```

In GitOps, the desired state should be represented in Git.

---

## Q7. How do you check whether an Argo CD Helm application is deployed correctly?

```bash
kubectl get application gitops-nginx-helm -n argocd
```

Then:

```bash
kubectl describe application gitops-nginx-helm -n argocd
```

And:

```bash
kubectl get deployment -n gitops-demo
kubectl get pods -n gitops-demo
kubectl get svc -n gitops-demo
```

Verify:

```text
Sync Status = Synced
Health = Healthy
Pods = Running
Deployment = desired replicas available
```

---

# 49. Practice Checklist

Complete these tasks in order.

## Practice 1 — Create Chart

```bash
helm create gitops-nginx
```

Create:

```text
Chart.yaml
values.yaml
templates/deployment.yaml
templates/service.yaml
```

---

## Practice 2 — Validate

```bash
helm lint ./gitops-nginx
```

---

## Practice 3 — Render

```bash
helm template gitops-nginx ./gitops-nginx
```

---

## Practice 4 — Push to Git

Push the chart to:

```text
helm/gitops-nginx
```

---

## Practice 5 — Create Argo CD Application

Create:

```text
gitops-nginx-helm
```

Repository:

```text
YOUR_GITHUB_REPOSITORY_URL
```

Path:

```text
helm/gitops-nginx
```

Destination:

```text
https://kubernetes.default.svc
```

Namespace:

```text
gitops-demo
```

Sync:

```text
Manual
```

---

## Practice 6 — Deploy

Sync from Argo CD.

Verify:

```bash
kubectl get all -n gitops-demo
```

---

## Practice 7 — Status

Run:

```bash
kubectl get application gitops-nginx-helm -n argocd
```

And:

```bash
kubectl describe application gitops-nginx-helm -n argocd
```

Understand:

```text
Synced
Healthy
OutOfSync
Degraded
Missing
```

---

## Practice 8 — Upgrade Replicas

Change:

```yaml
replicaCount: 2
```

to:

```yaml
replicaCount: 3
```

Push.

Observe:

```text
OutOfSync
```

Sync.

Verify:

```bash
kubectl get deployment -n gitops-demo
```

---

## Practice 9 — Upgrade Image

Change:

```yaml
tag: "1.27"
```

to:

```yaml
tag: "1.28"
```

Push → observe OutOfSync → Sync → verify the Deployment.

---

## Practice 10 — Chart Version

Change:

```yaml
version: 0.1.0
```

to:

```yaml
version: 0.2.0
```

Push.

Observe that the Application may remain:

```text
Synced
```

if the rendered Kubernetes manifests have not changed.

---

## Practice 11 — Downgrade

Return:

```yaml
tag: "1.27"
```

Commit:

```text
Rollback nginx image to 1.27
```

Push → Sync → verify.

---

## Practice 12 — Drift

Change the live Deployment:

```bash
kubectl scale deployment gitops-nginx-helm \
  -n gitops-demo \
  --replicas=1
```

Git still says:

```text
replicas: 3
```

Observe:

```text
OutOfSync
```

Then reconcile the application.

---

# 50. Final Mental Model

The most important sentence from this lesson:

> **Helm packages and templates the Kubernetes application; Git stores the desired Helm configuration; Argo CD renders and reconciles that desired state with AKS.**

Remember the complete flow:

```text
GitHub
   ↓
Helm Chart
   ↓
Argo CD Application
   ↓
Helm Rendering
   ↓
Kubernetes Resources
   ↓
AKS
```

For an upgrade:

```text
Change values.yaml
      ↓
Git commit
      ↓
Git push
      ↓
Argo CD detects change
      ↓
OutOfSync
      ↓
Sync
      ↓
Kubernetes updated
```

For a downgrade:

```text
Bad Git change
      ↓
git revert / restore previous values
      ↓
Git push
      ↓
Argo CD detects change
      ↓
Sync
      ↓
Previous desired state restored
```

For drift:

```text
Git Desired State
       ≠
Kubernetes Actual State
       ↓
     Drift
       ↓
   OutOfSync
       ↓
   Reconciliation
       ↓
     Synced
```

## Key Terms

```text
Helm
    Kubernetes package manager and templating tool

Chart
    Reusable Helm package

Chart.yaml
    Helm Chart metadata

values.yaml
    Configuration values

templates/
    Parameterized Kubernetes manifests

Release
    Installed instance of a Helm Chart when using Helm directly

Argo CD Application
    GitOps object that defines what Argo CD should deploy and reconcile

Sync
    Apply desired Git state to Kubernetes

Synced
    Desired Kubernetes state matches live state

OutOfSync
    Desired Kubernetes state differs from live state

Healthy
    Resources are considered healthy

Drift
    Live Kubernetes state differs from desired Git state

Upgrade
    Move the application to a new desired configuration

GitOps Downgrade
    Restore an earlier desired configuration through Git and reconcile it with Argo CD
```
---
