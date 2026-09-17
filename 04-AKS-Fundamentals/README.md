# Lesson 04 — Azure AKS Fundamentals

## 🎯 Objective

In this lesson, we learn how Azure Kubernetes Service (AKS) works internally and how Kubernetes runs on Azure.

We will understand:

- What is AKS?
- AKS architecture
- Control Plane
- Worker Nodes
- Node Pools
- Azure CNI
- Azure CNI Overlay
- Networking basics
- Node IP
- Pod IP
- Service IP
- Kubernetes DNS
- Managed Identity
- Azure Container Registry (ACR)
- AKS managed resource group

For hands-on practice, we will use an **existing AKS cluster** instead of creating another cluster.

---

# 1. What is AKS?

AKS stands for:

```text
Azure Kubernetes Service
```

AKS is Microsoft's managed Kubernetes service.

With self-managed Kubernetes, we need to manage components such as:

```text
Kubernetes Control Plane
API Server
Scheduler
Controller Manager
etcd
```

With AKS:

```text
Azure
  |
  v
AKS
  |
  +--- Azure manages the Kubernetes control plane
  |
  +--- We manage workloads
       |
       +--- Pods
       +--- Deployments
       +--- Services
       +--- Nodes
       +--- Node Pools
       +--- Networking
       +--- Security
       +--- Storage
```

The key idea:

```text
AKS = Managed Kubernetes on Azure
```

---

# 2. AKS Architecture

The basic architecture is:

```text
                         Azure
                           |
                           v
                    +-------------+
                    |     AKS     |
                    +-------------+
                           |
             +-------------+-------------+
             |                           |
             v                           v
      Control Plane                 Worker Nodes
      Managed by Azure                   |
                                         |
                                  +------+------+
                                  |             |
                                  v             v
                                Node 1        Node 2
                                  |             |
                                  v             v
                                Pods          Pods
                                  |
                                  v
                             Application
```

The important separation is:

```text
Control Plane
      |
      | manages
      v
Worker Nodes
      |
      | run
      v
Pods
```

---

# 3. Control Plane

The Kubernetes Control Plane is responsible for managing the Kubernetes cluster.

Important components include:

```text
API Server
Scheduler
Controller Manager
etcd
```

In AKS, Azure manages the control plane.

We don't normally log into or manage the AKS control-plane machines directly.

---

# 4. Kubernetes API Server

The API Server is the main entry point to Kubernetes.

When we execute:

```bash
kubectl get pods
```

the flow is approximately:

```text
kubectl
   |
   v
Kubernetes API Server
   |
   v
Cluster State
```

When we execute:

```bash
kubectl apply -f deployment.yaml
```

the flow is:

```text
kubectl
   |
   v
API Server
   |
   v
Deployment
```

The API Server is therefore the central communication point for Kubernetes operations.

---

# 5. Kubernetes Scheduler

Suppose we deploy:

```yaml
spec:
  replicas: 3
```

Kubernetes needs to decide:

```text
Which worker node should run each Pod?
```

The Scheduler makes scheduling decisions based on available nodes and scheduling requirements.

Conceptually:

```text
Deployment
     |
     v
3 Pods
     |
     v
Scheduler
     |
     +------> Node 1
     |
     +------> Node 2
     |
     +------> Node 1
```

---

# 6. Controller Manager

Kubernetes controllers continuously work toward the desired state.

Example:

```text
Desired:
3 Pods

Actual:
2 Pods
```

The controller detects the difference:

```text
Desired State = 3
Actual State  = 2
```

and Kubernetes works to create the missing Pod.

This is one of the most important Kubernetes concepts:

```text
Desired State
      vs
Actual State
```

---

# 7. etcd

`etcd` is the distributed key-value store used by Kubernetes to store cluster state.

Conceptually:

```text
API Server
    |
    v
  etcd
    |
    v
Cluster State
```

In AKS, the control plane and its etcd are managed by Azure.

We therefore don't manually administer the AKS control-plane etcd.

---

# 8. Worker Nodes

Worker nodes are where application workloads actually run.

Architecture:

```text
AKS
 |
 +--- Control Plane
 |
 +--- Worker Nodes
        |
        +--- kubelet
        +--- container runtime
        +--- Pods
```

Check worker nodes:

```bash
kubectl get nodes -o wide
```

---

# 9. kubelet

`kubelet` is the Kubernetes node agent.

It runs on each worker node and communicates with the Kubernetes control plane.

Conceptually:

```text
API Server
    |
    v
 kubelet
    |
    v
Containers
```

The kubelet helps ensure that the Pods assigned to its node are running.

---

# 10. Container Runtime

Kubernetes needs a container runtime to run containers.

Our AKS cluster uses:

```text
containerd
```

Check:

```bash
kubectl get nodes -o wide
```

Look at:

```text
CONTAINER-RUNTIME
```

Example:

```text
containerd://2.3.3
```

---

# 11. Node Pools

A node pool is a group of worker nodes with a common configuration.

For example:

```text
AKS
 |
 +--- System Node Pool
 |
 +--- Application Node Pool
 |
 +--- GPU Node Pool
 |
 +--- Windows Node Pool
```

Different workloads can be placed into different node pools.

---

# 12. Node Pool vs Node

Don't confuse these two terms.

### Node Pool

A group/configuration of worker nodes.

Example:

```text
nodepool1
Count: 2
VM Size: Standard_B2s_v2
OS: Linux
```

### Node

An individual worker machine.

Example:

```text
aks-nodepool1-31816321-vmss000000
```

Relationship:

```text
AKS
 |
 +--- nodepool1
        |
        +--- Node 1
        |
        +--- Node 2
        |
        +--- Node 3
```

---

# 13. Inspect Node Pools

Run:

```bash
az aks nodepool list \
  --resource-group $RESOURCE_GROUP \
  --cluster-name $AKS_NAME \
  --output table
```

Important fields:

```text
Name
OsType
KubernetesVersion
VmSize
Count
MaxPods
ProvisioningState
Mode
```

---

# 14. System Node Pool

Our AKS cluster currently has:

```text
Mode: System
```

A System node pool is intended to run critical AKS/Kubernetes system workloads.

We can inspect these workloads using:

```bash
kubectl get pods -n kube-system -o wide
```

The architecture is:

```text
AKS
 |
 +--- System Node Pool
        |
        +--- Worker Node
               |
               +--- System Pods
```

---

# 15. Azure CNI

AKS requires networking for:

```text
Nodes
Pods
Services
```

One AKS networking option is:

```text
Azure CNI
```

Azure CNI integrates AKS networking with Azure networking.

However, there are different Azure CNI modes.

Our cluster uses:

```text
Azure CNI Overlay
```

This is important.

---

# 16. Azure CNI Overlay

Our AKS cluster reports:

```text
networkPlugin: azure
networkPluginMode: overlay
networkDataplane: azure
```

Therefore:

```text
Azure CNI
     +
Overlay Mode
```

Our Pod network is:

```text
10.244.0.0/16
```

The conceptual architecture is:

```text
Azure/VNet Network
        |
        | Node networking
        v
      Node
        |
        | Overlay networking
        v
      Pods
```

Do not simply describe this cluster as:

```text
Every Pod gets an IP directly from the Azure VNet
```

because this cluster specifically uses:

```text
Azure CNI Overlay
```

---

# 17. Check AKS Network Configuration

Run:

```bash
az aks show \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_NAME \
  --query "networkProfile" \
  --output json
```

Important fields include:

```text
networkPlugin
networkPluginMode
networkDataplane
networkPolicy
podCidr
serviceCidr
dnsServiceIp
outboundType
```

---

# 18. Our Actual AKS Network Configuration

Our cluster returned:

```text
networkPlugin: azure
networkPluginMode: overlay
networkDataplane: azure
networkPolicy: none
```

Pod CIDR:

```text
10.244.0.0/16
```

Service CIDR:

```text
10.0.0.0/16
```

DNS Service IP:

```text
10.0.0.10
```

Outbound type:

```text
loadBalancer
```

Load Balancer SKU:

```text
standard
```

---

# 19. Node IP

A worker node has an IP address.

Check:

```bash
kubectl get nodes -o wide
```

Our worker node currently has:

```text
Internal IP: 10.224.0.4
```

Conceptually:

```text
Node
 |
 +--- Node IP
      10.224.0.4
```

The Node IP represents the worker node's network address.

---

# 20. Pod IP

Pods also have IP addresses.

Check:

```bash
kubectl get pods -A -o wide
```

Look at:

```text
IP
NODE
```

With our Azure CNI Overlay configuration, Pod addresses use the Pod CIDR:

```text
10.244.0.0/16
```

Conceptually:

```text
Pod Network
10.244.0.0/16
       |
       +--- Pod 1
       |
       +--- Pod 2
       |
       +--- Pod 3
```

---

# 21. Service IP

Kubernetes Services have their own virtual IPs.

Check:

```bash
kubectl get svc -A
```

For example:

```text
NAME    TYPE        CLUSTER-IP
nginx   ClusterIP   10.0.x.x
```

Our Service CIDR is:

```text
10.0.0.0/16
```

So conceptually:

```text
Service CIDR
10.0.0.0/16
       |
       +--- Service IP
       |
       +--- Service IP
       |
       +--- Service IP
```

---

# 22. Node IP vs Pod IP vs Service IP

This is very important for interviews.

```text
Node IP
   ↓
Worker node

Pod IP
   ↓
Application Pod

Service IP
   ↓
Stable Kubernetes Service endpoint
```

Our environment:

```text
Node IP:
10.224.0.4

Pod Network:
10.244.0.0/16

Service Network:
10.0.0.0/16
```

---

# 23. Kubernetes DNS

Our DNS Service IP is:

```text
10.0.0.10
```

Kubernetes DNS provides service discovery.

Instead of hard-coding Pod IP addresses, applications can communicate using Kubernetes DNS names.

Conceptually:

```text
Application Pod
      |
      v
Kubernetes DNS
      |
      v
Service
      |
      v
Application Pods
```

This is one reason applications should normally communicate through Services rather than relying on individual Pod IPs.

---

# 24. Managed Identity

AKS needs to interact with Azure resources.

Examples:

```text
AKS
 |
 +--- Azure Load Balancer
 +--- Azure Disk
 +--- Azure Container Registry
 +--- Azure Key Vault
```

Instead of storing long-lived Azure credentials inside applications, Azure provides managed identities.

Check the AKS identity:

```bash
az aks show \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_NAME \
  --query "identity" \
  --output json
```

The important concept is:

```text
Managed Identity
       |
       v
Azure authentication
       |
       v
Azure resources
```

---

# 25. Why Managed Identity Is Important

Without managed identity, teams may be tempted to store:

```text
Client ID
Client Secret
Password
Access Key
```

inside applications or configuration.

Managed identity can reduce the need for storing long-lived Azure credentials.

Example:

```text
AKS / Azure Workload
        |
        v
Managed Identity
        |
        v
Azure Resource
```

We will study Azure identity in more detail later, especially when we reach:

```text
Lesson 19 — Secrets + Azure Key Vault
```

---

# 26. Azure Container Registry

ACR stands for:

```text
Azure Container Registry
```

It is Azure's private container registry.

Example image:

```text
myregistry.azurecr.io/myapp:1.0
```

The architecture is:

```text
Developer
    |
    v
Docker Build
    |
    v
Container Image
    |
    v
ACR
    |
    v
AKS
    |
    v
Pod
```

---

# 27. ACR in a CI/CD Pipeline

Eventually our GitOps architecture will look like:

```text
Developer
    |
    v
GitHub
    |
    v
GitHub Actions
    |
    +--- Build
    +--- Test
    +--- Security Scan
    |
    v
ACR
    |
    | Container Image
    v
Git Repository
    |
    | Desired Deployment State
    v
Argo CD
    |
    v
AKS
    |
    v
Pods
```

This creates a clean separation:

```text
GitHub Actions
      |
      | Build image
      v
     ACR
      |
      | Store image
      v
    Argo CD
      |
      | Deploy desired state
      v
     AKS
```

We will implement this later in the GitOps lessons.

---

# 28. AKS Managed Resource Group

When AKS is created, Azure creates a managed resource group for AKS infrastructure.

Our cluster has a resource group similar to:

```text
MC_rg-gitops-aks_aks-gitops-lab_centralindia
```

Conceptually:

```text
Your Resource Group
rg-gitops-aks
       |
       v
AKS
       |
       v
Managed Resource Group
MC_rg-gitops-aks_aks-gitops-lab_centralindia
```

The managed resource group can contain infrastructure such as:

```text
Virtual Machine Scale Set
Load Balancer
Public IP
Networking resources
Other AKS-managed infrastructure
```

Do not manually delete or modify AKS-managed resources.

Manage the cluster through:

```text
az aks ...
```

rather than directly modifying generated infrastructure.

---

# 29. Inspect the AKS Cluster

Set variables:

### Bash / Azure Cloud Shell

```bash
AKS_NAME="aks-gitops-lab"
RESOURCE_GROUP="rg-gitops-aks"
```

Check:

```bash
echo $AKS_NAME
echo $RESOURCE_GROUP
```

---

# 30. Check AKS

```bash
az aks list --output table
```

Expected information includes:

```text
Name
Location
ResourceGroup
KubernetesVersion
ProvisioningState
Fqdn
```

---

# 31. Check Nodes

```bash
kubectl get nodes -o wide
```

Our cluster currently has:

```text
Node:
aks-nodepool1-31816321-vmss000000

Status:
Ready

OS:
Ubuntu 24.04.4 LTS

Container Runtime:
containerd

Internal IP:
10.224.0.4
```

---

# 32. Check Node Pools

```bash
az aks nodepool list \
  --resource-group $RESOURCE_GROUP \
  --cluster-name $AKS_NAME \
  --output table
```

Our current node pool:

```text
Name:
nodepool1

OS:
Linux

VM Size:
Standard_B2s_v2

Count:
1

MaxPods:
250

Mode:
System
```

---

# 33. Check All Pods

```bash
kubectl get pods -A -o wide
```

This helps us understand:

```text
Namespace
Pod
Status
IP
Node
```

---

# 34. Check System Pods

```bash
kubectl get pods -n kube-system -o wide
```

This helps us understand what AKS/Kubernetes system workloads are running on the worker node.

---

# 35. Check Network Profile

```bash
az aks show \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_NAME \
  --query "networkProfile" \
  --output json
```

Important values from our cluster:

```text
Azure CNI:
azure

Mode:
overlay

Dataplane:
azure

Pod CIDR:
10.244.0.0/16

Service CIDR:
10.0.0.0/16

DNS Service IP:
10.0.0.10

Outbound:
loadBalancer
```

---

# 36. Check Managed Identity

```bash
az aks show \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_NAME \
  --query "identity" \
  --output json
```

Understand:

```text
AKS
 |
 v
Identity
 |
 v
Azure Resource Access
```

---

# 37. Important AKS Commands

### List AKS clusters

```bash
az aks list --output table
```

### Show AKS details

```bash
az aks show \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_NAME
```

### List node pools

```bash
az aks nodepool list \
  --resource-group $RESOURCE_GROUP \
  --cluster-name $AKS_NAME \
  --output table
```

### Get credentials

```bash
az aks get-credentials \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_NAME
```

### Kubernetes nodes

```bash
kubectl get nodes
```

### Detailed nodes

```bash
kubectl get nodes -o wide
```

### All Pods

```bash
kubectl get pods -A
```

### Pods with networking information

```bash
kubectl get pods -A -o wide
```

---

# 38. Interview Scenario

### Interviewer:

> Explain the architecture of an application running on AKS.

### Answer:

```text
AKS provides managed Kubernetes.

Azure manages the Kubernetes control plane, including the API
server, scheduler, controllers and etcd.

The workloads run on worker nodes inside node pools.

A Deployment manages the application Pods.

The ReplicaSet created by the Deployment maintains the desired
number of Pods.

A Service provides stable connectivity to the Pods.

In our AKS cluster, networking uses Azure CNI Overlay.

The node has an Azure network IP, while Pods use the configured
Pod CIDR.

Kubernetes Services use the Service CIDR and Kubernetes DNS
provides service discovery.

Container images can be stored in Azure Container Registry.

Azure managed identities can be used to authenticate workloads
to Azure resources without storing long-lived credentials.
```

---

# 39. Interview Scenario — Node Pool

### Interviewer:

> What is the difference between a node and a node pool?

Answer:

```text
A node is an individual worker machine.

A node pool is a group of nodes with a common configuration.

For example, I can have a system node pool for Kubernetes system
workloads and a separate application node pool for application
workloads.
```

---

# 40. Interview Scenario — Azure CNI Overlay

### Interviewer:

> What networking model is your AKS cluster using?

Answer:

```text
My AKS cluster uses Azure CNI with Overlay mode.

The network plugin is Azure and the network plugin mode is Overlay.

The Pod CIDR in my cluster is 10.244.0.0/16 and the Service CIDR
is 10.0.0.0/16.

Therefore, I need to distinguish the node network from the Pod
overlay network and the Kubernetes Service network.
```

---

# 41. Interview Scenario — Pod IP vs Service IP

### Interviewer:

> Why don't you directly use a Pod IP?

Answer:

```text
Pod IPs are associated with individual Pods and Pods are
ephemeral.

When Pods are recreated, their IPs can change.

A Kubernetes Service provides a stable endpoint and selects
the appropriate Pods using labels and selectors.
```

---

# 42. Important Mental Model

Remember:

```text
AKS
 |
 +--- Control Plane
 |      |
 |      +--- API Server
 |      +--- Scheduler
 |      +--- Controllers
 |      +--- etcd
 |
 +--- Node Pools
        |
        +--- Nodes
              |
              +--- kubelet
              +--- containerd
              +--- Pods
```

Networking:

```text
Azure CNI
    |
    +--- Node Network
    |
    +--- Pod Overlay Network
    |
    +--- Service Network
```

Identity:

```text
Managed Identity
       |
       v
Azure Resource Access
```

Images:

```text
Developer
    |
    v
Container Image
    |
    v
ACR
    |
    v
AKS
    |
    v
Pod
```

---

# 43. Final AKS Mental Model

The complete picture:

```text
                           AZURE
                             |
                             v
                    +------------------+
                    |       AKS        |
                    +------------------+
                             |
                +------------+------------+
                |                         |
                v                         v
        Managed Control Plane        Node Pools
                |                         |
                |                    +----+----+
                |                    |         |
                |                    v         v
                |                  Node      Node
                |                    |         |
                |                    +----+----+
                |                         |
                |                         v
                |                        Pods
                |
                +--- API Server
                +--- Scheduler
                +--- Controllers
                +--- etcd
```

Networking:

```text
Azure CNI Overlay
       |
       +--- Node Network
       |
       +--- Pod Network
       |      10.244.0.0/16
       |
       +--- Service Network
       |      10.0.0.0/16
       |
       +--- DNS
              10.0.0.10
```

Container images:

```text
GitHub Actions
      |
      v
     ACR
      |
      v
     AKS
      |
      v
     Pods
```

Identity:

```text
AKS / Workload
      |
      v
Managed Identity
      |
      v
Azure Resources
```

---

# 44. Lesson 04 Checklist

- [ ] Understand what AKS is
- [ ] Understand AKS architecture
- [ ] Understand Control Plane
- [ ] Understand API Server
- [ ] Understand Scheduler
- [ ] Understand Controller Manager
- [ ] Understand etcd
- [ ] Understand Worker Nodes
- [ ] Understand kubelet
- [ ] Understand containerd
- [ ] Understand Node Pools
- [ ] Understand System Node Pool
- [ ] Understand Azure CNI
- [ ] Understand Azure CNI Overlay
- [ ] Understand Node IP
- [ ] Understand Pod IP
- [ ] Understand Service IP
- [ ] Understand Service CIDR
- [ ] Understand Pod CIDR
- [ ] Understand Kubernetes DNS
- [ ] Understand Managed Identity
- [ ] Understand Azure Container Registry
- [ ] Understand AKS managed resource group
- [ ] Inspect your AKS cluster using Azure CLI
- [ ] Inspect nodes using kubectl
- [ ] Inspect node pools
- [ ] Inspect system Pods
- [ ] Inspect network configuration
- [ ] Inspect AKS identity
- [ ] Explain the architecture in an interview

---

# Lesson 04 Complete

The most important concepts to remember are:

```text
AKS
 ↓
Managed Kubernetes

Control Plane
 ↓
Manages the cluster

Node Pool
 ↓
Group of worker nodes

Node
 ↓
Runs Pods

Pod
 ↓
Runs containers

Azure CNI Overlay
 ↓
AKS networking

Service
 ↓
Stable application endpoint

Managed Identity
 ↓
Azure authentication

ACR
 ↓
Container image registry
```

The key architecture to remember for the next lessons is:

```text
GitHub
   |
   v
GitHub Actions
   |
   v
ACR
   |
   v
Argo CD
   |
   v
AKS
   |
   v
Node Pool
   |
   v
Node
   |
   v
Pod
   |
   v
Application
```

This is the foundation for the upcoming GitOps + Argo CD implementation.
