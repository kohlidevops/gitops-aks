# Lesson 12 — Helm Fundamentals for GitOps

## 🎯 Goal

In this lesson, we learn the fundamentals of **Helm** and understand why Helm is commonly used with GitOps.

By the end of this lesson, we should understand:

- What Helm is
- What a Helm Chart is
- What `values.yaml` is
- What Helm templates are
- What a Helm Release is
- How Helm converts templates into Kubernetes YAML
- `helm template`
- `helm install`
- `helm upgrade`
- `helm rollback`
- Helm vs raw Kubernetes YAML
- Helm vs GitOps
- Why Argo CD + Helm is useful

---

# 1. What is Helm?

Helm is a **package manager and templating tool for Kubernetes**.

Instead of writing many Kubernetes YAML files with hard-coded values, Helm allows us to create reusable templates.

For example, without Helm:

```yaml
replicas: 2

image:
  repository: nginx
  tag: "1.27"
```

If we want another environment with 5 replicas and a different image, we may need to modify the YAML manually.

With Helm:

```yaml
replicaCount: 2

image:
  repository: nginx
  tag: "1.27"
```

The values can be changed without rewriting the Kubernetes template.

---

# 2. Helm Mental Model

The most important concept:

```text
values.yaml + templates
          ↓
       Helm
          ↓
   Kubernetes YAML
          ↓
     Kubernetes
```

Think about it as:

```text
Template = How the Kubernetes resource should look

Values = What values we want

Helm = Combines both
```

For example:

```text
values.yaml

replicaCount: 3
image:
  repository: nginx
  tag: "1.27"
```

Template:

```yaml
replicas: {{ .Values.replicaCount }}

image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

Helm renders:

```yaml
replicas: 3

image: "nginx:1.27"
```

---

# 3. What is a Helm Chart?

A **Helm Chart** is a package containing Kubernetes templates and configuration.

Typical structure:

```text
gitops-nginx/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    └── service.yaml
```

Important files:

```text
Chart.yaml
    ↓
Chart information

values.yaml
    ↓
Configuration values

templates/
    ↓
Kubernetes resource templates
```

---

# 4. Create a Helm Chart

We will create a simple Nginx Helm Chart.

On the Ubuntu VM:

```bash
mkdir -p ~/lesson12
cd ~/lesson12
```

Create the chart:

```bash
helm create gitops-nginx
```

Helm creates a complete example chart.

For this lesson, we want a simple chart, so remove the default templates:

```bash
rm -rf gitops-nginx/templates/*
```

Our structure becomes:

```text
gitops-nginx/
├── Chart.yaml
├── values.yaml
└── templates/
```

We will create only the files needed for this lesson.

---

# 5. Chart.yaml

Edit:

```bash
vi gitops-nginx/Chart.yaml
```

Use:

```yaml
apiVersion: v2
name: gitops-nginx
description: Simple Nginx Helm chart for GitOps learning
type: application
version: 0.1.0
appVersion: "1.27"
```

### Important fields

```text
apiVersion
    Helm Chart API version

name
    Chart name

description
    Description of the chart

type
    application

version
    Chart version

appVersion
    Application version
```

---

# 6. values.yaml

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

This file contains configuration values.

For example:

```text
replicaCount = 2

image.repository = nginx

image.tag = 1.27

service.type = ClusterIP

service.port = 80
```

---

# 7. Deployment Template

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

Notice the Helm expressions:

```text
{{ .Release.Name }}

{{ .Values.replicaCount }}

{{ .Values.image.repository }}

{{ .Values.image.tag }}
```

These are not normal Kubernetes values.

They are Helm template expressions.

---

# 8. Service Template

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
spec:
  selector:
    app: {{ .Release.Name }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: 80
  type: {{ .Values.service.type }}
```

Again, Helm replaces the template expressions with values.

---

# 9. Helm Chart Structure

Our final chart:

```text
gitops-nginx/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    └── service.yaml
```

Mental model:

```text
Chart.yaml
     │
     ├── Chart information
     │
values.yaml
     │
     ├── Configuration
     │
templates/
     │
     ├── Deployment template
     └── Service template
```

---

# 10. Validate the Chart

Check Helm:

```bash
helm version
```

Run chart lint:

```bash
helm lint ./gitops-nginx
```

Expected result should indicate that the chart is valid.

The important command is:

```bash
helm lint
```

It helps identify chart/template problems before installation.

---

# 11. helm template

This is one of the most important Helm commands.

Run:

```bash
helm template my-nginx ./gitops-nginx
```

Helm renders the templates into normal Kubernetes YAML.

Conceptually:

```text
values.yaml
     +
templates/
     ↓
helm template
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

### Important

`helm template` does **not** deploy anything to Kubernetes.

It only renders the YAML.

---

# 12. Change the Replica Count

Change:

```yaml
replicaCount: 2
```

to:

```yaml
replicaCount: 3
```

Run:

```bash
helm template my-nginx ./gitops-nginx
```

Now the generated Deployment should contain:

```yaml
replicas: 3
```

This demonstrates how:

```text
values.yaml
     ↓
template
     ↓
generated Kubernetes YAML
```

works.

---

# 13. Override Values Using --set

We can also override values from the command line.

Run:

```bash
helm template my-nginx ./gitops-nginx --set replicaCount=5
```

Now Helm renders:

```yaml
replicas: 5
```

Even if `values.yaml` contains:

```yaml
replicaCount: 3
```

The command-line value overrides it for that rendering.

Mental model:

```text
values.yaml
     ↓
default configuration

--set
     ↓
temporary override
```

---

# 14. Helm Release

A **Release** is an installed instance of a Helm Chart.

For example:

```text
Chart:
gitops-nginx

Release:
helm-nginx
```

We could install the same chart multiple times:

```text
gitops-nginx Chart
       │
       ├── Release: dev-nginx
       │
       ├── Release: qa-nginx
       │
       └── Release: prod-nginx
```

The chart is the package/template.

The release is the running instance.

---

# 15. Install Helm Chart

For practice, use a separate namespace so we don't interfere with our existing Argo CD-managed application.

Create namespace:

```bash
kubectl create namespace helm-lab
```

Install the chart:

```bash
helm install helm-nginx ./gitops-nginx --namespace helm-lab
```

Now check releases:

```bash
helm list -n helm-lab
```

Check Kubernetes resources:

```bash
kubectl get all -n helm-lab
```

We should see resources such as:

```text
deployment.apps/helm-nginx
pod/helm-nginx-xxxxx
service/helm-nginx
```

---

# 16. Direct Helm Architecture

When we directly install Helm:

```text
Helm CLI
   │
   │ helm install
   ↓
Kubernetes API
   ↓
AKS
   ↓
Deployment
   ↓
Pods
```

This is **Helm**, but it is not GitOps.

Why?

Because Git is not controlling the desired state.

---

# 17. Helm Upgrade

Change:

```yaml
replicaCount: 3
```

to:

```yaml
replicaCount: 4
```

Then run:

```bash
helm upgrade helm-nginx ./gitops-nginx --namespace helm-lab
```

Check:

```bash
kubectl get deployment -n helm-lab
```

The deployment should now have:

```text
4 replicas
```

Check the release:

```bash
helm status helm-nginx -n helm-lab
```

---

# 18. Helm Status

Run:

```bash
helm status helm-nginx -n helm-lab
```

This shows information about the installed release.

Think:

```text
helm status
    ↓
What is the current state of this Helm release?
```

---

# 19. Get Helm Values

Run:

```bash
helm get values helm-nginx -n helm-lab
```

This shows the values used by the release.

---

# 20. Get Generated Kubernetes Manifest

Run:

```bash
helm get manifest helm-nginx -n helm-lab
```

This is very useful for troubleshooting.

It shows the Kubernetes manifests generated for the Helm release.

Mental model:

```text
Chart
 +
Values
 ↓
Helm
 ↓
Generated Kubernetes Manifest
```

---

# 21. Helm History

Run:

```bash
helm history helm-nginx -n helm-lab
```

Helm maintains release revisions.

For example:

```text
REVISION 1
    ↓
Initial installation

REVISION 2
    ↓
helm upgrade

REVISION 3
    ↓
Another upgrade
```

This becomes useful when troubleshooting upgrades.

---

# 22. Helm Rollback

Suppose a new release causes a problem.

We can roll back:

```bash
helm rollback helm-nginx 1 -n helm-lab
```

Check:

```bash
helm history helm-nginx -n helm-lab
```

The release can return to an earlier revision.

Mental model:

```text
Revision 1
    ↓
Revision 2
    ↓
Revision 3
    ↓
Problem
    ↓
helm rollback
    ↓
Earlier revision
```

---

# 23. Helm vs Raw Kubernetes YAML

### Without Helm

```text
deployment.yaml
service.yaml
configmap.yaml
secret.yaml
```

Configuration may be duplicated across environments.

For example:

```text
dev
    replicas: 2

qa
    replicas: 3

prod
    replicas: 10
```

### With Helm

We can use one chart:

```text
gitops-nginx/
├── Chart.yaml
├── values.yaml
└── templates/
```

Different values can be supplied for different environments.

For example:

```text
dev-values.yaml
qa-values.yaml
prod-values.yaml
```

The same templates can be reused.

---

# 24. Helm vs GitOps

This distinction is extremely important for interviews.

### Helm

Helm helps us:

```text
Package
+
Template
+
Install
+
Upgrade
+
Rollback
```

### GitOps

GitOps provides:

```text
Git
 ↓
Desired State
 ↓
Controller
 ↓
Reconciliation
 ↓
Kubernetes
```

Helm by itself does not continuously reconcile Kubernetes with Git.

---

# 25. Direct Helm vs GitOps Helm

### Direct Helm

```text
Developer
   ↓
helm install / helm upgrade
   ↓
Kubernetes
```

### GitOps + Helm

```text
Developer
   ↓
GitHub
   ↓
Helm Chart
   ↓
Argo CD
   ↓
Kubernetes API
   ↓
AKS
   ↓
Application
```

This is the architecture we will use in the next lesson.

---

# 26. Helm + Argo CD

In our GitOps architecture:

```text
                    GitHub
                      │
                      │
                Helm Chart
                      │
                      ↓
                   Argo CD
                      │
                      │ Reconcile
                      ↓
              Kubernetes API
                      │
                      ↓
                     AKS
                      │
              ┌───────┴───────┐
              ↓               ↓
         Deployment         Service
              ↓
             Pods
```

Important:

```text
Helm = packaging/template mechanism

Argo CD = GitOps controller/reconciliation mechanism

Kubernetes = runtime
```

---

# 27. Our Continuous Example

Throughout the GitOps lessons we used:

```text
Application:
gitops-nginx

Namespace:
gitops-demo

Container:
nginx:1.27
```

Previously:

```text
GitHub
   ↓
Raw Kubernetes YAML
   ↓
Argo CD
   ↓
AKS
```

After Helm:

```text
GitHub
   ↓
Helm Chart
   ↓
Argo CD
   ↓
AKS
```

This is the next step in our GitOps journey.

---

# 28. Important Helm Terms

| Term | Meaning |
|---|---|
| Helm | Kubernetes package manager and templating tool |
| Chart | Reusable Helm package |
| Chart.yaml | Chart metadata |
| values.yaml | Configuration values |
| templates | Parameterized Kubernetes manifests |
| Release | Installed instance of a chart |
| helm template | Render templates without deploying |
| helm install | Install a chart |
| helm upgrade | Upgrade a release |
| helm rollback | Roll back to an earlier revision |
| helm status | Show release status |
| helm history | Show release revisions |

---

# 29. Interview Discussion

### Q1. What is Helm?

Helm is a package manager and templating tool for Kubernetes. It allows us to package Kubernetes manifests into reusable charts and parameterize configuration using values.

---

### Q2. What is a Helm Chart?

A Helm Chart is a package containing Kubernetes resource templates, configuration values, and chart metadata.

---

### Q3. What is values.yaml?

`values.yaml` contains configuration values used by Helm templates.

For example:

```yaml
replicaCount: 3

image:
  repository: nginx
  tag: "1.27"
```

---

### Q4. What is a Helm Release?

A release is an installed instance of a Helm Chart.

For example:

```text
Chart:
gitops-nginx

Release:
helm-nginx
```

---

### Q5. What does helm template do?

It renders Helm templates into Kubernetes YAML without deploying the resources.

```bash
helm template my-nginx ./gitops-nginx
```

---

### Q6. What is the difference between helm install and helm upgrade?

`helm install` creates a new Helm release.

```bash
helm install helm-nginx ./gitops-nginx
```

`helm upgrade` updates an existing release.

```bash
helm upgrade helm-nginx ./gitops-nginx
```

---

### Q7. What is Helm rollback?

Rollback restores a Helm release to an earlier revision.

```bash
helm rollback helm-nginx 1
```

---

### Q8. Is Helm itself GitOps?

No.

Helm provides packaging, templating, installation, upgrades and rollback.

A GitOps controller such as Argo CD continuously compares desired state from Git with the Kubernetes cluster and reconciles differences.

---

### Q9. Why use Helm with Argo CD?

Helm provides reusable and parameterized Kubernetes manifests.

Argo CD provides Git-based desired state management and reconciliation.

Together:

```text
GitHub
   ↓
Helm Chart
   ↓
Argo CD
   ↓
Kubernetes
```

---

# 30. Production Scenario

### Scenario

Your company has:

```text
Development
QA
Production
```

All environments run the same application.

The application needs:

```text
Different replicas
Different image tags
Different service configuration
Different resource limits
```

Instead of maintaining completely different Kubernetes YAML files, we can create one Helm Chart:

```text
my-app/
├── Chart.yaml
├── values.yaml
└── templates/
```

Then provide environment-specific values:

```text
dev-values.yaml
qa-values.yaml
prod-values.yaml
```

Conceptually:

```text
                 Helm Chart
                     │
        ┌────────────┼────────────┐
        ↓            ↓            ↓
   dev-values    qa-values    prod-values
        ↓            ↓            ↓
       DEV          QA           PROD
```

With GitOps:

```text
GitHub
   ↓
Environment configuration
   ↓
Argo CD
   ↓
Helm
   ↓
Kubernetes
```

---

# 31. Most Important Mental Model

Remember this:

```text
Chart
    =
Reusable Kubernetes package

values.yaml
    =
Configuration

templates/
    =
Parameterized Kubernetes YAML

Release
    =
Installed instance of the Chart

Helm
    =
Renders/packages/manages Kubernetes applications

Argo CD
    =
GitOps controller that reconciles Git desired state with Kubernetes
```

The complete model:

```text
GitHub
   │
   │ Desired State
   ↓
Helm Chart
   │
   │ Templates + Values
   ↓
Argo CD
   │
   │ Reconciliation
   ↓
Kubernetes API
   │
   ↓
AKS
   │
   ↓
Application
```

---

# 32. Lesson 12 Checklist

Before moving to Lesson 13, make sure you understand:

- [ ] What Helm is
- [ ] What a Helm Chart is
- [ ] What `Chart.yaml` is
- [ ] What `values.yaml` is
- [ ] What `templates/` contains
- [ ] What a Helm Release is
- [ ] `helm lint`
- [ ] `helm template`
- [ ] `helm install`
- [ ] `helm upgrade`
- [ ] `helm status`
- [ ] `helm get values`
- [ ] `helm get manifest`
- [ ] `helm history`
- [ ] `helm rollback`
- [ ] Helm vs raw Kubernetes YAML
- [ ] Helm vs GitOps
- [ ] Direct Helm vs Argo CD + Helm

---

# Lesson 12 Complete

## Next Lesson

### Lesson 13 — Helm + Argo CD

In the next lesson we will connect what we learned here:

```text
Helm Chart
    +
Argo CD
    +
AKS
```

We will move from:

```text
GitHub
   ↓
Raw Kubernetes YAML
   ↓
Argo CD
   ↓
AKS
```

to:

```text
GitHub
   ↓
Helm Chart
   ↓
Argo CD
   ↓
AKS
```

The important goal will be understanding **how Argo CD uses a Helm Chart to deploy and continuously reconcile an application**.
