# Lesson 03 — Kubernetes Deployment Before GitOps

## 🎯 Objective

Before using GitOps and Argo CD, we need to understand how Kubernetes works when we deploy an application **manually**.

In this lesson, we will deploy a sample Nginx application to Azure AKS using `kubectl`.

We will learn:

- Kubernetes Namespace
- Deployment
- ReplicaSet
- Pod
- Service
- ConfigMap
- Secret
- Labels and Selectors
- `kubectl apply`
- `kubectl get`
- `kubectl describe`
- `kubectl logs`
- Scaling
- Kubernetes desired state vs actual state
- Basic Kubernetes troubleshooting

---

# 1. What We Are Going to Build

Our example application is:

```text
Nginx
```

The Kubernetes architecture will look like:

```text
Azure
  |
  v
AKS Cluster
  |
  v
Namespace: gitops-demo
  |
  +----------------------+
  |                      |
  v                      v
Deployment: nginx       Service: nginx
  |                      |
  v                      v
ReplicaSet             ClusterIP
  |
  +--------+--------+
  |                 |
  v                 v
Pod nginx-xxx      Pod nginx-yyy
  |
  v
Nginx Application
```

We will also create:

```text
ConfigMap
Secret
```

---

# 2. Important Kubernetes Mental Model

Remember this:

```text
Namespace
   ↓
Deployment
   ↓
ReplicaSet
   ↓
Pods
   ↓
Container
   ↓
Application
```

And:

```text
Service
   ↓
finds Pods using Labels
   ↓
provides stable network access
```

---

# 3. Kubernetes Namespace

A Namespace provides a logical boundary inside a Kubernetes cluster.

We will create:

```text
gitops-demo
```

Example:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: gitops-demo
```

Create it:

```bash
kubectl apply -f namespace.yaml
```

Check:

```bash
kubectl get namespaces
```

Or:

```bash
kubectl get ns
```

Expected:

```text
NAME
default
kube-system
kube-public
kube-node-lease
gitops-demo
```

---

# 4. Kubernetes Deployment

A Deployment manages application Pods.

Example:

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

Save this as:

```text
deployment.yaml
```

Apply:

```bash
kubectl apply -f deployment.yaml
```

Check:

```bash
kubectl get deployments -n gitops-demo
```

Expected:

```text
NAME    READY   UP-TO-DATE   AVAILABLE
nginx   2/2     2            2
```

---

# 5. What Happens Behind the Scenes?

When we execute:

```bash
kubectl apply -f deployment.yaml
```

we are not directly creating Pods ourselves.

The flow is:

```text
kubectl
   |
   v
Kubernetes API Server
   |
   v
Deployment
   |
   v
ReplicaSet
   |
   v
Pods
   |
   v
Container
   |
   v
Nginx
```

The Deployment says:

```text
I want 2 nginx Pods.
```

The ReplicaSet makes sure that 2 Pods exist.

If one Pod disappears:

```text
2 Pods
 ↓
1 Pod crashes
 ↓
ReplicaSet detects only 1 Pod
 ↓
ReplicaSet creates another Pod
 ↓
2 Pods again
```

This is one of the important Kubernetes concepts.

---

# 6. Deployment vs ReplicaSet vs Pod

### Deployment

Responsible for:

```text
Application version
Replica management
Rolling updates
Rollback
```

### ReplicaSet

Responsible for:

```text
Maintaining the required number of Pods
```

### Pod

The smallest deployable unit in Kubernetes.

Our Pod contains:

```text
Pod
 |
 └── nginx container
```

Check them:

```bash
kubectl get deployment -n gitops-demo
```

```bash
kubectl get replicaset -n gitops-demo
```

```bash
kubectl get pods -n gitops-demo
```

You can see the complete relationship:

```text
Deployment
    |
    v
ReplicaSet
    |
    +------ Pod
    |
    +------ Pod
```

---

# 7. Labels

Labels are key/value metadata attached to Kubernetes objects.

Our Pods have:

```yaml
labels:
  app: nginx
```

Check:

```bash
kubectl get pods -n gitops-demo --show-labels
```

Example:

```text
NAME                     LABELS
nginx-xxxxx              app=nginx
nginx-yyyyy              app=nginx
```

Labels are extremely important in Kubernetes because other resources use them to find objects.

---

# 8. Selectors

Our Deployment contains:

```yaml
selector:
  matchLabels:
    app: nginx
```

This means:

```text
Deployment manages Pods having:

app=nginx
```

Our Service also uses:

```yaml
selector:
  app: nginx
```

This means:

```text
Service sends traffic to Pods having:

app=nginx
```

Important relationship:

```text
Label:

app=nginx


Deployment Selector:

app=nginx


Service Selector:

app=nginx
```

Therefore:

```text
Service
   |
   | selector: app=nginx
   |
   v
Pods
  app=nginx
```

---

# 9. Kubernetes Service

Pods are temporary.

Their IP addresses can change.

A Service provides a stable network endpoint for the application.

Create:

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

Save as:

```text
service.yaml
```

Apply:

```bash
kubectl apply -f service.yaml
```

Check:

```bash
kubectl get service -n gitops-demo
```

Or:

```bash
kubectl get svc -n gitops-demo
```

Expected:

```text
NAME    TYPE        CLUSTER-IP      PORT(S)
nginx   ClusterIP   10.x.x.x        80/TCP
```

---

# 10. Service Port vs TargetPort

This is important for interviews.

```yaml
ports:
  - port: 80
    targetPort: 80
```

Meaning:

```text
Service port
     |
     v
    80
     |
     v
Pod/container port
     |
     v
    80
```

`port`:

```text
Port exposed by the Service
```

`targetPort`:

```text
Port on the application Pod
```

---

# 11. ConfigMap

ConfigMap stores non-sensitive configuration.

Example:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: gitops-demo
data:
  APP_ENV: "dev"
  LOG_LEVEL: "info"
```

Save as:

```text
configmap.yaml
```

Apply:

```bash
kubectl apply -f configmap.yaml
```

Check:

```bash
kubectl get configmap -n gitops-demo
```

Get details:

```bash
kubectl describe configmap nginx-config -n gitops-demo
```

Concept:

```text
ConfigMap
   |
   +-- APP_ENV=dev
   |
   +-- LOG_LEVEL=info
```

ConfigMap is intended for configuration that is not sensitive.

---

# 12. Secret

Kubernetes Secret stores sensitive configuration such as:

```text
Username
Password
Token
Certificate
```

Example:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: nginx-secret
  namespace: gitops-demo
type: Opaque
stringData:
  DB_USERNAME: admin
  DB_PASSWORD: mypassword
```

Save as:

```text
secret.yaml
```

Apply:

```bash
kubectl apply -f secret.yaml
```

Check:

```bash
kubectl get secrets -n gitops-demo
```

Describe:

```bash
kubectl describe secret nginx-secret -n gitops-demo
```

Important:

Kubernetes Secrets are not automatically equivalent to a production-grade external secrets-management system.

Later in this GitOps course we will learn:

```text
Azure Key Vault
       |
       v
Kubernetes
       |
       v
Application
```

---

# 13. ConfigMap vs Secret

| Feature | ConfigMap | Secret |
|---|---|---|
| Purpose | Non-sensitive configuration | Sensitive configuration |
| Example | APP_ENV | DB_PASSWORD |
| Kubernetes Object | ConfigMap | Secret |
| Used by Pods | Yes | Yes |
| Production secret management | Usually not enough | Usually integrate with external secret management |

Remember:

```text
ConfigMap → configuration

Secret → sensitive configuration
```

---

# 14. kubectl apply

`kubectl apply` sends the desired configuration to the Kubernetes API server.

Example:

```bash
kubectl apply -f deployment.yaml
```

You can apply multiple files:

```bash
kubectl apply \
  -f namespace.yaml \
  -f deployment.yaml \
  -f service.yaml \
  -f configmap.yaml \
  -f secret.yaml
```

Or:

```bash
kubectl apply -f .
```

if the directory contains only the Kubernetes manifests you want to apply.

---

# 15. kubectl get

`kubectl get` is mainly used to see the current state.

Examples:

```bash
kubectl get nodes
```

```bash
kubectl get pods -n gitops-demo
```

```bash
kubectl get deployments -n gitops-demo
```

```bash
kubectl get services -n gitops-demo
```

```bash
kubectl get all -n gitops-demo
```

---

# 16. kubectl describe

`kubectl describe` provides detailed information about a Kubernetes object.

Example:

```bash
kubectl describe pod <POD_NAME> -n gitops-demo
```

It can show:

```text
Pod status
Node
Container
Image
Environment
Mounts
Events
Scheduling information
```

Most importantly, look at:

```text
Events
```

When troubleshooting Kubernetes, Events are often very useful.

---

# 17. kubectl logs

To see application/container logs:

```bash
kubectl logs <POD_NAME> -n gitops-demo
```

Example:

```bash
kubectl get pods -n gitops-demo
```

Then:

```bash
kubectl logs nginx-xxxxx -n gitops-demo
```

For a Pod with multiple containers:

```bash
kubectl logs <POD_NAME> -c <CONTAINER_NAME> -n gitops-demo
```

---

# 18. Basic Kubernetes Troubleshooting Flow

Suppose the application is not working.

Start with:

```bash
kubectl get pods -n gitops-demo
```

If the Pod is not healthy:

```bash
kubectl describe pod <POD_NAME> -n gitops-demo
```

Then:

```bash
kubectl logs <POD_NAME> -n gitops-demo
```

Then check:

```bash
kubectl get events -n gitops-demo
```

For the Deployment:

```bash
kubectl describe deployment nginx -n gitops-demo
```

For the Service:

```bash
kubectl get svc nginx -n gitops-demo
```

Check Service endpoints:

```bash
kubectl get endpoints nginx -n gitops-demo
```

The basic troubleshooting flow:

```text
Application problem
       |
       v
kubectl get pods
       |
       v
Pod unhealthy?
       |
      Yes
       |
       v
kubectl describe pod
       |
       v
Check Events
       |
       v
kubectl logs
```

---

# 19. Create the AKS Cluster

For this lesson we use Azure AKS.

First verify Azure CLI:

```bash
az version
```

Login:

```bash
az login
```

Check the current subscription:

```bash
az account show --output table
```

List subscriptions:

```bash
az account list --output table
```

If required:

```bash
az account set --subscription "<SUBSCRIPTION_NAME_OR_ID>"
```

---

# 20. Register AKS Resource Provider

If Azure reports:

```text
MissingSubscriptionRegistration
Microsoft.ContainerService
```

register the provider:

```bash
az provider register --namespace Microsoft.ContainerService
```

Check the status:

```bash
az provider show \
  --namespace Microsoft.ContainerService \
  --query "registrationState" \
  --output tsv
```

Expected:

```text
Registered
```

If it says:

```text
Registering
```

wait and check again.

---

# 21. Create Resource Group

Example:

```bash
az group create \
  --name rg-gitops-aks \
  --location centralindia
```

Verify:

```bash
az group show \
  --name rg-gitops-aks \
  --output table
```

---

# 22. AKS VM Size

The VM size depends on what your subscription allows in the selected Azure region.

The initial example used:

```text
Standard_B2s
```

but the subscription returned an error saying that this SKU was not allowed in `centralindia`.

The Azure response listed:

```text
Standard_B2s_v2
```

among the available sizes.

Therefore, first check the available sizes:

```bash
az vm list-skus \
  --location centralindia \
  --resource-type virtualMachines \
  --query "[?contains(name, 'Standard_B')].{Name:name}" \
  --output table
```

Use a VM size that Azure reports as available for your subscription.

For example:

```text
Standard_B2s_v2
```

---

# 23. Create AKS

Example:

```bash
az aks create \
  --resource-group rg-gitops-aks \
  --name aks-gitops-lab \
  --node-count 1 \
  --node-vm-size Standard_B2s_v2 \
  --generate-ssh-keys
```

The exact VM size can vary depending on subscription and region.

Verify the cluster:

```bash
az aks show \
  --resource-group rg-gitops-aks \
  --name aks-gitops-lab \
  --output table
```

---

# 24. Connect kubectl to AKS

Get AKS credentials:

```bash
az aks get-credentials \
  --resource-group rg-gitops-aks \
  --name aks-gitops-lab
```

Verify:

```bash
kubectl config current-context
```

Check cluster:

```bash
kubectl cluster-info
```

Check nodes:

```bash
kubectl get nodes
```

Expected:

```text
NAME       STATUS   ROLES    AGE
aks-xxxxx  Ready    <none>   ...
```

The important part is:

```text
STATUS = Ready
```

---

# 25. Create Lesson Directory

Create a working directory:

```text
lesson-03/
├── namespace.yaml
├── deployment.yaml
├── service.yaml
├── configmap.yaml
└── secret.yaml
```

---

# 26. Namespace Manifest

`namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: gitops-demo
```

Apply:

```bash
kubectl apply -f namespace.yaml
```

---

# 27. Deployment Manifest

`deployment.yaml`

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

Apply:

```bash
kubectl apply -f deployment.yaml
```

Check:

```bash
kubectl get deployment -n gitops-demo
```

---

# 28. Check ReplicaSet

```bash
kubectl get replicasets -n gitops-demo
```

Or:

```bash
kubectl get rs -n gitops-demo
```

You should see a ReplicaSet created by the Deployment.

---

# 29. Check Pods

```bash
kubectl get pods -n gitops-demo
```

Expected:

```text
NAME                     READY   STATUS    RESTARTS
nginx-xxxxxxxxxx-xxxxx   1/1     Running   0
nginx-xxxxxxxxxx-yyyyy   1/1     Running   0
```

Check labels:

```bash
kubectl get pods \
  -n gitops-demo \
  --show-labels
```

---

# 30. Create Service

`service.yaml`

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

Apply:

```bash
kubectl apply -f service.yaml
```

Check:

```bash
kubectl get svc -n gitops-demo
```

---

# 31. Test Nginx

Because the Service is:

```text
ClusterIP
```

it is not directly accessible from the Internet.

For our lab, use port-forwarding:

```bash
kubectl port-forward service/nginx 8080:80 -n gitops-demo
```

Then open:

```text
http://localhost:8080
```

You should see the Nginx welcome page.

Stop port forwarding with:

```text
Ctrl + C
```

---

# 32. Create ConfigMap

`configmap.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: gitops-demo
data:
  APP_ENV: "dev"
  LOG_LEVEL: "info"
```

Apply:

```bash
kubectl apply -f configmap.yaml
```

Check:

```bash
kubectl get configmap -n gitops-demo
```

Describe:

```bash
kubectl describe configmap nginx-config -n gitops-demo
```

---

# 33. Create Secret

`secret.yaml`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: nginx-secret
  namespace: gitops-demo
type: Opaque
stringData:
  DB_USERNAME: admin
  DB_PASSWORD: mypassword
```

Apply:

```bash
kubectl apply -f secret.yaml
```

Check:

```bash
kubectl get secret -n gitops-demo
```

Describe:

```bash
kubectl describe secret nginx-secret -n gitops-demo
```

Do not commit real production credentials to GitHub.

This example password is only for learning.

---

# 34. Verify Everything

Run:

```bash
kubectl get all -n gitops-demo
```

Also:

```bash
kubectl get configmap -n gitops-demo
```

```bash
kubectl get secret -n gitops-demo
```

```bash
kubectl get pods -n gitops-demo --show-labels
```

---

# 35. Practice Scaling

Initially:

```yaml
replicas: 2
```

Change it to:

```yaml
replicas: 3
```

Then apply:

```bash
kubectl apply -f deployment.yaml
```

Check:

```bash
kubectl get pods -n gitops-demo
```

You should now have:

```text
3 Pods
```

Check Deployment:

```bash
kubectl get deployment nginx -n gitops-demo
```

Expected:

```text
READY
3/3
```

---

# 36. Practice Kubernetes Drift

This is important for understanding GitOps later.

Our YAML says:

```yaml
replicas: 3
```

But manually change the live Kubernetes object:

```bash
kubectl scale deployment nginx \
  --replicas=1 \
  -n gitops-demo
```

Check:

```bash
kubectl get deployment nginx -n gitops-demo
```

Now Kubernetes has:

```text
Desired in YAML = 3
Actual in cluster = 1
```

This is a state mismatch.

Restore the YAML state:

```bash
kubectl apply -f deployment.yaml
```

Check:

```bash
kubectl get pods -n gitops-demo
```

The Deployment should return to:

```text
3 Pods
```

This is the concept that we will later call:

```text
GitOps Drift
```

In this lesson we manually fixed it.

Later:

```text
Git
 ↓
Argo CD
 ↓
detects drift
 ↓
reconciles Kubernetes
```

---

# 37. Important kubectl Commands

## Cluster

```bash
kubectl cluster-info
```

```bash
kubectl get nodes
```

## Namespace

```bash
kubectl get ns
```

```bash
kubectl get all -n gitops-demo
```

## Deployment

```bash
kubectl get deployment -n gitops-demo
```

```bash
kubectl describe deployment nginx -n gitops-demo
```

## ReplicaSet

```bash
kubectl get rs -n gitops-demo
```

## Pods

```bash
kubectl get pods -n gitops-demo
```

```bash
kubectl describe pod <POD_NAME> -n gitops-demo
```

```bash
kubectl logs <POD_NAME> -n gitops-demo
```

## Service

```bash
kubectl get svc -n gitops-demo
```

```bash
kubectl describe svc nginx -n gitops-demo
```

## ConfigMap

```bash
kubectl get configmap -n gitops-demo
```

```bash
kubectl describe configmap nginx-config -n gitops-demo
```

## Secret

```bash
kubectl get secret -n gitops-demo
```

```bash
kubectl describe secret nginx-secret -n gitops-demo
```

---

# 38. Interview Discussion

## Q1. What is a Deployment?

A Deployment manages application Pods and provides:

```text
Replica management
Rolling updates
Rollback
Desired state
```

---

## Q2. What happens when you run kubectl apply?

Example:

```bash
kubectl apply -f deployment.yaml
```

The flow is:

```text
kubectl
 ↓
Kubernetes API Server
 ↓
Deployment
 ↓
ReplicaSet
 ↓
Pods
```

The Kubernetes controllers work to make the actual state match the desired state.

---

## Q3. Why do we need a ReplicaSet?

A ReplicaSet maintains the required number of Pods.

For example:

```yaml
replicas: 3
```

means the application should have three matching Pods.

If one disappears, the ReplicaSet works to create another.

---

## Q4. Why do we need a Service?

Pods are not stable endpoints.

A Service provides:

```text
Stable network endpoint
Service discovery
Load balancing across matching Pods
```

---

## Q5. How does a Service know which Pods to send traffic to?

Using labels and selectors.

Pod:

```yaml
labels:
  app: nginx
```

Service:

```yaml
selector:
  app: nginx
```

Therefore:

```text
Service selector
       |
       v
app=nginx
       |
       v
Matching Pods
```

---

## Q6. What is the difference between port and targetPort?

```yaml
port: 80
targetPort: 80
```

`port` is the Service port.

`targetPort` is the port on the application Pod.

---

## Q7. ConfigMap vs Secret?

```text
ConfigMap → non-sensitive configuration

Secret → sensitive configuration
```

Examples:

```text
ConfigMap:
APP_ENV=dev

Secret:
DB_PASSWORD=...
```

---

## Q8. How do you troubleshoot a Pod?

Start with:

```bash
kubectl get pods
```

Then:

```bash
kubectl describe pod <POD_NAME>
```

Then:

```bash
kubectl logs <POD_NAME>
```

And:

```bash
kubectl get events
```

---

## Q9. What happens if a Pod crashes?

The Kubernetes controllers detect that the actual state is different from the desired state.

For a Deployment-managed application:

```text
Deployment
   ↓
ReplicaSet
   ↓
Pod crashes
   ↓
ReplicaSet detects missing Pod
   ↓
New Pod created
```

---

## Q10. What is Kubernetes desired state?

Example:

```yaml
replicas: 3
```

This declares:

```text
I want 3 Pods.
```

Kubernetes continuously works toward that state.

---

# 39. Most Important Lesson Concept

Remember this architecture:

```text
                 Kubernetes Cluster
                        |
                        v
                 Namespace
                 gitops-demo
                        |
                        v
                  Deployment
                     nginx
                        |
                        v
                   ReplicaSet
                        |
             +----------+----------+
             |                     |
             v                     v
           Pod                   Pod
        nginx:1.27            nginx:1.27
             |                     |
             +----------+----------+
                        |
                        v
                    Service
                      nginx
                        |
                        v
                  Stable endpoint
```

And:

```text
ConfigMap → Configuration

Secret → Sensitive configuration

Labels → Identify objects

Selectors → Find matching objects
```

---

# 40. Lesson 03 Practice Checklist

Complete these tasks:

- [ ] Verify Azure CLI
- [ ] Login using Azure CLI
- [ ] Verify Azure subscription
- [ ] Register `Microsoft.ContainerService` if required
- [ ] Create Resource Group
- [ ] Check available AKS VM sizes
- [ ] Create AKS cluster
- [ ] Configure kubectl
- [ ] Run `kubectl get nodes`
- [ ] Create `gitops-demo` namespace
- [ ] Create Nginx Deployment
- [ ] Verify Deployment
- [ ] Verify ReplicaSet
- [ ] Verify Pods
- [ ] Check Pod labels
- [ ] Create Service
- [ ] Verify Service
- [ ] Port-forward the Service
- [ ] Open Nginx in browser
- [ ] Create ConfigMap
- [ ] Create Secret
- [ ] Practice `kubectl get`
- [ ] Practice `kubectl describe`
- [ ] Practice `kubectl logs`
- [ ] Scale Deployment from 2 → 3
- [ ] Create manual drift from 3 → 1
- [ ] Restore desired state using `kubectl apply`

---

# 41. Final Mental Model

Before GitOps:

```text
Developer
   |
   | kubectl apply
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
   |
   v
Application
```

We manually tell Kubernetes:

```text
"Deploy this."
"Scale this."
"Change this."
"Restore this."
```

In the next stages, GitOps changes the workflow:

```text
Developer
   |
   v
GitHub
   |
   | Desired State
   v
Argo CD
   |
   | Reconciliation
   v
AKS / Kubernetes
   |
   v
Application
```

The key transition is:

```text
Manual kubectl deployment
          ↓
GitOps
          ↓
Git becomes the source of truth
          ↓
Argo CD continuously reconciles Kubernetes
```

---

# 42. Cleanup — Avoid Azure Charges

After completing the lesson, if you do not need the AKS lab anymore, delete the entire resource group:

```bash
az group delete \
  --name rg-gitops-aks \
  --yes \
  --no-wait
```

Verify:

```bash
az group show \
  --name rg-gitops-aks
```

If the resource group is being deleted, Azure may return a not-found response later.

**Important:** Deleting the Resource Group deletes the AKS cluster and resources inside it.

---

# Lesson 03 Complete

At the end of this lesson, you should be able to explain:

```text
Namespace
Deployment
ReplicaSet
Pod
Service
ConfigMap
Secret
Labels
Selectors
Desired State
Actual State
kubectl apply
kubectl get
kubectl describe
kubectl logs
```

Most importantly, you should understand what happens when:

```bash
kubectl apply -f deployment.yaml
```

is executed.

That Kubernetes foundation is required before moving to:

```text
Lesson 04 — Azure AKS Fundamentals
```
