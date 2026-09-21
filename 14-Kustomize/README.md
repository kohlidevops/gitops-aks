# Lesson 14 — Kustomize

## 🎯 Goal

In this lesson, we learn **Kustomize** and understand how it is used in GitOps for managing multiple environments such as:

```text
DEV
QA
PROD
```

By the end of this lesson, we will understand and practice:

- What Kustomize is
- Base
- Overlay
- `kustomization.yaml`
- Environment-specific configuration
- DEV overlay
- QA overlay
- PROD overlay
- Replicas customization
- Labels customization
- Service customization
- Image customization
- Patches
- `kubectl kustomize`
- `kubectl apply -k`
- Kustomize vs Helm
- Kustomize + Argo CD architecture
- GitOps with Kustomize
- Troubleshooting Kustomize

---

# 1. Why Do We Need Kustomize?

Imagine we have one application:

```text
gitops-nginx
```

We need three environments:

```text
DEV
QA
PROD
```

The application is mostly the same, but some configuration is different.

For example:

| Configuration | DEV | QA | PROD |
|---|---:|---:|---:|
| Replicas | 1 | 2 | 5 |
| Image | nginx:1.27 | nginx:1.28 | nginx:1.27 |
| Service | ClusterIP | ClusterIP | LoadBalancer |
| Environment | dev | qa | prod |

Without Kustomize, we might create separate complete YAML files:

```text
dev/deployment.yaml
qa/deployment.yaml
prod/deployment.yaml
```

This creates duplication.

Kustomize provides:

```text
Common configuration
       ↓
      Base
       ↓
 ┌─────┼─────┐
 ↓     ↓     ↓
DEV   QA    PROD
 ↓     ↓     ↓
Overlay Overlay Overlay
```

The common configuration lives in the **Base**.

Environment-specific changes live in **Overlays**.

---

# 2. Kustomize Mental Model

Remember:

```text
BASE
  =
Common Kubernetes configuration

OVERLAY
  =
Environment-specific customization
```

For example:

```text
Base:
replicas: 2
image: nginx:1.27
service: ClusterIP
```

DEV:

```text
replicas: 1
environment: dev
```

QA:

```text
replicas: 2
environment: qa
```

PROD:

```text
replicas: 5
environment: prod
```

Kustomize combines:

```text
Base
 +
Overlay
 ↓
Final Kubernetes YAML
```

---

# 3. Kustomize vs Helm

We already learned Helm.

## Helm

```text
values.yaml
     +
templates/
     ↓
Helm
     ↓
Kubernetes YAML
```

## Kustomize

```text
Base YAML
     +
Overlay customization
     ↓
Kustomize
     ↓
Kubernetes YAML
```

The key difference:

```text
Helm
    =
Templates + Values

Kustomize
    =
Existing Kubernetes YAML + Customization
```

Kustomize generally does not require us to convert Kubernetes manifests into templates.

---

# 4. Kustomize Architecture

Our GitOps architecture:

```text
                    Git
                     │
              ┌──────┴──────┐
              ↓             ↓
             DEV           PROD
           overlay        overlay
              │             │
              └──────┬──────┘
                     ↓
                   Argo CD
                     ↓
                    AKS
```

We will also have QA:

```text
                    Git
                     │
        ┌────────────┼────────────┐
        ↓            ↓            ↓
       DEV          QA           PROD
     overlay      overlay       overlay
        │            │             │
        └────────────┼─────────────┘
                     ↓
                  Argo CD
                     ↓
                    AKS
```

---

# 5. What Is kustomization.yaml?

The most important Kustomize file is:

```text
kustomization.yaml
```

It tells Kustomize:

```text
Which resources should I use?

Which patches should I apply?

Which labels should I add?

Which images should I change?

Which replicas should I change?
```

Think:

```text
kustomization.yaml
        ↓
Instructions for Kustomize
```

It is different from:

```text
deployment.yaml
```

A Deployment defines a Kubernetes Deployment.

`kustomization.yaml` tells Kustomize how to assemble and customize Kubernetes resources.

---

# 6. Project Structure

We will create:

```text
lesson14-kustomize/
│
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
│
└── overlays/
    ├── dev/
    │   └── kustomization.yaml
    │
    ├── qa/
    │   └── kustomization.yaml
    │
    └── prod/
        ├── kustomization.yaml
        └── service-patch.yaml
```

---

# 7. Check Kustomize

On the Ubuntu VM:

```bash
kubectl version --client
```

Check Kustomize:

```bash
kubectl kustomize --help
```

You can also run:

```bash
kubectl kustomize version
```

The exact version output can vary depending on the kubectl release.

---

# 8. Create the Project

```bash
mkdir -p ~/lesson14-kustomize/base
mkdir -p ~/lesson14-kustomize/overlays/dev
mkdir -p ~/lesson14-kustomize/overlays/qa
mkdir -p ~/lesson14-kustomize/overlays/prod
```

Go to the directory:

```bash
cd ~/lesson14-kustomize
```

---

# 9. Create Base Deployment

Create:

```bash
vi base/deployment.yaml
```

Use:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gitops-nginx
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

This is the common application configuration.

---

# 10. Create Base Service

Create:

```bash
vi base/service.yaml
```

Use:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: gitops-nginx
spec:
  selector:
    app: gitops-nginx
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```

The Base contains the common Service configuration.

---

# 11. Create Base kustomization.yaml

Create:

```bash
vi base/kustomization.yaml
```

Use:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
```

This means:

```text
Base Kustomization
       ↓
deployment.yaml
service.yaml
```

---

# 12. Base Structure

Our Base is now:

```text
base/
├── deployment.yaml
├── service.yaml
└── kustomization.yaml
```

The Base contains:

```text
Application:
gitops-nginx

Replicas:
2

Image:
nginx:1.27

Service:
ClusterIP
```

This is our common configuration.

---

# 13. Build the Base

Run:

```bash
kubectl kustomize base
```

Kustomize generates Kubernetes YAML.

Conceptually:

```text
base/
   ↓
Kustomize
   ↓
Deployment + Service
```

Nothing is deployed yet.

---

# 14. Render Base to a File

Run:

```bash
kubectl kustomize base > base-rendered.yaml
```

Inspect:

```bash
cat base-rendered.yaml
```

This demonstrates:

```text
Source files
     ↓
Kustomize
     ↓
Final Kubernetes YAML
```

---

# 15. Create DEV Overlay

DEV should use:

```text
replicas = 1
environment = dev
```

Create:

```bash
vi overlays/dev/kustomization.yaml
```

Use:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

replicas:
  - name: gitops-nginx
    count: 1

labels:
  - pairs:
      environment: dev
```

Important:

```yaml
resources:
  - ../../base
```

means:

```text
DEV Overlay
     ↓
Uses Base
```

And:

```yaml
replicas:
  - name: gitops-nginx
    count: 1
```

customizes the Base Deployment.

---

# 16. Build DEV

Run:

```bash
kubectl kustomize overlays/dev
```

The generated Deployment should contain:

```yaml
replicas: 1
```

And resources should contain:

```yaml
environment: dev
```

The Base remains unchanged.

---

# 17. Create QA Overlay

QA should use:

```text
replicas = 2
environment = qa
```

Create:

```bash
vi overlays/qa/kustomization.yaml
```

Use:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

replicas:
  - name: gitops-nginx
    count: 2

labels:
  - pairs:
      environment: qa
```

Build:

```bash
kubectl kustomize overlays/qa
```

Expected:

```yaml
replicas: 2
```

and:

```yaml
environment: qa
```

---

# 18. Create PROD Overlay

PROD should use:

```text
replicas = 5
environment = prod
```

Create:

```bash
vi overlays/prod/kustomization.yaml
```

Use:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

replicas:
  - name: gitops-nginx
    count: 5

labels:
  - pairs:
      environment: prod
```

Build:

```bash
kubectl kustomize overlays/prod
```

Expected:

```yaml
replicas: 5
```

and:

```yaml
environment: prod
```

---

# 19. Base + Overlays

We now have:

```text
                  BASE
                   │
           ┌───────┼───────┐
           ↓       ↓       ↓
          DEV     QA      PROD
           │       │       │
          1       2       5
        replicas replicas replicas
```

The Base contains:

```text
Deployment
Service
Image
Ports
```

The overlays contain:

```text
Environment-specific differences
```

---

# 20. The Most Important Kustomize Concept

Remember:

> **Base contains what is common. Overlay contains what is different.**

For example:

```text
Base:
    nginx:1.27
    port: 80
    ClusterIP
    common labels

DEV:
    replicas: 1

QA:
    replicas: 2

PROD:
    replicas: 5
```

---

# 21. Add Environment Labels

We already added:

```text
environment=dev
environment=qa
environment=prod
```

This demonstrates that we can customize resources without changing the Base.

DEV:

```yaml
labels:
  - pairs:
      environment: dev
```

QA:

```yaml
labels:
  - pairs:
      environment: qa
```

PROD:

```yaml
labels:
  - pairs:
      environment: prod
```

---

# 22. Environment-Specific Service Type

Let's make PROD different.

Base:

```yaml
type: ClusterIP
```

DEV:

```text
ClusterIP
```

QA:

```text
ClusterIP
```

PROD:

```text
LoadBalancer
```

This demonstrates a real environment-specific customization.

---

# 23. Create PROD Service Patch

Create:

```bash
vi overlays/prod/service-patch.yaml
```

Use:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: gitops-nginx
spec:
  type: LoadBalancer
```

---

# 24. Update PROD kustomization.yaml

Edit:

```bash
vi overlays/prod/kustomization.yaml
```

Use:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

replicas:
  - name: gitops-nginx
    count: 5

labels:
  - pairs:
      environment: prod

patches:
  - path: service-patch.yaml
```

---

# 25. Build PROD

Run:

```bash
kubectl kustomize overlays/prod
```

The final Service should contain:

```yaml
type: LoadBalancer
```

The flow is:

```text
Base:
ClusterIP

        +

PROD Patch:
LoadBalancer

        ↓

Final PROD:
LoadBalancer
```

---

# 26. Important — Do Not Apply PROD Yet

The PROD overlay contains:

```yaml
type: LoadBalancer
```

When applied to AKS, Kubernetes may create Azure Load Balancer configuration.

For now, only render it:

```bash
kubectl kustomize overlays/prod
```

Do not manually apply the PROD overlay yet.

---

# 27. Kustomize Build vs Apply

Render only:

```bash
kubectl kustomize overlays/dev
```

This does not deploy anything.

To manually deploy:

```bash
kubectl apply -k overlays/dev
```

The `-k` means:

```text
Use Kustomize
```

Mental model:

```text
kubectl kustomize
    =
Render only

kubectl apply -k
    =
Render + Apply
```

For our GitOps workflow, Argo CD will perform the deployment.

---

# 28. Kustomize Image Customization

Kustomize can also customize container images.

Base:

```yaml
image: nginx:1.27
```

QA could use:

```text
nginx:1.28
```

without modifying the Base.

Update QA:

```bash
vi overlays/qa/kustomization.yaml
```

Use:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

replicas:
  - name: gitops-nginx
    count: 2

labels:
  - pairs:
      environment: qa

images:
  - name: nginx
    newTag: "1.28"
```

Build:

```bash
kubectl kustomize overlays/qa
```

Look for:

```yaml
image: nginx:1.28
```

---

# 29. Final Environment Configuration

We now have:

```text
                 BASE
                  │
        ┌─────────┼─────────┐
        ↓         ↓         ↓
       DEV       QA        PROD
        │         │         │
    replicas=1 replicas=2 replicas=5
    nginx:1.27 nginx:1.28 nginx:1.27
    ClusterIP  ClusterIP  LoadBalancer
        │         │         │
        ↓         ↓         ↓
     Argo CD   Argo CD   Argo CD
        │         │         │
        ↓         ↓         ↓
       AKS       AKS       AKS
```

One common application.

Different environment configurations.

Minimal duplication.

---

# 30. Push Kustomize to GitHub

Our repository should eventually look like:

```text
gitops-aks-demo/
│
├── helm/
│   └── gitops-nginx/
│
└── kustomize/
    └── gitops-nginx/
        │
        ├── base/
        │   ├── deployment.yaml
        │   ├── service.yaml
        │   └── kustomization.yaml
        │
        └── overlays/
            ├── dev/
            │   └── kustomization.yaml
            │
            ├── qa/
            │   └── kustomization.yaml
            │
            └── prod/
                ├── kustomization.yaml
                └── service-patch.yaml
```

Create the Git directory:

```bash
mkdir -p ~/gitops-aks-demo/kustomize
```

Copy the project:

```bash
cp -r ~/lesson14-kustomize \
  ~/gitops-aks-demo/kustomize/gitops-nginx
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
git add kustomize/gitops-nginx
```

Commit:

```bash
git commit -m "Add Kustomize base and environment overlays"
```

Push:

```bash
git push
```

---

# 31. Argo CD + Kustomize

Argo CD can work with Kustomize.

For example:

```text
Application:
gitops-nginx-dev

Path:
kustomize/gitops-nginx/overlays/dev
```

Another:

```text
Application:
gitops-nginx-qa

Path:
kustomize/gitops-nginx/overlays/qa
```

Another:

```text
Application:
gitops-nginx-prod

Path:
kustomize/gitops-nginx/overlays/prod
```

Argo CD processes the selected overlay.

---

# 32. Argo CD Architecture

```text
                         GitHub
                            │
              ┌─────────────┼─────────────┐
              ↓             ↓             ↓
             DEV           QA            PROD
           overlay       overlay        overlay
              │             │             │
              └─────────────┼─────────────┘
                            ↓
                         Argo CD
                            ↓
                     Kubernetes API
                            ↓
                           AKS
```

The important concept is:

```text
One Argo CD Application
        ↓
One selected environment overlay
```

---

# 33. Separate Argo CD Applications

A common model is:

```text
gitops-nginx-dev
    ↓
overlays/dev

gitops-nginx-qa
    ↓
overlays/qa

gitops-nginx-prod
    ↓
overlays/prod
```

Conceptually:

```text
Git
 │
 ├── DEV overlay ──→ Argo CD Application ──→ DEV
 │
 ├── QA overlay ───→ Argo CD Application ──→ QA
 │
 └── PROD overlay ─→ Argo CD Application ──→ PROD
```

---

# 34. Lab Environment vs Enterprise Environment

For this lab, we can represent:

```text
DEV
QA
PROD
```

using different namespaces in the same AKS cluster.

For example:

```text
gitops-dev
gitops-qa
gitops-prod
```

In an enterprise environment, they may instead be:

```text
DEV AKS
QA AKS
PROD AKS
```

The Kustomize Base + Overlay pattern remains the same.

---

# 35. Enterprise Architecture

A possible architecture:

```text
                         GitHub
                            │
          ┌─────────────────┼─────────────────┐
          ↓                 ↓                 ↓
      DEV overlay       QA overlay        PROD overlay
          │                 │                 │
          ↓                 ↓                 ↓
      Argo CD            Argo CD            Argo CD
          │                 │                 │
          ↓                 ↓                 ↓
       DEV AKS            QA AKS            PROD AKS
```

Another model is one Argo CD control plane managing multiple clusters:

```text
                    GitHub
                       │
                       ↓
                    Argo CD
                 ┌─────┼─────┐
                 ↓     ↓     ↓
               DEV    QA    PROD
               AKS    AKS    AKS
```

---

# 36. Kustomize Patches

A major Kustomize feature is patching.

Base:

```yaml
spec:
  replicas: 2
```

DEV:

```text
replicas: 1
```

PROD:

```text
replicas: 5
```

We don't need three complete Deployment YAML files.

Instead:

```text
Base Deployment
       +
Overlay customization
       ↓
Final Deployment
```

---

# 37. Kustomize Troubleshooting

If Argo CD reports an error, test the overlay locally.

DEV:

```bash
kubectl kustomize overlays/dev
```

QA:

```bash
kubectl kustomize overlays/qa
```

PROD:

```bash
kubectl kustomize overlays/prod
```

If the local build fails, fix Kustomize first.

---

# 38. Common Kustomize Problems

## Wrong relative path

Example:

```yaml
resources:
  - ../../base
```

Check:

```bash
ls ../../base
```

---

## Wrong resource name

Example:

```yaml
replicas:
  - name: gitops-nginx
```

must match:

```yaml
metadata:
  name: gitops-nginx
```

---

## Invalid patch

For:

```yaml
patches:
  - path: service-patch.yaml
```

make sure the patch identifies the correct Kubernetes resource.

---

## YAML indentation

Check with:

```bash
kubectl kustomize overlays/dev
```

YAML structure and indentation must be correct.

---

# 39. Troubleshooting Flow

When something fails:

```text
Argo CD
   ↓
Check Application
   ↓
Check error
   ↓
Build overlay locally
   ↓
kubectl kustomize overlays/dev
   ↓
If build fails:
    Fix Kustomize
   ↓
If build works:
    Check Kubernetes
```

Then inspect:

```bash
kubectl get pods -n <namespace>
```

and:

```bash
kubectl describe pod <pod> -n <namespace>
```

---

# 40. Interview Questions

## Q1. What is Kustomize?

> Kustomize is a Kubernetes configuration customization tool that allows us to reuse Base Kubernetes manifests and apply environment-specific customizations using Overlays.

---

## Q2. What is a Kustomize Base?

> A Base contains the common Kubernetes resources and configuration shared across environments.

Example:

```text
base/
├── deployment.yaml
├── service.yaml
└── kustomization.yaml
```

---

## Q3. What is an Overlay?

> An Overlay represents environment-specific customization on top of a Base.

Example:

```text
overlays/
├── dev/
├── qa/
└── prod/
```

---

## Q4. What is kustomization.yaml?

> `kustomization.yaml` is the configuration file that tells Kustomize which resources to use and which customizations, patches, labels, replicas, or image changes to apply.

---

## Q5. What is the difference between Helm and Kustomize?

> Helm uses templates and values to generate Kubernetes manifests, while Kustomize starts with Kubernetes manifests and applies environment-specific customizations or overlays without requiring templating.

---

## Q6. How do you manage DEV, QA and PROD using Kustomize?

```text
base/
   ↓
Common resources
   ↓
overlays/
   ├── dev/
   ├── qa/
   └── prod/
```

Each overlay references the Base and applies environment-specific configuration.

---

## Q7. How do you use Kustomize with Argo CD?

> I store the Base and environment Overlays in Git, then create separate Argo CD Applications pointing to the appropriate Overlay path such as `overlays/dev`, `overlays/qa`, or `overlays/prod`. Argo CD processes the selected Kustomize configuration and reconciles it with the target Kubernetes cluster.

---

# 41. Practice Checklist

## Practice 1 — Create Base

Create:

```text
base/
├── deployment.yaml
├── service.yaml
└── kustomization.yaml
```

Run:

```bash
kubectl kustomize base
```

Understand the generated YAML.

---

## Practice 2 — DEV Overlay

Create:

```text
overlays/dev/kustomization.yaml
```

Configure:

```text
replicas = 1
environment = dev
```

Run:

```bash
kubectl kustomize overlays/dev
```

---

## Practice 3 — QA Overlay

Create:

```text
overlays/qa/kustomization.yaml
```

Configure:

```text
replicas = 2
environment = qa
```

Run:

```bash
kubectl kustomize overlays/qa
```

---

## Practice 4 — PROD Overlay

Create:

```text
overlays/prod/kustomization.yaml
```

Configure:

```text
replicas = 5
environment = prod
```

Run:

```bash
kubectl kustomize overlays/prod
```

---

## Practice 5 — PROD Service Customization

Use a patch to change:

```text
ClusterIP
```

to:

```text
LoadBalancer
```

Run:

```bash
kubectl kustomize overlays/prod
```

Verify the generated Service.

Do not apply the PROD overlay yet.

---

## Practice 6 — QA Image Customization

Make QA use:

```text
nginx:1.28
```

while DEV and PROD remain:

```text
nginx:1.27
```

Run:

```bash
kubectl kustomize overlays/qa
```

Verify the generated image.

---

## Practice 7 — Push to GitHub

Repository structure:

```text
kustomize/
└── gitops-nginx/
    ├── base/
    └── overlays/
        ├── dev/
        ├── qa/
        └── prod/
```

Commit and push.

---

# 42. Important Commands

### Render Base

```bash
kubectl kustomize base
```

### Render DEV

```bash
kubectl kustomize overlays/dev
```

### Render QA

```bash
kubectl kustomize overlays/qa
```

### Render PROD

```bash
kubectl kustomize overlays/prod
```

### Render to file

```bash
kubectl kustomize overlays/dev > dev-rendered.yaml
```

### Manually apply DEV

```bash
kubectl apply -k overlays/dev
```

Remember:

```text
kubectl kustomize
    =
Render only

kubectl apply -k
    =
Render + Apply
```

For GitOps, Argo CD will perform the deployment.

---

# 43. Final Mental Model

The most important sentence:

> **Kustomize lets us keep common Kubernetes YAML in a Base and apply environment-specific changes through Overlays.**

Remember:

```text
Base
 =
Common configuration

Overlay
 =
Environment-specific configuration

kustomization.yaml
 =
Instructions for Kustomize
```

Complete architecture:

```text
                    Git
                     │
              ┌──────┼──────┐
              ↓      ↓      ↓
             DEV    QA     PROD
           overlay overlay overlay
              │      │       │
              └──────┼───────┘
                     ↓
                  Argo CD
                     ↓
             Kubernetes API
                     ↓
                    AKS
```

---

# 44. Three-Environment Mental Model

```text
                    BASE
                     │
          ┌──────────┼──────────┐
          ↓          ↓          ↓
         DEV        QA         PROD
          │          │          │
      replicas=1 replicas=2 replicas=5
      nginx:1.27  nginx:1.28  nginx:1.27
      ClusterIP   ClusterIP   LoadBalancer
          │          │          │
          ↓          ↓          ↓
       Argo CD    Argo CD    Argo CD
          │          │          │
          ↓          ↓          ↓
         AKS        AKS        AKS
```

For our lab, DEV/QA/PROD can be represented by different namespaces in the same AKS cluster.

In a real enterprise setup, they may be separate Kubernetes clusters.

---

# 45. Final Comparison

```text
                    Kubernetes Configuration

Raw YAML
   │
   └── Direct Kubernetes manifests


Helm
   │
   └── Templates + Values
          ↓
       Kubernetes


Kustomize
   │
   └── Base + Overlays
          ↓
       Kubernetes


GitOps
   │
   └── Git
        ↓
      Argo CD
        ↓
      Kubernetes
```

They can also be combined:

```text
Git
 │
 ├── Helm Chart
 │       ↓
 │    Argo CD
 │       ↓
 │      AKS
 │
 └── Kustomize Overlay
         ↓
      Argo CD
         ↓
        AKS
```

---

# 46. Lesson 14 Checklist

Before moving to the next lesson, make sure you understand:

- [ ] What Kustomize is
- [ ] What a Base is
- [ ] What an Overlay is
- [ ] What `kustomization.yaml` is
- [ ] How Base + Overlay works
- [ ] DEV overlay
- [ ] QA overlay
- [ ] PROD overlay
- [ ] Replica customization
- [ ] Label customization
- [ ] Image customization
- [ ] Service customization
- [ ] Patches
- [ ] `kubectl kustomize`
- [ ] `kubectl apply -k`
- [ ] Kustomize vs Helm
- [ ] Kustomize + Argo CD
- [ ] GitOps environment architecture
- [ ] Kustomize troubleshooting

---
