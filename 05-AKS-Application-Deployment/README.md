# Lesson 05 — AKS Application Deployment

## Objective

In this lesson, we deploy a containerized application to Azure Kubernetes Service (AKS).

We will understand the complete flow:

```text
Application Code
      ↓
Dockerfile
      ↓
Docker Image
      ↓
Azure Container Registry (ACR)
      ↓
AKS pulls the image
      ↓
Deployment
      ↓
ReplicaSet
      ↓
Pods
      ↓
Service
      ↓
Azure Load Balancer
      ↓
Browser
```

The important idea is:

> **ACR stores container images. AKS runs containers. Deployment manages Pods. Service provides network access to the application.**

---

# 1. Environment Used

For this lab, we already have an existing AKS cluster.

We do **not** create another AKS cluster.

### AKS

```text
Cluster Name:
aks-gitops-lab

Resource Group:
rg-gitops-aks

Location:
centralindia

Kubernetes Version:
1.35.7

Node VM Size:
Standard_B2s_v2
```

### ACR

```text
Registry:
acrgitopslab11068

Login Server:
acrgitopslab11068.azurecr.io

SKU:
Basic

Location:
centralindia
```

### Kubernetes Namespace

We reuse the existing namespace:

```text
gitops-demo
```

---

# 2. Important Lab Environment Decision

Initially, the lesson used Docker commands from Azure Cloud Shell.

For example:

```bash
docker images
```

But Cloud Shell returned:

```text
Cannot connect to the Docker daemon at unix:///home/latchu/.docker/run/docker.sock.
Is the docker daemon running?
```

The reason is that the Cloud Shell environment does not provide a running Docker daemon suitable for our Docker build workflow.

We also tried:

```bash
az acr build
```

but ACR Tasks returned:

```text
TasksOperationsNotAllowed
```

Therefore, for this lab we use an **Azure Ubuntu VM with Docker installed**.

The final build/push environment is:

```text
Azure Ubuntu VM
      ↓
Docker
      ↓
Docker Image
      ↓
ACR
```

This is also useful because it gives us practical experience with Docker running on a Linux build machine.

---

# 3. Azure Authentication Problem We Encountered

When using the Ubuntu VM, we initially tried:

```bash
az login
```

The login failed with:

```text
AADSTS530035:
Access has been blocked by security defaults.
```

We also tried:

```bash
az login --use-device-code
```

which produced the same problem.

The issue was not:

- Docker
- Ubuntu
- AKS
- ACR
- Wrong password

The Microsoft Entra tenant's security policy was blocking the authentication flow.

Instead of disabling security settings, we used the Azure VM's **Managed Identity**.

---

# 4. Azure VM Managed Identity

The Ubuntu VM was given a **System Assigned Managed Identity**.

We verified the identity through the Azure Instance Metadata Service.

```bash
curl -H Metadata:true \
  "http://169.254.169.254/metadata/identity/info?api-version=2018-02-01"
```

After enabling the identity, Azure returned:

```json
{
  "tenantId": "dd08c73a-948e-45de-9ce4-845e0568830a"
}
```

We then requested an Azure token:

```bash
curl -H Metadata:true \
  "http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://management.azure.com/" \
  | python3 -m json.tool
```

The response contained:

```json
{
    "access_token": "*****",
    "client_id": "e15570e1-8ac0-411b-90d6-4684bfb0fbf6",
    "expires_in": "86400",
    "resource": "https://management.azure.com/",
    "token_type": "Bearer"
}
```

Never expose the real `access_token`.

---

# 5. Authenticate Azure CLI Using Managed Identity

Instead of:

```bash
az login
```

we use:

```bash
az login --identity
```

This means:

```text
Ubuntu VM
    ↓
System Assigned Managed Identity
    ↓
Microsoft Entra ID
    ↓
Azure access token
```

This is a common Azure workload authentication pattern.

The VM does not need a stored Azure username/password.

---

# 6. Docker Application

We create a simple Nginx-based application.

Directory:

```text
aks-demo/
├── Dockerfile
└── index.html
```

---

# 7. Application Code

Create `index.html`:

```html
<!DOCTYPE html>
<html>
<head>
    <title>AKS GitOps Demo</title>
</head>
<body>
    <h1>Hello from AKS!</h1>
    <p>Application deployed using Docker, ACR and AKS.</p>
</body>
</html>
```

---

# 8. Dockerfile

Create `Dockerfile`:

```dockerfile
FROM nginx:1.27

COPY index.html /usr/share/nginx/html/index.html

EXPOSE 80
```

### What happens?

```text
FROM nginx:1.27
```

Uses the Nginx container image as the base image.

```text
COPY index.html /usr/share/nginx/html/index.html
```

Copies our application page into Nginx's web root.

```text
EXPOSE 80
```

Documents that the container listens on port 80.

---

# 9. Build the Docker Image

From the Ubuntu VM:

```bash
cd ~/aks-demo
```

Build:

```bash
docker build \
  -t acrgitopslab11068.azurecr.io/aks-demo:v1 \
  .
```

Verify:

```bash
docker images
```

Expected image:

```text
acrgitopslab11068.azurecr.io/aks-demo
```

with tag:

```text
v1
```

---

# 10. ACR Authentication

Our Azure Container Registry is:

```text
acrgitopslab11068.azurecr.io
```

The Ubuntu VM Managed Identity was assigned:

```text
AcrPush
```

at the ACR resource scope.

Verify the Docker/ACR authentication:

```bash
az acr login --name acrgitopslab11068
```

Expected:

```text
Login Succeeded
```

Important:

`az acr login` succeeding proves that the VM can authenticate to the registry.

---

# 11. Push Image to ACR

Push the image:

```bash
docker push acrgitopslab11068.azurecr.io/aks-demo:v1
```

Successful output contains something similar to:

```text
Layer already exists
...
v1: digest: sha256:...
```

In our lab the final image digest was:

```text
sha256:9344041685f4753f003f5a37dda65b0e133d04180148d8dc8823b28e7b820302
```

The image is now stored in ACR:

```text
ACR
└── aks-demo
    └── v1
```

---

# 12. Understanding `Layer already exists`

During the push we saw:

```text
Layer already exists
```

This is not an error.

Docker images consist of layers.

For example:

```text
nginx base layers
       +
application layer
       ↓
Docker image
```

If a layer already exists in ACR, Docker does not upload it again.

This saves:

- Time
- Network bandwidth
- Storage

The important success indicator is the final digest:

```text
v1: digest: sha256:...
```

---

# 13. ACR Permissions

There are two important permissions in this architecture.

### Build machine

The Ubuntu VM needs:

```text
AcrPush
```

because it performs:

```text
docker push
```

### AKS

AKS needs:

```text
AcrPull
```

because AKS must perform:

```text
docker pull
```

Conceptually:

```text
Ubuntu VM
    |
    | AcrPush
    ↓
   ACR
    ↑
    | AcrPull
    |
AKS Kubelet Identity
```

Remember:

```text
AcrPush → push image into ACR

AcrPull → pull image from ACR
```

---

# 14. Why `az aks update --attach-acr` Caused Permission Problems

We initially tried:

```bash
az aks update \
  --resource-group rg-gitops-aks \
  --name aks-gitops-lab \
  --attach-acr $ACR_NAME
```

The command failed because the Ubuntu VM's Managed Identity did not have permission to read/manage the AKS resource.

The error included:

```text
Microsoft.ContainerService/managedClusters/read
```

The VM identity was:

```text
Client ID:
e15570e1-8ac0-411b-90d6-4684bfb0fbf6

Object ID:
0c0f1bad-5673-4175-83b9-c1686e86e5ef
```

This taught an important lesson:

> A VM having `AcrPush` permission does not mean the VM can manage an AKS cluster.

Azure permissions are resource/action specific.

---

# 15. AKS Credentials and Permissions

We also tried:

```bash
az aks get-credentials \
  --resource-group rg-gitops-aks \
  --name aks-gitops-lab
```

The VM identity received:

```text
Microsoft.ContainerService/managedClusters/listClusterUserCredential/action
```

authorization failure.

This means:

```text
VM Managed Identity
       |
       | AcrPush
       ↓
      ACR

but

VM Managed Identity
       |
       | ❌ AKS credential permission
       ↓
      AKS
```

Therefore, an identity that can push images to ACR does not automatically have Kubernetes access.

---

# 16. Remove the Previous Manual Deployment

Before Lesson 05, the namespace contained the manually deployed Nginx application from Lesson 03:

```text
Deployment:
nginx

Pods:
3

Service:
nginx
```

We removed only the application resources:

```bash
kubectl delete deployment nginx -n gitops-demo
```

```bash
kubectl delete service nginx -n gitops-demo
```

Verify:

```bash
kubectl get all -n gitops-demo
```

We keep the namespace:

```text
gitops-demo
```

because it will be reused for this lesson and later GitOps lessons.

---

# 17. AKS Application Deployment

After the image is available in ACR, we deploy it to AKS.

Application architecture:

```text
ACR
 |
 | aks-demo:v1
 ↓
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
Service
 |
 ↓
Azure Load Balancer
 |
 ↓
Browser
```

---

# 18. Deployment Manifest

Create:

```text
deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: aks-demo
  namespace: gitops-demo
spec:
  replicas: 2

  selector:
    matchLabels:
      app: aks-demo

  template:
    metadata:
      labels:
        app: aks-demo

    spec:
      containers:
        - name: aks-demo
          image: acrgitopslab11068.azurecr.io/aks-demo:v1

          ports:
            - containerPort: 80
```

The important part is:

```yaml
image: acrgitopslab11068.azurecr.io/aks-demo:v1
```

This tells Kubernetes exactly which image to run.

---

# 19. Apply Deployment

```bash
kubectl apply -f deployment.yaml
```

Check the Deployment:

```bash
kubectl get deployment -n gitops-demo
```

Expected:

```text
NAME       READY   UP-TO-DATE   AVAILABLE
aks-demo   2/2     2            2
```

Check Pods:

```bash
kubectl get pods -n gitops-demo -o wide
```

Expected:

```text
NAME                        READY   STATUS    RESTARTS
aks-demo-xxxxxxxxxx-xxxxx   1/1     Running   0
aks-demo-xxxxxxxxxx-xxxxx   1/1     Running   0
```

---

# 20. What Kubernetes Does Behind the Scenes

When we run:

```bash
kubectl apply -f deployment.yaml
```

the flow is:

```text
kubectl
   ↓
Kubernetes API Server
   ↓
Deployment
   ↓
ReplicaSet
   ↓
Pod
   ↓
Kubelet
   ↓
Container Runtime
   ↓
Pull image from ACR
   ↓
Run container
```

The Deployment does not directly run the container.

Instead:

```text
Deployment
    ↓
ReplicaSet
    ↓
Pods
```

The Pod contains the application container.

---

# 21. Image Pull Flow

When a Pod needs:

```text
acrgitopslab11068.azurecr.io/aks-demo:v1
```

AKS must retrieve the image from ACR.

Conceptually:

```text
AKS Node
   |
   | authenticate
   ↓
AKS Kubelet Identity
   |
   | AcrPull
   ↓
Azure Container Registry
   |
   | image
   ↓
aks-demo:v1
   |
   ↓
Container Runtime
   |
   ↓
Pod
```

If the AKS identity does not have `AcrPull`, the Pod can fail with:

```text
ErrImagePull
```

or:

```text
ImagePullBackOff
```

---

# 22. Troubleshooting ImagePullBackOff

If the Pod shows:

```text
ImagePullBackOff
```

do not immediately restart the Pod.

First inspect it:

```bash
kubectl get pods -n gitops-demo
```

Then:

```bash
kubectl describe pod <POD_NAME> -n gitops-demo
```

Look at:

```text
Events:
```

Common causes:

```text
Wrong image name
Wrong image tag
ACR authentication/permission issue
Image does not exist
Registry access problem
```

Verify the image exists in ACR:

```bash
az acr repository show-tags \
  --name acrgitopslab11068 \
  --repository aks-demo \
  --output table
```

Expected:

```text
v1
```

---

# 23. Application Logs

Once the Pod is running:

```bash
kubectl logs <POD_NAME> -n gitops-demo
```

For Nginx, logs may show HTTP requests.

Remember:

```text
kubectl logs
```

shows application/container logs.

While:

```text
kubectl describe pod
```

is particularly useful for Kubernetes events and configuration problems.

---

# 24. Create a Service

The Pod IP is not intended to be the stable application endpoint.

Create:

```text
service.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: aks-demo
  namespace: gitops-demo
spec:
  selector:
    app: aks-demo

  ports:
    - port: 80
      targetPort: 80

  type: LoadBalancer
```

---

# 25. Apply the Service

```bash
kubectl apply -f service.yaml
```

Check:

```bash
kubectl get svc -n gitops-demo
```

Initially you may see:

```text
NAME       TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)
aks-demo   LoadBalancer   10.x.x.x       <pending>     80:xxxxx/TCP
```

Wait for Azure to provision the external load balancer.

Run again:

```bash
kubectl get svc -n gitops-demo
```

Eventually:

```text
NAME       TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)
aks-demo   LoadBalancer   10.x.x.x       <PUBLIC-IP>      80:xxxxx/TCP
```

---

# 26. Access the Application

Use the external IP:

```text
http://<EXTERNAL-IP>
```

The browser should display:

```text
Hello from AKS!

Application deployed using Docker, ACR and AKS.
```

The complete request path is:

```text
Browser
   ↓
Azure Public IP
   ↓
Azure Load Balancer
   ↓
Kubernetes Service
   ↓
Service selector
   ↓
aks-demo Pods
   ↓
Nginx
   ↓
index.html
```

---

# 27. Service Selector

The Service contains:

```yaml
selector:
  app: aks-demo
```

The Deployment creates Pods with:

```yaml
labels:
  app: aks-demo
```

Therefore Kubernetes connects them:

```text
Service
 selector:
 app=aks-demo
      ↓
Pods
 label:
 app=aks-demo
```

This is extremely important.

If the labels and selector don't match, the Service won't have usable endpoints.

Check:

```bash
kubectl get endpoints -n gitops-demo
```

You should see Pod endpoints.

---

# 28. Useful Verification Commands

### Check Deployment

```bash
kubectl get deployment -n gitops-demo
```

### Check ReplicaSet

```bash
kubectl get replicaset -n gitops-demo
```

### Check Pods

```bash
kubectl get pods -n gitops-demo -o wide
```

### Check Service

```bash
kubectl get svc -n gitops-demo
```

### Check Service endpoints

```bash
kubectl get endpoints -n gitops-demo
```

### Check Events

```bash
kubectl get events -n gitops-demo
```

### Describe a Pod

```bash
kubectl describe pod <POD_NAME> -n gitops-demo
```

### View logs

```bash
kubectl logs <POD_NAME> -n gitops-demo
```

---

# 29. Complete Troubleshooting Flow

When an application is not working, use this order:

```text
1. Deployment
       ↓
2. Pods
       ↓
3. Pod Events
       ↓
4. Container Logs
       ↓
5. Service
       ↓
6. Endpoints
       ↓
7. External Load Balancer
```

Commands:

```bash
kubectl get deployment -n gitops-demo
```

```bash
kubectl get pods -n gitops-demo
```

```bash
kubectl describe pod <POD_NAME> -n gitops-demo
```

```bash
kubectl logs <POD_NAME> -n gitops-demo
```

```bash
kubectl get svc -n gitops-demo
```

```bash
kubectl get endpoints -n gitops-demo
```

```bash
kubectl get events -n gitops-demo
```

---

# 30. Important Difference: ACR vs AKS

This is an important interview topic.

### Azure Container Registry

ACR is a **container image registry**.

It stores:

```text
Images
Tags
Image layers
Manifests
```

Example:

```text
acrgitopslab11068.azurecr.io/aks-demo:v1
```

### AKS

AKS is the **Kubernetes platform** that runs the application.

It manages:

```text
Pods
Deployments
ReplicaSets
Services
Nodes
Scheduling
Networking
```

Therefore:

```text
ACR = Store the image

AKS = Run the image
```

---

# 31. Important Difference: Deployment vs Service

### Deployment

Deployment manages application Pods.

Example:

```yaml
replicas: 2
```

It ensures two Pods should exist.

```text
Deployment
    ↓
ReplicaSet
    ↓
2 Pods
```

### Service

Service provides a stable network endpoint for the Pods.

```text
Service
    ↓
Pods
```

A Service also provides a stable virtual IP even though individual Pods can be replaced.

---

# 32. Why We Don't Use Pod IP Directly

Pod IPs are dynamic.

Example:

```text
Pod 1 → 10.244.x.x
Pod 2 → 10.244.x.x
```

If a Pod is deleted and recreated, its IP can change.

Therefore:

```text
Browser
   ↓
Service
   ↓
Pods
```

rather than:

```text
Browser
   ↓
Pod IP
```

The Service provides the stable abstraction.

---

# 33. Image Tagging

We used:

```text
v1
```

instead of:

```text
latest
```

Example:

```text
acrgitopslab11068.azurecr.io/aks-demo:v1
```

Explicit version tags make deployments easier to understand and troubleshoot.

Later, we will study a proper image tagging strategy in:

```text
Lesson 17 — Image Tagging Strategy
```

---

# 34. End-to-End Architecture

The complete Lesson 05 architecture is:

```text
                 Developer
                     |
                     |
                Dockerfile
                     |
                     ↓
              Ubuntu VM
             Docker Engine
                     |
                     | docker build
                     ↓
                Docker Image
                     |
                     | docker push
                     ↓
        ┌──────────────────────────┐
        │ Azure Container Registry │
        │                          │
        │ aks-demo:v1              │
        └────────────┬─────────────┘
                     |
                     | AcrPull
                     ↓
             AKS Kubelet Identity
                     |
                     ↓
                 AKS Node
                     |
                     ↓
                Deployment
                     |
                     ↓
                 ReplicaSet
                     |
             ┌───────┴───────┐
             ↓               ↓
           Pod 1            Pod 2
             |               |
             └───────┬───────┘
                     ↓
                  Service
                     |
                     ↓
              Azure Load Balancer
                     |
                     ↓
                  Browser
```

---

# 35. Permissions Architecture

There are two important identity paths:

```text
                ACR
             /       \
            /         \
       AcrPush       AcrPull
          ↑             ↑
          |             |
    Ubuntu VM          AKS
    Identity           Kubelet
```

### Ubuntu VM

```text
Managed Identity
        ↓
     AcrPush
        ↓
       ACR
```

Purpose:

```text
docker push
```

### AKS

```text
Kubelet Identity
        ↓
     AcrPull
        ↓
       ACR
```

Purpose:

```text
Pod image pull
```

This separation follows the principle of giving an identity only the permissions it needs.

---

# 36. Problems We Encountered During This Lesson

## Problem 1 — Docker daemon unavailable

Command:

```bash
docker images
```

Error:

```text
Cannot connect to the Docker daemon
```

Cause:

Cloud Shell did not provide a running Docker daemon.

Solution:

Use an Azure Ubuntu VM with Docker installed.

---

## Problem 2 — ACR Tasks blocked

Command:

```bash
az acr build \
  --registry $ACR_NAME \
  --image aks-demo:v1 \
  .
```

Error:

```text
TasksOperationsNotAllowed
```

Cause:

ACR Tasks were not permitted for this subscription/registry.

Solution:

Build the image using Docker on the Ubuntu VM.

---

## Problem 3 — Azure CLI personal login blocked

Command:

```bash
az login
```

Error:

```text
AADSTS530035:
Access has been blocked by security defaults.
```

Solution:

Use the Azure VM's Managed Identity:

```bash
az login --identity
```

---

## Problem 4 — VM couldn't manage AKS

Command:

```bash
az aks update --attach-acr ...
```

Error:

```text
Microsoft.ContainerService/managedClusters/read
```

Cause:

The VM identity had ACR permissions but did not have AKS management permissions.

Lesson:

```text
AcrPush
```

does not grant:

```text
AKS management permissions
```

---

## Problem 5 — AKS credential access denied

Command:

```bash
az aks get-credentials ...
```

Error:

```text
Microsoft.ContainerService/managedClusters/listClusterUserCredential/action
```

Cause:

The VM identity did not have the required AKS credential permission.

Lesson:

Azure RBAC permissions are specific to resources and actions.

---

# 37. Interview Questions

## Q1. Explain your container deployment architecture.

Answer:

> I build the application into a Docker image and push the image to Azure Container Registry. AKS pulls the image from ACR and runs it through a Kubernetes Deployment. The Deployment creates and manages Pods through a ReplicaSet. A Kubernetes Service provides stable access to the Pods, and for external access I use a LoadBalancer Service backed by Azure Load Balancing.

---

## Q2. What is ACR?

Answer:

> Azure Container Registry is Microsoft's private container registry service. It stores container images, tags, manifests and image layers. AKS can pull container images from ACR.

---

## Q3. Why do we need ACR?

Answer:

> Kubernetes needs access to a container image. Instead of depending on a public registry, an organization can store private application images in ACR and allow AKS to pull them using Azure identity and RBAC.

---

## Q4. What permission does the build server need?

Answer:

> The build server or VM that pushes images needs `AcrPush`.

---

## Q5. What permission does AKS need?

Answer:

> The AKS kubelet identity needs `AcrPull` to pull private container images from ACR.

---

## Q6. What happens when a Pod shows ImagePullBackOff?

Answer:

> I first check the Pod events using `kubectl describe pod`. I verify the image name and tag, confirm that the image exists in ACR, and check whether the AKS identity has permission to pull the image.

---

## Q7. Why use Deployment?

Answer:

> A Deployment provides declarative management of application Pods. It creates a ReplicaSet and ensures the desired number of Pods are running. It also supports rolling updates and rollback.

---

## Q8. Why use Service?

Answer:

> Pods are ephemeral and their IP addresses can change. A Service provides a stable network endpoint and selects Pods using labels.

---

## Q9. What is the difference between AcrPush and AcrPull?

Answer:

```text
AcrPush → Push images to ACR

AcrPull → Pull images from ACR
```

The build environment normally needs `AcrPush`, while the AKS runtime identity needs `AcrPull`.

---

## Q10. Why did we use an Ubuntu VM instead of Cloud Shell for Docker?

Answer:

> Cloud Shell did not provide a usable Docker daemon for our build workflow, and ACR Tasks were also not permitted in the subscription. Therefore, I used an Azure Ubuntu VM with Docker installed to build the image and push it to ACR.

---

# 38. Lesson 05 Checklist

### Docker

- [x] Understand Dockerfile
- [x] Build Docker image
- [x] Tag image
- [x] Understand image layers

### Azure Container Registry

- [x] Create ACR
- [x] Understand ACR
- [x] Authenticate to ACR
- [x] Push image to ACR
- [x] Understand `AcrPush`

### Azure Identity

- [x] Understand Managed Identity
- [x] Enable VM Managed Identity
- [x] Authenticate Azure CLI using Managed Identity
- [x] Understand Azure RBAC
- [x] Understand `AcrPush` vs `AcrPull`

### AKS

- [ ] Give AKS Kubelet Identity `AcrPull`
- [ ] Deploy application
- [ ] Verify Pods
- [ ] Create Service
- [ ] Get external IP
- [ ] Access application from browser

### Troubleshooting

- [x] Docker daemon issue
- [x] ACR Tasks restriction
- [x] Azure CLI authentication issue
- [x] Azure RBAC issue
- [x] AKS credential permission issue
- [ ] ImagePullBackOff
- [ ] Service endpoint troubleshooting

---

# 39. Final Mental Model

Remember this simple technical flow:

```text
Docker
   ↓
Build image
   ↓
ACR
   ↓
Store image
   ↓
AKS
   ↓
Pull image
   ↓
Deployment
   ↓
ReplicaSet
   ↓
Pods
   ↓
Service
   ↓
Application access
```

And remember the two permissions:

```text
Build machine → AcrPush → ACR

AKS Kubelet   → AcrPull → ACR
```

The most important Lesson 05 statement:

> **Docker builds the image, ACR stores the image, AKS pulls and runs the image, Deployment manages the Pods, and Service provides stable network access.**

---

# Lesson 05 Complete

Next:

```text
Lesson 06 — What is Argo CD?
```

We will take the application that we manually deployed to AKS and introduce the GitOps model:

```text
GitHub
   ↓
Argo CD
   ↓
AKS
```

This is where the previous manual deployment approach starts changing into **GitOps-based deployment**.
