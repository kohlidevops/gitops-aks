# Lesson 07 — Install Argo CD on AKS

> **GitOps Learning Path — Azure AKS + GitHub + Argo CD**

---

## 1. Lesson Goal

In Lesson 06, we learned what Argo CD is and how its components work.

In this lesson, we actually install Argo CD into our existing AKS cluster.

Our environment:

```text
Azure Resource Group: rg-gitops-aks
AKS Cluster:          aks-gitops-lab
Kubernetes Version:   1.35.7
Application Namespace: gitops-demo
Argo CD Namespace:     argocd
```

We are **reusing the existing AKS cluster**.

We are not creating another AKS cluster.

---

# 2. Where Does Argo CD Run?

Argo CD itself runs inside Kubernetes.

After installation, our AKS cluster will look approximately like:

```text
AKS Cluster
│
├── kube-system
│
├── gitops-demo
│   └── Application workloads
│
└── argocd
    ├── argocd-server
    ├── argocd-repo-server
    ├── argocd-application-controller
    ├── argocd-redis
    └── Other Argo CD components
```

The important concept is:

> **Argo CD is itself a Kubernetes application running inside AKS.**

---

# 3. Argo CD Components

The important components we will see are:

```text
Argo CD
│
├── argocd-server
│
├── argocd-repo-server
│
├── argocd-application-controller
│
└── argocd-redis
```

---

## 3.1 argocd-server

Provides:

```text
Argo CD UI
Argo CD API
CLI access
```

Think:

> **argocd-server = Front door of Argo CD**

---

## 3.2 argocd-repo-server

Responsible for:

```text
Git repository access
Manifest retrieval
Manifest generation/rendering
```

Later we will use:

```text
YAML
Helm
Kustomize
```

Think:

> **repo-server = Git/manifests side of Argo CD**

---

## 3.3 argocd-application-controller

This is one of the most important Argo CD components.

It continuously compares:

```text
Desired State
      vs
Actual State
```

and performs reconciliation.

Think:

> **Application Controller = GitOps reconciliation brain**

---

## 3.4 argocd-redis

Redis is used internally by Argo CD for caching and internal data.

It is not our application database.

Think:

> **Redis = Argo CD internal cache**

---

# 4. Lesson 06 Architecture vs Lesson 07

Before installing Argo CD:

```text
GitHub
   │
   │ Desired State
   ▼
   ?
   │
   ▼
  AKS
```

After Lesson 07:

```text
GitHub
   │
   ▼
┌──────────────────────────┐
│         Argo CD          │
│                          │
│ API Server               │
│ Repository Server        │
│ Application Controller   │
│ Redis                    │
└────────────┬─────────────┘
             │
             ▼
      Kubernetes API
             │
             ▼
            AKS
```

---

# 5. Step 1 — Verify AKS Access

We are running `kubectl` from our Azure Ubuntu VM:

```text
test-machine
```

First check the current Kubernetes context:

```bash
kubectl config current-context
```

Then:

```bash
kubectl get nodes
```

Expected:

```text
NAME                                STATUS   ROLES    AGE   VERSION
aks-nodepool1-xxxx-vmss000000       Ready    <none>   ...   v1.35.7
```

The important part is:

```text
STATUS = Ready
```

---

# 6. Step 2 — Check Whether argocd Namespace Exists

Run:

```bash
kubectl get namespace argocd
```

If this is a fresh installation, you may get:

```text
Error from server (NotFound): namespaces "argocd" not found
```

That is expected.

---

# 7. Step 3 — Create the argocd Namespace

Run:

```bash
kubectl create namespace argocd
```

Expected:

```text
namespace/argocd created
```

Verify:

```bash
kubectl get namespaces
```

You should see:

```text
argocd
```

---

# 8. Why Do We Use a Separate Namespace?

We keep Argo CD components separate from our application.

```text
AKS
│
├── argocd
│   └── Argo CD components
│
└── gitops-demo
    └── Our application
```

This provides better:

```text
Organization
Isolation
Resource management
RBAC management
Troubleshooting
```

---

# 9. Step 4 — Install Argo CD

Run:

```bash
kubectl apply \
  -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

This downloads the official Argo CD installation manifest and applies the Kubernetes resources into the `argocd` namespace.

You may see many resources being created:

```text
customresourcedefinition...
serviceaccount...
role...
rolebinding...
service...
deployment...
statefulset...
```

This is expected.

Argo CD is not a single Pod.

It consists of multiple Kubernetes resources and components.

---

# 10. Understand the Installation Command

```bash
kubectl apply
```

Means:

> Create or update Kubernetes resources from the manifest.

---

```bash
-n argocd
```

Means:

> Use the `argocd` namespace.

---

```bash
-f
```

Means:

> Read the Kubernetes manifest from the specified location.

---

The URL:

```text
https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

contains the Argo CD installation manifest.

So the flow is:

```text
Argo CD Installation Manifest
             │
             ▼
          kubectl
             │
             ▼
     Kubernetes API
             │
             ▼
            AKS
             │
             ▼
       Argo CD Components
```

---

# 11. Step 5 — Check Argo CD Pods

Run:

```bash
kubectl get pods -n argocd
```

Immediately after installation, some Pods may be:

```text
Pending
ContainerCreating
Running
```

For example:

```text
NAME                                  READY   STATUS
argocd-server-xxxxx                  0/1     ContainerCreating
argocd-repo-server-xxxxx             0/1     ContainerCreating
argocd-redis-xxxxx                   0/1     ContainerCreating
argocd-application-controller-xxxxx  0/1     Running
```

Wait for the components to become ready.

Run:

```bash
kubectl get pods -n argocd
```

Again.

Healthy Pods should eventually show:

```text
READY   STATUS
1/1     Running
```

The exact Pod names and complete component list can vary by Argo CD release.

---

# 12. Running vs Ready

This is an important Kubernetes concept.

Suppose:

```text
argocd-server-xxxxx   0/1   Running
```

`Running` means the container has started.

But:

```text
0/1
```

means:

> The Pod has one container, but it is not Ready.

A healthy Pod normally shows:

```text
1/1 Running
```

This distinction is useful during Kubernetes troubleshooting and interviews.

---

# 13. Watch Pod Startup

Run:

```bash
kubectl get pods -n argocd -w
```

The `-w` means:

```text
watch
```

You can observe the transition:

```text
Pending
   ↓
ContainerCreating
   ↓
Running
```

Stop watching:

```text
Ctrl + C
```

---

# 14. Step 6 — Check Deployments

Run:

```bash
kubectl get deployments -n argocd
```

You can also run:

```bash
kubectl get deployments -n argocd -o wide
```

This helps us see which Argo CD components are deployed as Kubernetes Deployments.

---

# 15. Step 7 — Check Services

Run:

```bash
kubectl get svc -n argocd
```

You should see services such as:

```text
argocd-server
argocd-repo-server
argocd-redis
```

The exact list can vary depending on the Argo CD version.

---

# 16. Why Does Argo CD Need Services?

Pods are temporary.

Their IP addresses can change.

A Kubernetes Service provides a stable way to communicate with the Pods.

For example:

```text
argocd-server Service
        │
        ▼
argocd-server Pod
```

So clients don't need to know the individual Pod IP.

---

# 17. Step 8 — Understand argocd-server

The main component we need for UI and CLI access is:

```text
argocd-server
```

Conceptually:

```text
Client
   │
   ▼
argocd-server Service
   │
   ▼
argocd-server Pod
   │
   ▼
Argo CD API/UI
```

---

# 18. Step 9 — Port Forwarding

Our `kubectl` is running on:

```text
Azure Ubuntu VM
```

not on our local Windows machine.

Therefore, when we run:

```bash
kubectl port-forward
```

the port is opened on the **Ubuntu VM**.

By default, Kubernetes binds port-forwarding to:

```text
127.0.0.1
```

That means only the Ubuntu VM itself can access it.

For our lab, we want the Azure VM's network interface to listen on port `8080`.

---

# 19. Start Port Forwarding on 0.0.0.0

First stop any existing port-forward:

```text
Ctrl + C
```

Then run:

```bash
kubectl port-forward \
  --address 0.0.0.0 \
  svc/argocd-server \
  -n argocd \
  8080:443
```

Expected output will be similar to:

```text
Forwarding from 0.0.0.0:8080 -> 8080
Forwarding from [::]:8080 -> 8080
```

The exact output may show the target port as `8080` because the Argo CD server Service maps its HTTPS service port to the server container's port.

---

# 20. What Does 0.0.0.0 Mean?

Our previous configuration:

```text
127.0.0.1:8080
```

means:

```text
Only this Ubuntu VM
        │
        ▼
127.0.0.1:8080
        │
        ▼
Argo CD
```

After using:

```text
--address 0.0.0.0
```

we get:

```text
0.0.0.0:8080
        │
        ▼
Azure Ubuntu VM
        │
        ▼
Argo CD
```

The VM can now accept traffic arriving on its network interfaces, subject to Azure NSG/firewall rules.

---

# 21. Our Actual Lab Architecture

This is important for our environment.

```text
                  Internet
                     │
                     │ TCP 8080
                     ▼
          ┌─────────────────────┐
          │ Azure Ubuntu VM     │
          │ test-machine        │
          │                     │
          │ Public IP           │
          │       │             │
          │       ▼             │
          │  0.0.0.0:8080       │
          │       │             │
          │       ▼             │
          │ kubectl             │
          │ port-forward        │
          └─────────┬───────────┘
                    │
                    ▼
             argocd-server
                    │
                    ▼
               Argo CD UI
```

So we are **not** doing:

```text
Windows localhost
       ↓
Argo CD
```

We are doing:

```text
Windows Browser
       ↓
Azure VM Public IP:8080
       ↓
Ubuntu VM 0.0.0.0:8080
       ↓
kubectl port-forward
       ↓
Argo CD
```

---

# 22. Azure NSG Requirement

`kubectl port-forward --address 0.0.0.0` only makes the Ubuntu VM listen on the network interface.

Azure can still block the connection through the VM's:

```text
Network Security Group (NSG)
```

Therefore, TCP port:

```text
8080
```

must be allowed by the VM's network security rules.

For a lab, preferably restrict the source to your own public IP rather than:

```text
0.0.0.0/0
```

because exposing the Argo CD login endpoint to the entire internet is unnecessary.

---

# 23. Verify Port 8080 on Ubuntu

On the Ubuntu VM:

```bash
ss -lntp | grep 8080
```

Expected:

```text
LISTEN ... 0.0.0.0:8080 ...
```

This confirms that the VM is listening on all IPv4 interfaces for port `8080`.

---

# 24. Access Argo CD UI

Since the port-forward is running on the Azure Ubuntu VM, use the **Ubuntu VM's Public IP** from your browser.

Example:

```text
https://<UBUNTU_VM_PUBLIC_IP>:8080
```

Do not use:

```text
https://localhost:8080
```

on your Windows machine.

Because:

```text
localhost
```

would refer to your Windows machine.

Our actual path is:

```text
https://<VM_PUBLIC_IP>:8080
       ↓
Azure Ubuntu VM
       ↓
kubectl port-forward
       ↓
Argo CD
```

---

# 25. HTTPS Certificate Warning

When opening:

```text
https://<VM_PUBLIC_IP>:8080
```

your browser may display a certificate warning.

This can happen because the Argo CD server is using a certificate that your browser does not trust for this lab endpoint.

For the learning environment, you can proceed through the browser warning if appropriate.

In production, we would configure proper TLS and certificate management.

---

# 26. Step 10 — Get Initial Admin Password

Argo CD creates an initial administrator account.

Username:

```text
admin
```

The initial password is stored in a Kubernetes Secret.

Check:

```bash
kubectl get secret argocd-initial-admin-secret -n argocd
```

Retrieve the password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

The command returns the initial password.

Do not put this password into GitHub.

---

# 27. Why Is the Password Stored in a Kubernetes Secret?

Credentials should not normally be hard-coded directly into:

```text
Deployment YAML
```

or:

```text
Git repository
```

Kubernetes Secrets provide a mechanism for storing sensitive values.

Later we will study:

```text
Secrets
Azure Key Vault
External Secrets
Argo CD Security
```

---

# 28. Step 11 — Argo CD CLI

Argo CD also provides a command-line interface:

```text
argocd
```

The CLI communicates with:

```text
argocd-server
```

Conceptually:

```text
Ubuntu Terminal
      │
      │ argocd CLI
      ▼
argocd-server
      │
      ▼
   Argo CD
```

Examples that we will use in later lessons:

```bash
argocd version
```

```bash
argocd app list
```

```bash
argocd app get <application-name>
```

```bash
argocd app sync <application-name>
```

For Lesson 07, our main focus is installation and connectivity.

We will use Application commands more heavily in Lesson 08.

---

# 29. Check Argo CD Resources

Run:

```bash
kubectl get all -n argocd
```

This is a useful command to get an overall view.

Also run:

```bash
kubectl get pods -n argocd
```

```bash
kubectl get deployments -n argocd
```

```bash
kubectl get svc -n argocd
```

---

# 30. Troubleshooting

If a Pod isn't starting:

```bash
kubectl get pods -n argocd
```

Identify the problematic Pod.

Then:

```bash
kubectl describe pod <POD_NAME> -n argocd
```

Look at:

```text
Events
```

Then check logs:

```bash
kubectl logs <POD_NAME> -n argocd
```

Check namespace events:

```bash
kubectl get events -n argocd --sort-by=.lastTimestamp
```

---

# 31. Kubernetes Troubleshooting Pattern

Remember this pattern:

```text
kubectl get
      ↓
kubectl describe
      ↓
kubectl logs
      ↓
kubectl get events
```

Example:

```bash
kubectl get pods -n argocd
```

If something is wrong:

```bash
kubectl describe pod <POD_NAME> -n argocd
```

Then:

```bash
kubectl logs <POD_NAME> -n argocd
```

Then:

```bash
kubectl get events -n argocd --sort-by=.lastTimestamp
```

This pattern is useful for many Kubernetes troubleshooting scenarios.

---

# 32. Important: Do Not Create an Argo CD Application Yet

For Lesson 07, don't do:

```text
❌ Connect GitHub repository
❌ Create Argo CD Application
❌ Sync application
❌ Enable automatic sync
❌ Deploy gitops-nginx through Argo CD
```

Those belong to the upcoming lessons.

Our progression is:

```text
Lesson 06
Understand Argo CD
       ↓
Lesson 07
Install Argo CD
       ↓
Lesson 08
First Argo CD Application
       ↓
Lesson 09
Automatic Sync
```

---

# 33. Complete Lesson 07 Architecture

Our actual lab architecture:

```text
                         GitHub
                            │
                            │
                            ▼
                  ┌─────────────────────┐
                  │       Argo CD       │
                  │                     │
                  │ argocd-server       │
                  │ argocd-repo-server  │
                  │ application-controller
                  │ argocd-redis        │
                  └──────────┬──────────┘
                             │
                             ▼
                     Kubernetes API
                             │
                             ▼
                            AKS
                             │
              ┌──────────────┴──────────────┐
              │                             │
              ▼                             ▼
        argocd namespace              gitops-demo
              │                             │
              │                             │
        Argo CD itself                 Our application
```

UI access:

```text
Windows Browser
      │
      │ https://<VM_PUBLIC_IP>:8080
      ▼
Azure Ubuntu VM
      │
      │ 0.0.0.0:8080
      ▼
kubectl port-forward
      │
      ▼
argocd-server Service
      │
      ▼
Argo CD
```

---

# 34. Interview Questions

## Q1. How do you install Argo CD on AKS?

**Answer:**

> "I first verify access to the AKS cluster using kubectl. Then I create a dedicated `argocd` namespace and apply the official Argo CD installation manifests. After installation, I verify the Pods, Deployments and Services. For a lab environment, I can access the Argo CD server using kubectl port-forward. In production, I would normally expose it through an appropriate ingress or load-balancing solution with proper TLS, authentication and RBAC."

---

## Q2. Where does Argo CD run?

**Answer:**

> "Argo CD runs as Kubernetes workloads inside the Kubernetes cluster. Components such as the API server, repository server and application controller run as Pods, normally in a dedicated `argocd` namespace."

---

## Q3. What does argocd-server do?

**Answer:**

> "`argocd-server` provides the Argo CD API and user interface. The Argo CD CLI and UI communicate with this server."

---

## Q4. What does argocd-repo-server do?

**Answer:**

> "The repository server communicates with Git repositories, retrieves application manifests and generates the desired manifests that Argo CD needs."

---

## Q5. What does the Application Controller do?

**Answer:**

> "The Application Controller continuously compares the desired state with the actual Kubernetes state and performs reconciliation when differences are detected."

---

## Q6. Why does Argo CD use Redis?

**Answer:**

> "Argo CD uses Redis internally for caching and internal data. It is not the application's database."

---

## Q7. Why use a separate `argocd` namespace?

**Answer:**

> "It separates Argo CD's own components from application workloads and makes resource management, RBAC and troubleshooting easier."

---

## Q8. What is port forwarding?

**Answer:**

> "Port forwarding creates a temporary connection from a local or network interface port to a Kubernetes Service or Pod. In our lab, kubectl runs on an Azure Ubuntu VM and forwards port 8080 on the VM to the Argo CD server inside AKS."

---

## Q9. Why did we use `--address 0.0.0.0`?

**Answer:**

> "By default, kubectl port-forward listens on localhost, or 127.0.0.1. Since our kubectl command is running on an Azure Ubuntu VM and we want to access Argo CD through the VM's public IP, we bind the port-forward to `0.0.0.0`. Azure NSG rules still control whether external traffic can reach the port."

---

# 35. Lesson 07 Practice

Run these commands in order.

### 1. Verify context

```bash
kubectl config current-context
```

### 2. Verify AKS

```bash
kubectl get nodes
```

### 3. Create namespace

```bash
kubectl create namespace argocd
```

### 4. Install Argo CD

```bash
kubectl apply \
  -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 5. Check Pods

```bash
kubectl get pods -n argocd
```

### 6. Watch Pods

```bash
kubectl get pods -n argocd -w
```

### 7. Check Deployments

```bash
kubectl get deployments -n argocd
```

### 8. Check Services

```bash
kubectl get svc -n argocd
```

### 9. Check all resources

```bash
kubectl get all -n argocd
```

### 10. Get initial password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

### 11. Start port forwarding on the Ubuntu VM

```bash
kubectl port-forward \
  --address 0.0.0.0 \
  svc/argocd-server \
  -n argocd \
  8080:443
```

### 12. Verify the port

Open another Ubuntu terminal and run:

```bash
ss -lntp | grep 8080
```

Expected:

```text
0.0.0.0:8080
```

### 13. Access the UI

From your Windows browser:

```text
https://<UBUNTU_VM_PUBLIC_IP>:8080
```

Login:

```text
Username: admin
Password: <initial Argo CD password>
```

---

# 36. Lesson 07 Completion Checklist

## Concepts

- [ ] Understand where Argo CD runs
- [ ] Understand `argocd-server`
- [ ] Understand `argocd-repo-server`
- [ ] Understand `argocd-application-controller`
- [ ] Understand `argocd-redis`
- [ ] Understand why Argo CD uses a separate namespace
- [ ] Understand Kubernetes Services
- [ ] Understand port forwarding
- [ ] Understand `127.0.0.1` vs `0.0.0.0`
- [ ] Understand how our Azure VM exposes the Argo CD UI

## Hands-on

- [ ] Verify AKS access
- [ ] Create `argocd` namespace
- [ ] Install Argo CD
- [ ] Verify Argo CD Pods
- [ ] Verify Argo CD Deployments
- [ ] Verify Argo CD Services
- [ ] Retrieve initial admin password
- [ ] Start port forwarding on Ubuntu VM
- [ ] Verify `0.0.0.0:8080`
- [ ] Access Argo CD through VM Public IP
- [ ] Login to Argo CD UI

---

# 37. Final Mental Model

Remember this:

```text
                     GitHub
                        │
                        ▼
                 Desired State
                        │
                        ▼
                ┌──────────────┐
                │   Argo CD    │
                │              │
                │ API Server   │
                │ Repo Server  │
                │ Controller   │
                │ Redis        │
                └──────┬───────┘
                       │
                       ▼
                Kubernetes API
                       │
                       ▼
                      AKS
```

And for our lab UI access:

```text
Windows Browser
      │
      │ https://<VM_PUBLIC_IP>:8080
      ▼
Azure Ubuntu VM
      │
      │ 0.0.0.0:8080
      ▼
kubectl port-forward
      │
      ▼
argocd-server
      │
      ▼
   Argo CD UI
```

The key sentence:

> **Argo CD runs inside AKS. Its server provides the UI/API, its repository server works with Git, and its Application Controller performs GitOps reconciliation. In our lab, we access the Argo CD server through a port-forward running on the Azure Ubuntu VM.**

---
