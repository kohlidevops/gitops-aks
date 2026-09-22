# Lesson 14 — Kustomize + Argo CD Multi-Environment GitOps Lab

## Objective

In this lab, we built a complete multi-environment GitOps deployment using:

- Kubernetes / Azure AKS
- Kustomize
- GitHub
- Argo CD
- DEV / QA / PROD environment overlays

We started with a common Kubernetes base and created environment-specific customizations using Kustomize.

The final architecture is:

```text
                         GitHub
                           |
                 gitops-kustomize-aks
                           |
                      Kustomize
                           |
              +------------+------------+
              |            |            |
             DEV           QA          PROD
              |            |            |
           Overlay       Overlay      Overlay
              |            |            |
              v            v            v
          Argo CD       Argo CD      Argo CD
              |            |            |
              v            v            v
         gitops-dev    gitops-qa    gitops-prod
              |            |            |
           ClusterIP     ClusterIP   LoadBalancer
```

---

# 1. What We Built

We created one common application:

```text
gitops-nginx
```

using a common Kustomize `base`.

Then we created three environment overlays:

```text
DEV
QA
PROD
```

Each environment uses the same base but has different configuration.

Final configuration:

| Environment | Namespace | Replicas | Image | Service |
|---|---|---:|---|---|
| DEV | `gitops-dev` | 2 | `nginx:1.27` | ClusterIP |
| QA | `gitops-qa` | 2 | `nginx:1.29` | ClusterIP |
| PROD | `gitops-prod` | 6 | `nginx:1.27` | LoadBalancer |

The exact replica counts and QA image changed during the lab through Git commits.

The important concept is that **we did not create separate Deployment YAML files for DEV, QA, and PROD**.

We created one base and customized it through overlays.

---

# 2. Git Repository

For Lesson 14 we created a new GitHub repository:

```text
gitops-kustomize-aks
```

This was kept separate from the previous:

```text
gitops-aks-demo
```

repository used in earlier lessons.

Repository structure:

```text
gitops-kustomize-aks/
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
            │
            ├── dev/
            │   ├── kustomization.yaml
            │   └── namespace.yaml
            │
            ├── qa/
            │   ├── kustomization.yaml
            │   └── namespace.yaml
            │
            └── prod/
                ├── kustomization.yaml
                ├── namespace.yaml
                └── service-patch.yaml
```

---

# 3. Kustomize Base

The base contains the common application configuration.

```text
base/
├── deployment.yaml
├── service.yaml
└── kustomization.yaml
```

The base should contain configuration that is common across environments.

---

## 3.1 Base Deployment

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

The base provides:

```text
Deployment name = gitops-nginx
Container       = nginx
Default image   = nginx:1.27
Container port  = 80
```

The replica count is later customized by the environment overlays.

---

# 4. Base Service

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

The base Service is:

```text
ClusterIP
```

DEV and QA keep this configuration.

PROD changes it to `LoadBalancer` using a patch.

---

# 5. Base Kustomization

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
```

The base therefore combines:

```text
deployment.yaml
+
service.yaml
```

---

# 6. DEV Overlay

DEV structure:

```text
overlays/dev/
├── kustomization.yaml
└── namespace.yaml
```

## Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: gitops-dev
```

## Kustomization

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: gitops-dev

resources:
  - ../../base
  - namespace.yaml

replicas:
  - name: gitops-nginx
    count: 1

labels:
  - pairs:
      environment: dev
```

The initial DEV configuration was:

```text
Namespace    = gitops-dev
Replicas     = 1
Image        = nginx:1.27
Service      = ClusterIP
Environment  = dev
```

During the GitOps change exercise, DEV was changed from:

```text
1 replica
```

to:

```text
2 replicas
```

through a Git change.

---

# 7. QA Overlay

QA structure:

```text
overlays/qa/
├── kustomization.yaml
└── namespace.yaml
```

## Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: gitops-qa
```

## Kustomization

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: gitops-qa

resources:
  - ../../base
  - namespace.yaml

replicas:
  - name: gitops-nginx
    count: 2

images:
  - name: nginx
    newTag: "1.28"

labels:
  - pairs:
      environment: qa
```

The initial QA configuration was:

```text
Namespace    = gitops-qa
Replicas     = 2
Image        = nginx:1.28
Service      = ClusterIP
Environment  = qa
```

During the GitOps change exercise, the image was changed:

```text
nginx:1.28
        ↓
nginx:1.29
```

through Git.

---

# 8. PROD Overlay

PROD structure:

```text
overlays/prod/
├── kustomization.yaml
├── namespace.yaml
└── service-patch.yaml
```

## Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: gitops-prod
```

## Service Patch

```yaml
apiVersion: v1
kind: Service
metadata:
  name: gitops-nginx
spec:
  type: LoadBalancer
```

The base Service is:

```text
ClusterIP
```

The PROD patch changes it to:

```text
LoadBalancer
```

## PROD Kustomization

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: gitops-prod

resources:
  - ../../base
  - namespace.yaml

replicas:
  - name: gitops-nginx
    count: 5

patches:
  - path: service-patch.yaml

labels:
  - pairs:
      environment: prod
```

Initial PROD configuration:

```text
Namespace    = gitops-prod
Replicas     = 5
Image        = nginx:1.27
Service      = LoadBalancer
Environment  = prod
```

During the GitOps change exercise, PROD was changed from:

```text
5 replicas
```

to:

```text
6 replicas
```

through Git.

---

# 9. Validate Kustomize Before Argo CD

Before deploying anything to AKS, we validated all three overlays.

DEV:

```bash
kubectl kustomize kustomize/gitops-nginx/overlays/dev
```

QA:

```bash
kubectl kustomize kustomize/gitops-nginx/overlays/qa
```

PROD:

```bash
kubectl kustomize kustomize/gitops-nginx/overlays/prod
```

All three rendered successfully:

```text
DEV OK
QA OK
PROD OK
```

This demonstrated:

```text
Kustomize overlay
       ↓
Rendered Kubernetes manifests
```

before involving Argo CD.

---

# 10. Argo CD Applications

We created three separate Argo CD Applications.

```text
gitops-nginx-dev
gitops-nginx-qa
gitops-nginx-prod
```

Each Application points to the same Git repository but a different Kustomize overlay.

---

## DEV Application

```text
Application:
gitops-nginx-dev

Repository:
gitops-kustomize-aks

Path:
kustomize/gitops-nginx/overlays/dev

Destination:
https://kubernetes.default.svc

Namespace:
gitops-dev
```

---

## QA Application

```text
Application:
gitops-nginx-qa

Repository:
gitops-kustomize-aks

Path:
kustomize/gitops-nginx/overlays/qa

Destination:
https://kubernetes.default.svc

Namespace:
gitops-qa
```

---

## PROD Application

```text
Application:
gitops-nginx-prod

Repository:
gitops-kustomize-aks

Path:
kustomize/gitops-nginx/overlays/prod

Destination:
https://kubernetes.default.svc

Namespace:
gitops-prod
```

---

# 11. Why Three Argo CD Applications?

Each Application watches a different desired state.

```text
                   GitHub
                      |
          gitops-kustomize-aks
                      |
       +--------------+--------------+
       |              |              |
       v              v              v
    DEV path        QA path        PROD path
       |              |              |
       v              v              v
    Argo CD         Argo CD         Argo CD
    Application     Application     Application
       |              |              |
       v              v              v
  gitops-dev       gitops-qa      gitops-prod
```

Therefore:

```text
DEV change
    ↓
Only DEV Application becomes OutOfSync

QA change
    ↓
Only QA Application becomes OutOfSync

PROD change
    ↓
Only PROD Application becomes OutOfSync
```

This was demonstrated during the lab.

---

# 12. DEV Deployment

After creating the DEV Argo CD Application, we synchronized it.

We verified:

```bash
kubectl get all -n gitops-dev
```

The resulting resources included:

```text
Pod
Service
Deployment
ReplicaSet
```

We verified:

```bash
kubectl get deployment -n gitops-dev
```

Initially:

```text
gitops-nginx   1/1
```

Image:

```bash
kubectl get deployment gitops-nginx -n gitops-dev \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

Result:

```text
nginx:1.27
```

Environment label:

```bash
kubectl get deployment gitops-nginx -n gitops-dev \
  -o jsonpath='{.metadata.labels.environment}{"\n"}'
```

Result:

```text
dev
```

Service:

```text
ClusterIP
```

---

# 13. QA Deployment

We created and synchronized:

```text
gitops-nginx-qa
```

The resulting environment had:

```text
Namespace:
gitops-qa

Replicas:
2

Image:
nginx:1.28

Service:
ClusterIP

Environment:
qa
```

The Deployment was verified as:

```text
2/2
```

The image was verified as:

```text
nginx:1.28
```

---

# 14. PROD Deployment

We created and synchronized:

```text
gitops-nginx-prod
```

The resulting environment had:

```text
Namespace:
gitops-prod

Replicas:
5

Image:
nginx:1.27

Service:
LoadBalancer

Environment:
prod
```

The Deployment was verified:

```text
5/5
```

The Service initially showed:

```text
EXTERNAL-IP: <pending>
```

and then Azure assigned an external IP.

The final PROD Service was:

```text
TYPE:
LoadBalancer
```

with an Azure external IP.

This demonstrated the difference between:

```text
ClusterIP
```

and:

```text
LoadBalancer
```

in AKS.

---

# 15. Final Multi-Environment Verification

We verified:

```bash
kubectl get deployment -A | grep gitops-nginx
```

Result:

```text
gitops-dev     gitops-nginx    1/1
gitops-qa      gitops-nginx    2/2
gitops-prod    gitops-nginx    5/5
```

After the environment-change exercises, the final desired configuration became:

```text
gitops-dev     gitops-nginx    2/2
gitops-qa      gitops-nginx    2/2
gitops-prod    gitops-nginx    6/6
```

Services were verified:

```bash
kubectl get svc -A | grep gitops-nginx
```

Result:

```text
gitops-dev     gitops-nginx    ClusterIP
gitops-qa      gitops-nginx    ClusterIP
gitops-prod    gitops-nginx    LoadBalancer
```

Argo CD Applications were verified:

```bash
kubectl get applications -n argocd
```

Result:

```text
gitops-nginx-dev     Synced    Healthy
gitops-nginx-prod    Synced    Healthy
gitops-nginx-qa      Synced    Healthy
```

---

# 16. GitOps Change Exercise — DEV

We changed:

```yaml
count: 1
```

to:

```yaml
count: 2
```

in:

```text
overlays/dev/kustomization.yaml
```

Then:

```bash
git add .
git commit -m "Scale dev nginx to two replicas"
git push
```

Argo CD detected:

```text
OutOfSync
```

because:

```text
Git desired state = 2
Kubernetes actual state = 1
```

After synchronization:

```text
DEV = 2 replicas
```

---

# 17. GitOps Change Exercise — QA

We changed:

```text
nginx:1.28
```

to:

```text
nginx:1.29
```

in:

```text
overlays/qa/kustomization.yaml
```

Then:

```bash
git add .
git commit -m "Upgrade QA nginx image"
git push
```

Argo CD detected the QA Application as:

```text
OutOfSync
```

After synchronization:

```text
QA image = nginx:1.29
```

DEV and PROD were not changed.

This demonstrated environment isolation.

---

# 18. GitOps Change Exercise — PROD

We changed:

```yaml
count: 5
```

to:

```yaml
count: 6
```

in:

```text
overlays/prod/kustomization.yaml
```

Then:

```bash
git add .
git commit -m "Scale production nginx to six replicas"
git push
```

Argo CD detected:

```text
gitops-nginx-prod
OutOfSync
```

After synchronization:

```text
PROD = 6 replicas
```

The PROD Service remained:

```text
LoadBalancer
```

because the Service configuration was not changed.

---

# 19. GitOps Drift Exercise

We intentionally created drift directly in Kubernetes.

Git desired state:

```text
PROD replicas = 6
```

We manually changed Kubernetes:

```bash
kubectl scale deployment gitops-nginx \
  -n gitops-prod \
  --replicas=3
```

Now:

```text
Git desired state:
6

Kubernetes actual state:
3
```

Argo CD detected:

```text
OutOfSync
```

We then synchronized the Application.

Kubernetes returned to:

```text
6 replicas
```

This demonstrated:

```text
Git desired state
       |
       v
     Argo CD
       |
       v
Kubernetes actual state
       |
       |
   manual change
       |
       v
     DRIFT
       |
       v
   OutOfSync
       |
       v
     Sync
       |
       v
Desired state restored
```

---

# 20. Important Concepts Learned

## Kustomize Base

Contains common Kubernetes configuration.

```text
base
```

Example:

```text
Deployment
Service
```

---

## Kustomize Overlay

Environment-specific customization.

```text
overlays/dev
overlays/qa
overlays/prod
```

---

## Kustomize `replicas`

Allows environment-specific replica counts without duplicating the Deployment YAML.

Example:

```yaml
replicas:
  - name: gitops-nginx
    count: 5
```

---

## Kustomize `images`

Allows an environment to change the container image.

Example:

```yaml
images:
  - name: nginx
    newTag: "1.29"
```

---

## Kustomize Patch

Allows an overlay to modify part of a resource.

Example:

```yaml
patches:
  - path: service-patch.yaml
```

PROD used this to change:

```text
ClusterIP
```

to:

```text
LoadBalancer
```

---

# 21. Kustomize vs Helm

Both can solve environment customization, but their approach is different.

### Kustomize

```text
Base YAML
   +
Overlay
   ↓
Modified YAML
```

It works directly with Kubernetes manifests.

### Helm

```text
Templates
   +
values.yaml
   ↓
Rendered Kubernetes manifests
```

Helm uses templates and values.

The key mental model from our lessons:

```text
Kustomize:
"Take these Kubernetes YAML files and customize them."

Helm:
"Take this parameterized chart and render Kubernetes YAML."
```

Both can be used with Argo CD.

---

# 22. Argo CD + Kustomize Flow

The complete GitOps flow is:

```text
Developer
    |
    | Git commit / push
    v
GitHub
    |
    | desired state
    v
Kustomize Overlay
    |
    | rendered manifests
    v
Argo CD
    |
    | compare desired vs actual
    v
Kubernetes API
    |
    v
AKS
    |
    v
Pods / Services / Deployments
```

---

# 23. Desired State vs Actual State

This lab demonstrated the difference clearly.

### Desired State

Stored in:

```text
GitHub
```

Example:

```text
PROD replicas = 6
```

### Actual State

Exists in:

```text
AKS
```

Example:

```text
PROD replicas = 3
```

### Drift

```text
Desired != Actual
```

Argo CD reports:

```text
OutOfSync
```

---

# 24. Important Interview Mental Model

If an interviewer asks:

> How do you manage DEV, QA and PROD using Kustomize and Argo CD?

A practical answer is:

> "I maintain a common Kustomize base containing the reusable Kubernetes resources and separate overlays for DEV, QA and PROD. Each overlay contains environment-specific customizations such as replicas, image versions, namespaces and Service configuration. I create separate Argo CD Applications pointing to each overlay. Argo CD renders the Kustomize configuration, compares the desired state with the live AKS state, and synchronizes the environment when required."

---

# 25. Another Interview Question

### Why don't you maintain three separate Deployment YAML files?

Because that creates duplication.

Instead:

```text
base/deployment.yaml
```

contains the common application definition.

Then:

```text
dev/kustomization.yaml
qa/kustomization.yaml
prod/kustomization.yaml
```

customize only what is different.

This makes environment configuration easier to maintain and reduces duplicated YAML.

---

# 26. Production Mental Model

A production repository could look like:

```text
repo/
│
├── base/
│
└── overlays/
    ├── dev/
    ├── qa/
    └── prod/
```

Argo CD could then have:

```text
Application: myapp-dev
Path: overlays/dev

Application: myapp-qa
Path: overlays/qa

Application: myapp-prod
Path: overlays/prod
```

Each environment has its own desired state and reconciliation lifecycle.

---

# 27. Troubleshooting Commands Used in This Lab

Check Applications:

```bash
kubectl get applications -n argocd
```

Describe an Application:

```bash
kubectl describe application gitops-nginx-dev -n argocd
```

Check deployments:

```bash
kubectl get deployment -A
```

Check pods:

```bash
kubectl get pods -A
```

Check services:

```bash
kubectl get svc -A
```

Check a specific environment:

```bash
kubectl get all -n gitops-dev
kubectl get all -n gitops-qa
kubectl get all -n gitops-prod
```

Check deployment image:

```bash
kubectl get deployment gitops-nginx -n gitops-qa \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

Check replicas:

```bash
kubectl get deployment gitops-nginx -n gitops-prod
```

Render Kustomize:

```bash
kubectl kustomize kustomize/gitops-nginx/overlays/dev
kubectl kustomize kustomize/gitops-nginx/overlays/qa
kubectl kustomize kustomize/gitops-nginx/overlays/prod
```

---

# 28. Final Architecture

```text
                         GitHub
                           |
                gitops-kustomize-aks
                           |
                  Kustomize Structure
                           |
               +-----------+-----------+
               |           |           |
             BASE        OVERLAYS
                           |
              +------------+------------+
              |            |            |
             DEV           QA          PROD
              |            |            |
              v            v            v
       kustomize/dev  kustomize/qa  kustomize/prod
              |            |            |
              v            v            v
       Argo CD App    Argo CD App    Argo CD App
              |            |            |
              v            v            v
         gitops-dev    gitops-qa    gitops-prod
              |            |            |
              v            v            v
          Kubernetes    Kubernetes    Kubernetes
              |            |            |
             AKS          AKS          AKS
```

---

# 29. Lesson 14 Key Takeaways

```text
1. Kustomize provides environment customization.

2. Base contains common Kubernetes resources.

3. Overlays contain environment-specific changes.

4. We can avoid duplicating Deployment YAML.

5. Different environments can have different replicas.

6. Different environments can use different image versions.

7. Different environments can use different Service types.

8. Argo CD can point each Application to a different Kustomize overlay.

9. Git remains the desired-state source of truth.

10. Argo CD compares desired state with live state.

11. Git changes can make an Application OutOfSync.

12. Manual synchronization applies the Git desired state.

13. Direct Kubernetes changes can create GitOps drift.

14. Argo CD detects drift.

15. Synchronization restores the desired state.

16. DEV, QA and PROD can be independently managed while sharing the same base.
```

---

# 30. Final Lesson 14 Mental Model

The complete mental model to remember is:

```text
                Kustomize
                    |
          "How should environments
             be different?"
                    |
                    v
             DEV / QA / PROD
                    |
                    v
                GitHub
                    |
                    v
                Argo CD
                    |
          "Does Kubernetes match
             Git's desired state?"
                    |
                    v
                  AKS
                    |
                    v
             Actual workload
```

### One-line interview answer

> **Kustomize manages environment-specific Kubernetes configuration, while Argo CD continuously reconciles those desired configurations from Git with the actual state running in Kubernetes.**

---

## Lesson 14 Completion Checklist

- [x] Created separate Lesson 14 GitHub repository
- [x] Created Kustomize base
- [x] Created DEV overlay
- [x] Created QA overlay
- [x] Created PROD overlay
- [x] Created environment namespaces
- [x] Used Kustomize replica customization
- [x] Used Kustomize image customization
- [x] Used Kustomize Service patch
- [x] Validated DEV rendering
- [x] Validated QA rendering
- [x] Validated PROD rendering
- [x] Created DEV Argo CD Application
- [x] Created QA Argo CD Application
- [x] Created PROD Argo CD Application
- [x] Deployed DEV to AKS
- [x] Deployed QA to AKS
- [x] Deployed PROD to AKS
- [x] Verified ClusterIP in DEV
- [x] Verified ClusterIP in QA
- [x] Verified LoadBalancer in PROD
- [x] Changed DEV replicas through Git
- [x] Changed QA image through Git
- [x] Changed PROD replicas through Git
- [x] Observed OutOfSync
- [x] Tested Kubernetes drift
- [x] Reconciled drift using Argo CD
- [x] Verified all three Applications as Synced and Healthy

# Lesson 14 Completed

**Kustomize + Argo CD + AKS + DEV/QA/PROD Multi-Environment GitOps**
