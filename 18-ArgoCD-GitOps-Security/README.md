# Lesson 18 — Argo CD Security

## Objective

In this lesson, we will understand and practice the security model around Argo CD, Kubernetes, Git repositories, secrets, and Azure identities.

We will use our existing:

```text
AKS:
aks-gitops-lab

Resource Group:
rg-gitops-aks

ACR:
acrgitopslab11068

GitHub Repository:
gitops-kustomize-aks

Argo CD Applications:
gitops-nginx-dev
gitops-nginx-qa
gitops-nginx-prod
```

No new AKS cluster, ACR, VM, or other Azure resources are required.

---

# 1. Security Mental Model

Our GitOps architecture currently looks like:

```text
                    GitHub
                       │
                       │ Desired State
                       ↓
                   Argo CD
                       │
                       │ Kubernetes API
                       ↓
                      AKS
                       │
                       ↓
                  Application
```

Security adds several authorization layers:

```text
GitHub
  │
  │ Repository Credentials
  ↓
Argo CD
  │
  │ Argo CD RBAC
  ↓
Argo CD Projects
  │
  │ Kubernetes permissions
  ↓
AKS
  │
  │ Workload Identity
  ↓
Azure Resources
```

The main security topics are:

```text
1. Argo CD RBAC
2. Argo CD Projects
3. Repository credentials
4. Kubernetes permissions
5. Secrets
6. Azure Managed Identity
7. Least privilege
```

---

# 2. Argo CD RBAC

RBAC means:

```text
Role-Based Access Control
```

The question Argo CD RBAC answers is:

> What is this Argo CD user allowed to do?

For example:

```text
Developer
   ↓
Can view DEV
Can sync DEV
Cannot sync PROD
Cannot administer Argo CD
```

while:

```text
Platform Admin
   ↓
Can manage Applications
Can manage Projects
Can manage Argo CD
```

Conceptually:

```text
User
 ↓
Argo CD RBAC
 ↓
Allowed Argo CD operation
```

---

# 3. Argo CD RBAC vs Kubernetes RBAC

This is one of the most important concepts.

There are two different RBAC systems.

## Argo CD RBAC

Controls what a user can do inside Argo CD.

```text
User
 ↓
Argo CD
 ↓
Can view application?
Can sync application?
Can delete application?
Can modify project?
```

## Kubernetes RBAC

Controls what an identity can do against the Kubernetes API.

```text
Identity
 ↓
Kubernetes API
 ↓
Can create Deployment?
Can update Service?
Can read Secret?
Can delete Namespace?
```

Therefore:

```text
Argo CD RBAC
      ≠
Kubernetes RBAC
```

They work at different layers.

---

# 4. Argo CD Projects

An Argo CD Project provides boundaries around Applications.

A Project can control:

```text
Which repositories?
        ↓
Which clusters?
        ↓
Which namespaces?
        ↓
Which Kubernetes resources?
```

Example:

```text
Project: gitops-dev-project

Repository:
    gitops-kustomize-aks

Destination:
    AKS

Namespace:
    gitops-dev
```

Conceptually:

```text
Application
     ↓
Argo CD Project
     ↓
Allowed deployment boundary
```

---

# 5. Why Argo CD Projects Matter

Imagine someone creates an Application that attempts to deploy to:

```yaml
destination:
  namespace: kube-system
```

instead of:

```yaml
destination:
  namespace: gitops-dev
```

A properly configured Project can restrict where that Application is allowed to deploy.

Therefore:

```text
Project
   ↓
Deployment boundary
```

Projects can help prevent an Application from using an unauthorized:

- Git repository
- Cluster
- Namespace
- Resource type

---

# 6. Repository Credentials

Argo CD needs access to Git repositories.

Our architecture is:

```text
Argo CD
   ↓
GitHub
   ↓
gitops-kustomize-aks
```

For private repositories, Argo CD needs repository authentication.

Possible mechanisms include:

```text
SSH credentials
HTTPS credentials
GitHub token
Other supported authentication methods
```

Security principle:

> Argo CD should have access only to the repositories it actually needs.

---

# 7. Repository Credentials vs Application Credentials

These are different.

Argo CD may have:

```text
Argo CD
   ↓
GitHub repository credential
```

That does NOT mean:

```text
Application Pod
   ↓
GitHub credential
```

The application should not automatically receive Argo CD's repository credentials.

Think of them separately:

```text
Argo CD
 └── Git repository credentials
```

versus:

```text
Application
 └── Application secrets
```

---

# 8. Kubernetes Permissions

Argo CD contains Kubernetes components such as:

```text
argocd-server
argocd-repo-server
argocd-application-controller
```

The Application Controller is especially important because it reconciles Applications with Kubernetes.

Conceptually:

```text
Argo CD Application Controller
             ↓
       Kubernetes API
             ↓
     create/update/delete
             ↓
        Kubernetes
```

Therefore we need to understand:

> What Kubernetes permissions does the Argo CD controller have?

---

# 9. Why Excessive Kubernetes Permissions Are Dangerous

Imagine an identity can:

```text
create
update
delete
```

across:

```text
Deployments
Services
Secrets
Namespaces
ClusterRoles
ClusterRoleBindings
```

That is powerful access.

If that identity is compromised, the potential blast radius is large.

Therefore:

```text
Permissions
    ↓
as narrow as practical
```

This is the principle of:

```text
Least Privilege
```

---

# 10. Namespaced vs Cluster-Scoped Resources

Some Kubernetes resources are namespaced:

```text
Deployment
Service
ConfigMap
Secret
Pod
```

For example:

```text
namespace: gitops-dev
```

Other resources are cluster-scoped:

```text
Namespace
ClusterRole
ClusterRoleBinding
CustomResourceDefinition
```

Therefore:

```text
Namespaced permission
        ≠
Cluster-wide permission
```

Cluster-scoped permissions generally have a larger blast radius.

---

# 11. Kubernetes ServiceAccounts

Argo CD components run using Kubernetes ServiceAccounts.

For example:

```text
argocd-application-controller
```

Conceptually:

```text
Argo CD Application Controller
             ↓
      ServiceAccount
             ↓
       Kubernetes RBAC
             ↓
       Kubernetes API
```

This is the identity we will investigate during the lab.

---

# 12. Secrets

Kubernetes Secrets may contain:

```text
Database passwords
API keys
Tokens
Certificates
Credentials
```

Example:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: database-secret
type: Opaque
stringData:
  username: appuser
  password: example-password
```

Putting production credentials directly into Git is dangerous.

Bad:

```text
GitHub
   ↓
Git repository
   ↓
Production password
```

Remember:

```text
Git = Desired State
```

Git is not automatically a secure secret vault.

---

# 13. GitOps Secret Problem

Our GitOps architecture is:

```text
Git
 ↓
Argo CD
 ↓
Kubernetes
```

But we don't want:

```text
GitHub
   ↓
Plaintext production password
```

Production GitOps commonly separates:

```text
Application configuration
```

from:

```text
Secret management
```

Possible technologies include:

```text
Azure Key Vault
External Secrets
Secrets Store CSI Driver
Sealed Secrets
SOPS
```

We will focus on Azure-oriented approaches in later lessons.

---

# 14. Azure Managed Identity

Azure Managed Identity allows workloads to access Azure resources without storing a client secret in the workload.

Instead of:

```text
Application
   ↓
Client ID + Client Secret
   ↓
Azure
```

we can use:

```text
Application
   ↓
Workload Identity
   ↓
Azure Managed Identity
   ↓
Azure Resource
```

This reduces the need to store long-lived Azure credentials in applications.

---

# 15. Important: Argo CD Does Not Automatically Become an Azure Identity

Installing Argo CD on AKS does not automatically create:

```text
Argo CD
   ↓
Azure Managed Identity
```

An Azure identity mechanism must be deliberately configured.

For example:

```text
Kubernetes ServiceAccount
          ↓
Azure Workload Identity
          ↓
Azure Managed Identity
          ↓
Azure Resource
```

This is a separate configuration from installing Argo CD.

---

# 16. Azure Key Vault Example

Suppose our application needs:

```text
DB_PASSWORD
```

stored in:

```text
Azure Key Vault
```

A possible architecture is:

```text
Azure Key Vault
       ↑
       │
Managed Identity
       ↑
       │
Workload Identity
       ↑
       │
Application Pod
```

The workload receives access according to its Azure permissions.

The important principle is:

```text
Secret
 ↓
Key Vault

Identity
 ↓
Only required secret access
```

---

# 17. ACR Identity

Our application also needs to pull:

```text
acrgitopslab11068.azurecr.io/aks-demo
```

Conceptually:

```text
AKS
 ↓
Azure Identity
 ↓
ACR
 ↓
Container Image
```

This is different from:

```text
Argo CD
 ↓
GitHub
```

and different from:

```text
GitHub Actions
 ↓
Azure
```

Keep these identity paths separate.

---

# 18. Three Important Identity Paths

## GitHub Actions → Azure

Our CI pipeline uses GitHub OIDC:

```text
GitHub Actions
      ↓
OIDC
      ↓
Azure
```

This is used by our CI workflow.

---

## Argo CD → GitHub

Argo CD uses configured repository authentication:

```text
Argo CD
      ↓
Repository Credentials
      ↓
GitHub
```

This is used to read the GitOps repository.

---

## Kubernetes → Azure

A workload can use:

```text
Pod
 ↓
Kubernetes ServiceAccount
 ↓
Azure Workload Identity
 ↓
Managed Identity
 ↓
Azure Resource
```

These are three different authentication paths.

---

# 19. Least Privilege

Least privilege means:

> Give an identity only the permissions required to perform its job.

Example:

Bad:

```text
Developer
 ↓
All Argo Applications
 ↓
DEV + QA + PROD
 ↓
Sync + Delete + Admin
```

More restricted:

```text
Developer
 ↓
DEV Project
 ↓
View + Sync
```

Another example:

Bad:

```text
Application
 ↓
Azure Subscription Owner
```

More restricted:

```text
Application
 ↓
Managed Identity
 ↓
Key Vault
 ↓
Required secret access
```

The exact permissions depend on the application's requirements.

---

# 20. Security Architecture

Our target mental model is:

```text
                 USER
                   │
                   ↓
             Argo CD RBAC
                   │
                   ↓
             Argo CD Project
                   │
          ┌────────┴────────┐
          ↓                 ↓
     Git Repository      Destination
          │                 │
          ↓                 ↓
       GitHub               AKS
                            │
                     Kubernetes RBAC
                            │
                            ↓
                         Workload
                            │
                    Azure Workload Identity
                            │
                            ↓
                    Managed Identity
                       /          \
                      ↓            ↓
                 Key Vault        ACR
```

---

# 21. Practice — Complete Security Lab

## Lab Objective

By the end of the lab, you should be able to:

```text
Inspect
   ↓
Understand
   ↓
Configure
   ↓
Test
   ↓
Verify
```

the major Argo CD security layers.

We will use the existing environment.

No new Azure infrastructure is required.

---

# Exercise 01 — Security Baseline

First inspect the current Argo CD installation.

Run:

```bash
kubectl get pods -n argocd
```

Then:

```bash
kubectl get serviceaccounts -n argocd
```

Then:

```bash
kubectl get appprojects -n argocd
```

Then:

```bash
kubectl get applications -n argocd
```

### Goal

Build this mental picture:

```text
AKS
 └── argocd
      ├── argocd-server
      ├── argocd-repo-server
      ├── argocd-application-controller
      └── ServiceAccounts
```

Do not modify anything yet.

---

# Exercise 02 — Understand Argo CD ServiceAccounts

Run:

```bash
kubectl get pods -n argocd \
  -o custom-columns=NAME:.metadata.name,SERVICE_ACCOUNT:.spec.serviceAccountName
```

Then:

```bash
kubectl get serviceaccount \
  argocd-application-controller \
  -n argocd \
  -o yaml
```

### Goal

Understand:

```text
Argo CD Application Controller
             ↓
      Kubernetes ServiceAccount
             ↓
       Kubernetes permissions
```

---

# Exercise 03 — Kubernetes RBAC Investigation

Test whether the Argo CD controller can perform different operations.

### Can it read Deployments?

```bash
kubectl auth can-i \
  get deployments \
  --as=system:serviceaccount:argocd:argocd-application-controller
```

### Can it create Deployments?

```bash
kubectl auth can-i \
  create deployments \
  --as=system:serviceaccount:argocd:argocd-application-controller
```

### Can it delete Deployments?

```bash
kubectl auth can-i \
  delete deployments \
  --as=system:serviceaccount:argocd:argocd-application-controller
```

### Can it read Secrets?

```bash
kubectl auth can-i \
  get secrets \
  --as=system:serviceaccount:argocd:argocd-application-controller
```

### Can it create ClusterRoles?

```bash
kubectl auth can-i \
  create clusterroles \
  --as=system:serviceaccount:argocd:argocd-application-controller
```

### Can it delete Namespaces?

```bash
kubectl auth can-i \
  delete namespaces \
  --as=system:serviceaccount:argocd:argocd-application-controller
```

Create a small table from the results:

```text
Operation                    Result
------------------------------------------------
Get Deployment               ?
Create Deployment            ?
Delete Deployment            ?
Get Secret                   ?
Create ClusterRole           ?
Delete Namespace             ?
```

### Goal

Understand the actual permissions rather than assuming them.

---

# Exercise 04 — Find Where Kubernetes Permissions Come From

Inspect the ClusterRole:

```bash
kubectl get clusterrole \
  argocd-application-controller \
  -o yaml
```

Look at:

```yaml
rules:
```

Then inspect the binding:

```bash
kubectl get clusterrolebinding \
  -o yaml | grep -B 10 -A 10 \
  argocd-application-controller
```

### Goal

Understand:

```text
ServiceAccount
      ↓
ClusterRoleBinding
      ↓
ClusterRole
      ↓
Permissions
```

This is Kubernetes RBAC.

---

# Exercise 05 — Inspect Argo CD RBAC

Run:

```bash
kubectl get configmap argocd-rbac-cm \
  -n argocd \
  -o yaml
```

Look for:

```text
policy.csv
```

Also inspect:

```bash
kubectl get configmap argocd-cm \
  -n argocd \
  -o yaml
```

### Goal

Understand that Argo CD RBAC controls:

```text
User
 ↓
Argo CD
 ↓
Allowed Argo operation
```

while Kubernetes RBAC controls:

```text
Identity
 ↓
Kubernetes API
 ↓
Allowed Kubernetes operation
```

---

# Exercise 06 — Inspect Argo CD Projects

Run:

```bash
kubectl get appprojects -n argocd
```

Inspect the default project:

```bash
kubectl get appproject default \
  -n argocd \
  -o yaml
```

Look for:

```yaml
spec:
  sourceRepos:
  destinations:
  clusterResourceWhitelist:
  namespaceResourceWhitelist:
```

### Goal

Understand:

```text
Project
 ├── Which Git repositories?
 ├── Which clusters?
 ├── Which namespaces?
 └── Which resources?
```

---

# Exercise 07 — Create a DEV Security Project

Create:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: gitops-dev-project
  namespace: argocd
spec:
  description: GitOps DEV project

  sourceRepos:
    - "https://github.com/kohlidevops@100069489/gitops-kustomize-aks.git"

  destinations:
    - namespace: gitops-dev
      server: https://kubernetes.default.svc

  namespaceResourceWhitelist:
    - group: "*"
      kind: "*"

  clusterResourceWhitelist: []
```

Save it as:

```text
gitops-dev-project.yaml
```

Apply:

```bash
kubectl apply -f gitops-dev-project.yaml
```

Verify:

```bash
kubectl get appproject \
  gitops-dev-project \
  -n argocd
```

### Goal

Understand:

```text
Project
   ↓
Allowed Repository
   ↓
Allowed Destination
```

---

# Exercise 08 — Connect DEV Application to the Project

Our DEV Application currently uses a Project.

Check it:

```bash
kubectl get application gitops-nginx-dev \
  -n argocd \
  -o jsonpath='{.spec.project}'
```

Change the DEV Application to:

```yaml
spec:
  project: gitops-dev-project
```

Then verify:

```bash
kubectl get application gitops-nginx-dev \
  -n argocd \
  -o jsonpath='{.spec.project}'
```

Expected:

```text
gitops-dev-project
```

Then:

```bash
kubectl get application gitops-nginx-dev \
  -n argocd
```

### Goal

Understand:

```text
Application
     ↓
Argo CD Project
     ↓
Deployment Boundary
```

---

# Exercise 09 — Test Destination Restriction

Our Project allows:

```text
namespace: gitops-dev
```

Now understand what happens if an Application attempts:

```yaml
destination:
  namespace: gitops-prod
```

The Project does not authorize that destination.

The important concept:

```text
gitops-dev-project
        ↓
gitops-dev
```

does not mean:

```text
gitops-dev-project
        ↓
any namespace
```

### Goal

Understand destination restrictions.

---

# Exercise 10 — Test Repository Restriction

Our Project allows only the configured GitOps repository.

Conceptually:

```text
Project
  ↓
sourceRepos
  ↓
Approved Repository
```

If an Application attempts to use another repository:

```text
some-other-repository
```

the Project should reject the source because it is not authorized.

### Goal

Understand source repository boundaries.

---

# Exercise 11 — Understand Resource Restrictions

Consider:

```text
Deployment
Service
ConfigMap
Secret
Namespace
ClusterRole
ClusterRoleBinding
```

These don't all have the same security impact.

For example:

```text
Deployment
   ↓
Application-level resource
```

while:

```text
ClusterRoleBinding
   ↓
Cluster-wide authorization
```

A secure Project can restrict which resources it is allowed to manage.

### Goal

Understand:

```text
Namespaced resources
        vs
Cluster-scoped resources
```

---

# Exercise 12 — Design Developer and Admin Roles

Design two conceptual roles.

## Developer

```text
Can:
  View DEV
  Sync DEV

Cannot:
  Sync PROD
  Delete PROD
  Modify Projects
  Administer Argo CD
```

## Platform Admin

```text
Can:
  Manage Applications
  Manage Projects
  Manage Argo CD configuration
```

The model is:

```text
User
 ↓
Argo CD RBAC
 ↓
Role
 ↓
Allowed Actions
```

---

# Exercise 13 — Test Argo CD RBAC

After configuring the appropriate test identities/roles, test:

```text
Developer → DEV → Sync
Developer → PROD → Sync
Developer → Project → Modify
Developer → Argo configuration → Modify
```

The intended security model is:

```text
Developer
   │
   ├── DEV sync       → Allowed
   ├── PROD sync      → Denied
   ├── Project admin  → Denied
   └── Argo admin     → Denied
```

### Goal

Understand:

```text
Authentication
      ↓
Authorization
      ↓
Allowed / Denied
```

---

# Exercise 14 — Repository Credentials

Inspect repository-related Secret metadata without exposing credentials:

```bash
kubectl get secrets -n argocd \
  -o custom-columns=NAME:.metadata.name,TYPE:.type
```

Do not decode or share credential values.

Understand:

```text
Argo CD
   ↓
Repository Credential
   ↓
GitHub
   ↓
GitOps Repository
```

This is different from:

```text
Application
   ↓
Application Secret
```

---

# Exercise 15 — Secret Security

Create a temporary learning-only Secret in DEV:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: lesson18-test-secret
  namespace: gitops-dev
type: Opaque
stringData:
  username: testuser
  password: temporary-password
```

Save as:

```text
lesson18-test-secret.yaml
```

Apply:

```bash
kubectl apply -f lesson18-test-secret.yaml
```

Test whether the Argo Application Controller can read Secrets:

```bash
kubectl auth can-i \
  get secrets \
  -n gitops-dev \
  --as=system:serviceaccount:argocd:argocd-application-controller
```

After testing, delete the temporary Secret:

```bash
kubectl delete secret lesson18-test-secret \
  -n gitops-dev
```

### Goal

Understand the security question:

```text
Can Argo CD read Kubernetes Secrets?
```

---

# Exercise 16 — Why Plaintext Secrets Shouldn't Be in Git

Bad:

```yaml
stringData:
  password: MyProductionPassword
```

inside:

```text
GitHub
   ↓
Git repository
```

Better:

```text
Azure Key Vault
       ↓
Workload Identity
       ↓
Application
```

Other approaches include:

```text
External Secrets
Secrets Store CSI Driver
Sealed Secrets
SOPS
```

The important principle:

```text
Git
 ≠
Secret Vault
```

---

# Exercise 17 — Inspect Azure Managed Identity

Run:

```bash
az aks show \
  --resource-group rg-gitops-aks \
  --name aks-gitops-lab \
  --query identity
```

Then:

```bash
az aks show \
  --resource-group rg-gitops-aks \
  --name aks-gitops-lab \
  --query identityProfile
```

### Goal

Understand the Azure identity associated with AKS.

---

# Exercise 18 — Separate the Three Identity Paths

## GitHub Actions → Azure

```text
GitHub Actions
      ↓
OIDC
      ↓
Azure
```

Used by our CI pipeline.

## Argo CD → GitHub

```text
Argo CD
      ↓
Repository Credentials
      ↓
GitHub
```

Used to read the GitOps repository.

## Kubernetes → Azure

```text
Pod
 ↓
Kubernetes ServiceAccount
 ↓
Azure Workload Identity
 ↓
Managed Identity
 ↓
Azure Resource
```

Used when Kubernetes workloads need Azure access.

### Goal

Be able to explain why these are separate identity paths.

---

# Exercise 19 — Least Privilege Investigation

Test several permissions:

```bash
kubectl auth can-i \
  get pods \
  --as=system:serviceaccount:argocd:argocd-application-controller
```

```bash
kubectl auth can-i \
  delete pods \
  --as=system:serviceaccount:argocd:argocd-application-controller
```

```bash
kubectl auth can-i \
  get secrets \
  --as=system:serviceaccount:argocd:argocd-application-controller
```

```bash
kubectl auth can-i \
  create clusterrolebindings \
  --as=system:serviceaccount:argocd:argocd-application-controller
```

```bash
kubectl auth can-i \
  delete namespaces \
  --as=system:serviceaccount:argocd:argocd-application-controller
```

Record:

```text
Permission                    Result
------------------------------------------------
get pods                      ?
delete pods                   ?
get secrets                   ?
create clusterrolebindings    ?
delete namespaces             ?
```

Then ask:

> Does Argo really need every one of these permissions?

This is the least-privilege mindset.

---

# Exercise 20 — Security Failure Scenario

Imagine someone configures:

```yaml
destination:
  namespace: kube-system
```

Ask:

```text
Should a normal application Project be allowed
to deploy there?
```

Then compare:

```text
cluster-admin
```

with:

```text
application-specific permissions
```

The purpose is to understand the **blast radius** of excessive permissions.

---

# Exercise 21 — End-to-End Security Model

At the end of the lab, you should be able to draw:

```text
                         USER
                          │
                          ↓
                    Argo CD RBAC
                          │
                          ↓
                    Argo CD Project
                    /          \
                   /            \
          Source Repository    Destination
                 │                 │
                 ↓                 ↓
              GitHub              AKS
                                    │
                              Kubernetes RBAC
                                    │
                                    ↓
                                  Pod
                                    │
                            Workload Identity
                                    │
                                    ↓
                            Managed Identity
                                    │
                    ┌───────────────┴───────────────┐
                    ↓                               ↓
                Key Vault                           ACR
```

---

# Exercise 22 — Final Security Scenarios

## Scenario 1

A developer can log into Argo CD.

Question:

> Does that automatically mean they can deploy to PROD?

Explain the role of:

```text
Argo CD RBAC
+
Project
+
Application
```

---

## Scenario 2

Argo CD can access GitHub.

Question:

> Does that mean application Pods can access GitHub?

Explain:

```text
Argo repository credentials
          ≠
Application credentials
```

---

## Scenario 3

A Pod needs Azure Key Vault.

Question:

> Should we put an Azure client secret inside the Pod?

Explain the workload identity approach.

---

## Scenario 4

Argo can create Deployments.

Question:

> Does that mean it should also be able to create ClusterRoleBindings?

Discuss:

```text
Namespaced resources
        vs
Cluster-scoped resources
```

and least privilege.

---

## Scenario 5

A developer can access DEV.

Question:

> How can we prevent that developer from syncing PROD?

Discuss:

```text
Argo CD RBAC
+
Argo CD Projects
+
Application boundaries
```

---

# 23. Final Security Architecture

The complete mental model:

```text
                         USER
                           │
                           ↓
                     Argo CD RBAC
                           │
                           ↓
                     Argo CD Project
                           │
                   ┌───────┴───────┐
                   ↓               ↓
             Git Repository    Destination
                   │               │
                   ↓               ↓
                GitHub             AKS
                                   │
                            Kubernetes RBAC
                                   │
                                   ↓
                                Workload
                                   │
                            Workload Identity
                                   │
                                   ↓
                           Managed Identity
                              /          \
                             ↓            ↓
                        Key Vault         ACR
```

The security principle throughout the architecture is:

```text
                 LEAST PRIVILEGE

       Give only what is required
                    ↓
       Nothing unnecessarily more
```

---

# 24. Lesson 18 Completion Checklist

```text
[ ] Understand Argo CD RBAC

[ ] Understand Kubernetes RBAC

[ ] Understand the difference between them

[ ] Understand Argo CD Projects

[ ] Understand repository restrictions

[ ] Understand destination restrictions

[ ] Understand resource restrictions

[ ] Identify Argo CD ServiceAccounts

[ ] Inspect Argo CD controller permissions

[ ] Use kubectl auth can-i

[ ] Understand repository credentials

[ ] Understand why Git credentials should not reach application Pods

[ ] Understand Kubernetes Secrets

[ ] Understand why plaintext production secrets should not be committed to Git

[ ] Understand Azure Managed Identity

[ ] Understand Azure Workload Identity

[ ] Understand GitHub OIDC

[ ] Distinguish GitHub OIDC from Azure Workload Identity

[ ] Understand ACR identity

[ ] Understand Key Vault identity

[ ] Analyze excessive permissions

[ ] Understand least privilege

[ ] Test allowed operations

[ ] Test denied operations

[ ] Understand the complete Argo CD security architecture

[ ] Explain the security model in an interview
```

---

# 25. Interview Questions

## Q1. What is Argo CD RBAC?

> Argo CD RBAC controls what authenticated users or groups can do within Argo CD, such as viewing, syncing, or deleting Applications.

## Q2. What is an Argo CD Project?

> An Argo CD Project provides boundaries for Applications by restricting source repositories, destination clusters/namespaces, and resource types.

## Q3. What's the difference between Argo CD RBAC and Kubernetes RBAC?

> Argo CD RBAC controls user actions within Argo CD. Kubernetes RBAC controls permissions against the Kubernetes API.

## Q4. Why shouldn't production secrets be stored as plaintext in Git?

> Anyone with access to the repository could potentially access the credentials. A dedicated secret-management solution provides better control over secret access.

## Q5. What is least privilege?

> Grant an identity only the permissions required for its specific task and avoid unnecessary access.

## Q6. Does installing Argo CD on AKS automatically give it Azure Managed Identity?

> No. An Azure identity mechanism such as Azure Workload Identity must be explicitly configured for Kubernetes workloads to access Azure resources as an Azure identity.

## Q7. What identity does Argo CD use to access Git?

> Argo CD uses configured repository authentication, such as supported HTTPS or SSH-based credentials. This is separate from the Azure identity used by Kubernetes workloads.

## Q8. Why shouldn't Argo CD have unrestricted cluster-admin permissions?

> Because compromising Argo CD or its controller could then provide a much larger level of access to the cluster. Restricting permissions reduces the potential blast radius.

---

# 26. Final Mental Model

Remember this:

```text
                 WHO?
                  │
                  ↓
             Argo CD RBAC
                  │
                  ↓
                WHAT?
                  │
                  ↓
            Argo CD Project
                  │
                  ↓
               WHERE?
                  │
                  ↓
        Cluster / Namespace
                  │
                  ↓
               HOW?
                  │
                  ↓
         Kubernetes RBAC
                  │
                  ↓
             Azure Access
                  │
                  ↓
         Workload Identity
                  │
                  ↓
         Managed Identity
                  │
                  ↓
          Azure Resources
```

For secrets:

```text
Git
 │
 ├── Application configuration
 │
 └── Secret reference
          │
          ↓
     Secret Store
          │
          ↓
       Workload
```

---

# 27. Hands-On Execution Order

Do not execute the entire lab at once.

Use this sequence:

```text
Exercise 01
    ↓
Inspect current security
    ↓
Exercise 02
    ↓
Understand ServiceAccounts
    ↓
Exercise 03
    ↓
Test Kubernetes permissions
    ↓
Exercise 04
    ↓
Find the RBAC configuration
    ↓
Exercise 05
    ↓
Understand Argo CD RBAC
    ↓
Exercise 06
    ↓
Understand Projects
    ↓
Exercise 07
    ↓
Create DEV security Project
    ↓
Exercise 08
    ↓
Attach DEV Application
    ↓
Exercise 09–10
    ↓
Test Project restrictions
    ↓
Exercise 11–13
    ↓
Understand and test RBAC
    ↓
Exercise 14–16
    ↓
Repository credentials + Secrets
    ↓
Exercise 17–18
    ↓
Azure identities
    ↓
Exercise 19–20
    ↓
Least privilege + failure scenarios
    ↓
Exercise 21–22
    ↓
End-to-end security validation
```

**Do not modify production RBAC or production Projects blindly.** During the hands-on session, inspect first, make one controlled change, test it, and verify the result before moving to the next exercise.

## Lesson 18 Final Principle

```text
Secure GitOps is not just:

"Protect Argo CD login."

It is:

User
 ↓
Argo CD RBAC
 ↓
Project
 ↓
Repository
 ↓
Destination
 ↓
Kubernetes RBAC
 ↓
Workload Identity
 ↓
Azure Resource
```

Every layer should have only the access required for its job.
