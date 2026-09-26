# Lesson 19 — GitOps Secrets Management with Azure Key Vault

## Objective

In this lesson, we will learn how to manage Kubernetes secrets without storing secret values directly in Git.

We will build:

```text
Git
 │
 │ ExternalSecret manifest
 ▼
Argo CD
 │
 ▼
External Secrets Operator
 │
 │ Azure Workload Identity
 ▼
Azure Managed Identity
 │
 │ Key Vault Secrets User
 ▼
Azure Key Vault
 │
 │ database-password
 ▼
Kubernetes Secret
 │
 ▼
Application
```

The important principle is:

> **Git stores the reference to the secret, not the secret value.**

---

# 1. Why Plaintext Secrets in Git Are Bad

Consider:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: database-secret
stringData:
  password: MyPassword123
```

This is dangerous because the actual password is stored in Git.

Even if someone later changes the file to:

```yaml
password: NEW_PASSWORD
```

the old password may still exist in Git history.

For example:

```text
Git Repository
│
├── Commit 1
│     password: MyPassword123
│
├── Commit 2
│     password: NewPassword456
│
└── Current
      password: NewPassword456
```

Deleting the secret from the current version does not automatically erase it from Git history.

Therefore:

```text
Git
❌ Secret values

Git
✅ Secret references / desired configuration
```

---

# 2. GitOps Does Not Mean Secrets Must Be Stored in Git

GitOps means Git contains the desired state.

But the desired state does not have to contain sensitive values.

Instead of:

```yaml
password: MyPassword123
```

Git can contain:

```yaml
remoteRef:
  key: database-password
```

The actual value remains in Azure Key Vault.

Therefore:

```text
Git
 │
 │ "I need database-password"
 ▼
ExternalSecret
 │
 ▼
External Secrets Operator
 │
 ▼
Azure Key Vault
 │
 │ actual password
 ▼
Kubernetes Secret
```

---

# 3. Components Used in This Lesson

We use the following components:

```text
Azure Key Vault
External Secrets Operator
Azure Workload Identity
User Assigned Managed Identity
Kubernetes ServiceAccount
ExternalSecret
SecretStore
Kubernetes Secret
```

Each component has a different responsibility.

---

# 4. Azure Key Vault

Azure Key Vault stores the actual secret value.

Example:

```text
Key Vault
└── kv-gitops-secrets-12345
      │
      └── database-password
             │
             └── MyPassword123
```

The value:

```text
MyPassword123
```

does NOT go into Git.

For this lab:

```text
Key Vault:
kv-gitops-secrets-12345
```

---

# 5. External Secrets Operator

External Secrets Operator (ESO) runs inside Kubernetes.

Its responsibility is to retrieve secrets from an external secret provider and create Kubernetes Secrets.

Conceptually:

```text
Azure Key Vault
      │
      │ secret value
      ▼
External Secrets Operator
      │
      ▼
Kubernetes Secret
```

ESO uses two important Kubernetes resources:

```text
SecretStore
ExternalSecret
```

---

# 6. SecretStore

`SecretStore` tells ESO:

> How and where should I retrieve the secret?

Example:

```yaml
kind: SecretStore
metadata:
  name: azure-keyvault
  namespace: gitops-dev

spec:
  provider:
    azurekv:
      authType: WorkloadIdentity
      vaultUrl: ...
      tenantId: ...
      serviceAccountRef:
        name: external-secrets-azure
```

The SecretStore contains connection/authentication configuration.

It does NOT contain the actual password.

---

# 7. ExternalSecret

`ExternalSecret` tells ESO:

> Which secret should I retrieve and what Kubernetes Secret should I create?

Example:

```yaml
kind: ExternalSecret
metadata:
  name: database-secret
  namespace: gitops-dev

spec:
  secretStoreRef:
    name: azure-keyvault
    kind: SecretStore

  target:
    name: database-secret

  data:
    - secretKey: password
      remoteRef:
        key: database-password
```

This means:

```text
Azure Key Vault:
database-password

        ↓

Kubernetes Secret:
database-secret

        ↓

Key:
password
```

---

# 8. Azure Workload Identity

We do NOT want to put an Azure client secret inside Kubernetes.

Instead, Kubernetes uses an Azure federated identity.

The authentication flow is:

```text
Kubernetes ServiceAccount
        │
        │ OIDC token
        ▼
Azure Workload Identity
        │
        │ Federated trust
        ▼
User Assigned Managed Identity
        │
        ▼
Azure Key Vault
```

This removes the need to store an Azure client secret.

---

# 9. Three Different Identity Paths

This is important because we already use multiple identities in our GitOps architecture.

## GitHub Actions

```text
GitHub Actions
      │
      │ OIDC
      ▼
Azure
```

Used for CI.

---

## Argo CD

```text
Argo CD
   │
   │ Repository credentials
   ▼
GitHub
```

Used to read Git desired state.

---

## Kubernetes Workload Identity

```text
Kubernetes ServiceAccount
      │
      │ OIDC
      ▼
Azure Managed Identity
      │
      ▼
Azure Key Vault
```

Used by workloads/operators inside AKS to access Azure resources.

Do not confuse these three identity paths.

---

# 10. Lab Environment

Existing environment:

```text
Resource Group:
rg-gitops-aks

AKS:
aks-gitops-lab

Location:
centralindia

Namespace:
gitops-dev
```

Azure Key Vault:

```text
kv-gitops-secrets-12345
```

User Assigned Managed Identity:

```text
mi-external-secrets
```

Managed Identity Client ID:

```text
246cc7ad-56b2-4bd0-8dd3-a606d3bf5e22
```

Tenant ID:

```text
dd08c73a-948e-45de-9ce4-845e0568830a
```

Kubernetes ServiceAccount:

```text
external-secrets-azure
```

---

# 11. Important Permission Model

We use different permissions for different identities.

## Your administrator/user account

Used to create/manage the teaching secret.

Example role:

```text
Key Vault Secrets Officer
```

---

## External Secrets Managed Identity

Used by ESO to read secrets.

Role:

```text
Key Vault Secrets User
```

The Managed Identity should NOT be given:

```text
Owner
Contributor
Key Vault Administrator
```

for this purpose.

The goal is least privilege:

```text
User
 │
 │ manage secrets
 ▼
Key Vault Secrets Officer

External Secrets Managed Identity
 │
 │ read secrets
 ▼
Key Vault Secrets User
```

---

# 12. Practice — Part 1
# Create a Test Secret in Key Vault

Create a temporary teaching secret:

```text
Name:
database-password

Value:
MyPassword123
```

Do not use a real production password.

Using Azure CLI:

```bash
az keyvault secret set \
  --vault-name kv-gitops-secrets-12345 \
  --name database-password \
  --value "MyPassword123"
```

If your Azure account does not have permission to create secrets, use Azure Portal or ask the Azure administrator to grant the appropriate Key Vault data-plane permission.

Verify:

```bash
az keyvault secret show \
  --vault-name kv-gitops-secrets-12345 \
  --name database-password
```

Do not paste the secret value into Git.

---

# 13. Practice — Part 2
# Verify External Secrets Operator

Check the namespace:

```bash
kubectl get namespace external-secrets
```

Check the pods:

```bash
kubectl get pods -n external-secrets
```

Check the CRDs:

```bash
kubectl get crd | grep external-secrets
```

We need:

```text
secretstores.external-secrets.io
externalsecrets.external-secrets.io
```

---

# 14. Practice — Part 3
# Verify the SecretStore API Version

Do not assume the ESO version.

Check:

```bash
kubectl get crd secretstores.external-secrets.io \
  -o jsonpath='{.spec.versions[*].name}'
```

Example:

```text
v1 v1beta1
```

Use the supported `v1` API when available.

---

# 15. Practice — Part 4
# Verify Azure Key Vault Provider Schema

Do not assume the provider field is called `azure`.

The correct ESO provider is:

```text
azurekv
```

Check the installed schema:

```bash
kubectl explain secretstore.spec.provider.azurekv --recursive
```

Important fields include:

```text
authType
vaultUrl
tenantId
serviceAccountRef
```

Supported authentication types include:

```text
ServicePrincipal
ManagedIdentity
WorkloadIdentity
```

For this lesson we use:

```text
WorkloadIdentity
```

---

# 16. Practice — Part 5
# Verify AKS OIDC

Check:

```bash
az aks show \
  --resource-group rg-gitops-aks \
  --name aks-gitops-lab \
  --query oidcIssuerProfile.issuerUrl \
  --output tsv
```

Example:

```text
https://centralindia.oic.prod-aks.azure.com/<tenant-id>/<cluster-id>/
```

The exact URL must be taken from your AKS cluster.

Do not manually invent the URL.

---

# 17. Practice — Part 6
# Verify AKS Workload Identity

Check:

```bash
az aks show \
  --resource-group rg-gitops-aks \
  --name aks-gitops-lab \
  --query "{oidc:oidcIssuerProfile.enabled,workloadIdentity:securityProfile.workloadIdentity.enabled}" \
  --output table
```

Expected:

```text
oidc    workloadIdentity
------  ----------------
True    True
```

Both are required for this implementation.

---

# 18. Practice — Part 7
# Create Kubernetes ServiceAccount

Create:

```bash
kubectl create serviceaccount external-secrets-azure \
  -n gitops-dev
```

Verify:

```bash
kubectl get serviceaccount \
  external-secrets-azure \
  -n gitops-dev
```

---

# 19. Practice — Part 8
# Configure ServiceAccount for Workload Identity

Add the Workload Identity label:

```bash
kubectl label serviceaccount external-secrets-azure \
  -n gitops-dev \
  azure.workload.identity/use=true
```

Add the Managed Identity client ID:

```bash
kubectl annotate serviceaccount external-secrets-azure \
  -n gitops-dev \
  azure.workload.identity/client-id=246cc7ad-56b2-4bd0-8dd3-a606d3bf5e22
```

Verify:

```bash
kubectl get serviceaccount \
  external-secrets-azure \
  -n gitops-dev \
  -o yaml
```

Expected:

```yaml
metadata:
  annotations:
    azure.workload.identity/client-id: 246cc7ad-56b2-4bd0-8dd3-a606d3bf5e22
  labels:
    azure.workload.identity/use: "true"
```

---

# 20. Practice — Part 9
# Create User Assigned Managed Identity

The identity used by ESO is:

```text
mi-external-secrets
```

It must exist before creating the federated credential.

Verify:

```bash
az identity show \
  --name mi-external-secrets \
  --resource-group rg-gitops-aks \
  --query "{clientId:clientId,principalId:principalId,id:id}" \
  --output table
```

Expected information:

```text
Client ID
Principal ID
Resource ID
```

---

# 21. Practice — Part 10
# Grant Key Vault Access to Managed Identity

Assign:

```text
Key Vault Secrets User
```

to:

```text
mi-external-secrets
```

at the Key Vault scope:

```text
kv-gitops-secrets-12345
```

The purpose is:

```text
mi-external-secrets
        │
        │ READ
        ▼
Azure Key Vault secrets
```

The Managed Identity should not manage or delete Key Vault resources.

---

# 22. Practice — Part 11
# Create Federated Identity Credential

The federated credential connects the Kubernetes ServiceAccount to the Azure Managed Identity.

The subject is:

```text
system:serviceaccount:gitops-dev:external-secrets-azure
```

The audience is:

```text
api://AzureADTokenExchange
```

Create the federated credential using the AKS OIDC issuer obtained earlier.

Example:

```bash
az identity federated-credential create \
  --name fic-external-secrets-azure \
  --identity-name mi-external-secrets \
  --resource-group rg-gitops-aks \
  --issuer "<AKS-OIDC-ISSUER-URL>" \
  --subject "system:serviceaccount:gitops-dev:external-secrets-azure" \
  --audiences "api://AzureADTokenExchange"
```

If your Azure identity cannot create federated credentials, create it from the Azure Portal:

```text
Managed Identities
    ↓
mi-external-secrets
    ↓
Federated credentials
    ↓
Add credential
    ↓
Kubernetes accessing Azure resources
```

Use:

```text
Issuer:
<AKS OIDC issuer>

Namespace:
gitops-dev

Service account:
external-secrets-azure
```

---

# 23. Practice — Part 12
# Create SecretStore

Now create:

```yaml
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: azure-keyvault
  namespace: gitops-dev

spec:
  provider:
    azurekv:
      authType: WorkloadIdentity

      vaultUrl: "https://kv-gitops-secrets-12345.vault.azure.net"

      tenantId: "dd08c73a-948e-45de-9ce4-845e0568830a"

      serviceAccountRef:
        name: external-secrets-azure
```

Save as:

```text
secretstore-azure.yaml
```

Apply:

```bash
kubectl apply -f secretstore-azure.yaml
```

Expected:

```text
secretstore.external-secrets.io/azure-keyvault created
```

---

# 24. Practice — Part 13
# Verify SecretStore

Run:

```bash
kubectl get secretstore azure-keyvault \
  -n gitops-dev
```

Then:

```bash
kubectl describe secretstore azure-keyvault \
  -n gitops-dev
```

Successful output should contain:

```text
Conditions:
  Message: store validated
  Reason:  Valid
  Status:  True
  Type:    Ready
```

This means:

```text
SecretStore
    │
    ├── Azure provider configuration ✅
    ├── Workload Identity             ✅
    ├── ServiceAccount                ✅
    └── Key Vault connection          ✅
```

---

# 25. Practice — Part 14
# Create ExternalSecret

Create:

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: database-secret
  namespace: gitops-dev

spec:
  refreshInterval: 1h

  secretStoreRef:
    name: azure-keyvault
    kind: SecretStore

  target:
    name: database-secret
    creationPolicy: Owner
    deletionPolicy: Retain

  data:
    - secretKey: password
      remoteRef:
        key: database-password
```

Save as:

```text
external-secret.yaml
```

Apply:

```bash
kubectl apply -f external-secret.yaml
```

Expected:

```text
externalsecret.external-secrets.io/database-secret created
```

---

# 26. Practice — Part 15
# Verify ExternalSecret

Run:

```bash
kubectl get externalsecret database-secret \
  -n gitops-dev
```

Then:

```bash
kubectl describe externalsecret database-secret \
  -n gitops-dev
```

Successful output should contain:

```text
Conditions:
  Message: secret synced
  Reason:  SecretSynced
  Status:  True
  Type:    Ready
```

You may also see:

```text
Binding:
  Name: database-secret
```

This means ESO successfully created the Kubernetes Secret.

---

# 27. Practice — Part 16
# Verify Kubernetes Secret

Run:

```bash
kubectl get secret database-secret \
  -n gitops-dev
```

Expected:

```text
NAME              TYPE     DATA   AGE
database-secret   Opaque   1      ...
```

The Kubernetes Secret now exists.

The flow is:

```text
Azure Key Vault
    │
    │ database-password
    ▼
External Secrets Operator
    │
    ▼
Kubernetes Secret
    │
    └── database-secret
          └── password
```

---

# 28. Verify the Secret Value

If you want to verify that the value matches Key Vault:

```bash
kubectl get secret database-secret \
  -n gitops-dev \
  -o jsonpath='{.data.password}' | base64 --decode
```

This prints the secret value.

Do not paste the actual password into Git, chat, screenshots, tickets, or documentation.

---

# 29. How the Complete Authentication Flow Works

This is the most important part of the lesson.

Suppose ESO needs:

```text
database-password
```

The flow is:

```text
1. ExternalSecret
        │
        │ "Get database-password"
        ▼
2. SecretStore
        │
        │ Azure Key Vault provider
        ▼
3. Kubernetes ServiceAccount
        │
        │ external-secrets-azure
        ▼
4. Azure Workload Identity
        │
        │ OIDC token
        ▼
5. Federated Identity Credential
        │
        │ trusted subject
        ▼
6. mi-external-secrets
        │
        │ Key Vault Secrets User
        ▼
7. Azure Key Vault
        │
        │ database-password
        ▼
8. External Secrets Operator
        │
        ▼
9. Kubernetes Secret
```

---

# 30. Important Difference: ExternalSecret vs SecretStore

Remember:

```text
SecretStore
=
How do I connect?
```

while:

```text
ExternalSecret
=
What secret do I want?
```

Example:

```text
SecretStore
└── Azure Key Vault
    └── Workload Identity
```

and:

```text
ExternalSecret
└── database-password
```

Together:

```text
ExternalSecret
      │
      ▼
SecretStore
      │
      ▼
Azure Key Vault
```

---

# 31. Why the SecretStore Must Exist First

If the ExternalSecret says:

```yaml
secretStoreRef:
  name: azure-keyvault
```

but there is no:

```text
SecretStore/azure-keyvault
```

ESO cannot retrieve the secret.

You will see an error similar to:

```text
could not get SecretStore "azure-keyvault",
SecretStore.external-secrets.io "azure-keyvault" not found
```

Therefore:

```text
SecretStore
      ↓
ExternalSecret
      ↓
Kubernetes Secret
```

---

# 32. Application Consuming the Secret

The application does not need to know about Azure Key Vault.

It simply consumes the Kubernetes Secret.

Example:

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: database-secret
        key: password
```

The application sees:

```text
DB_PASSWORD
```

It does not need to know:

```text
Azure Key Vault
Managed Identity
OIDC
Federated Credential
ExternalSecret
```

Those are handled by the platform.

---

# 33. Test Secret Rotation

This is an important practical exercise.

Change the secret in Azure Key Vault:

```bash
az keyvault secret set \
  --vault-name kv-gitops-secrets-12345 \
  --name database-password \
  --value "NewTemporaryPassword456"
```

Our ExternalSecret has:

```yaml
refreshInterval: 1h
```

Therefore ESO periodically checks the external secret.

For a faster lab test, temporarily use a shorter refresh interval:

```yaml
refreshInterval: 1m
```

Then wait for the next reconciliation.

Check:

```bash
kubectl get externalsecret database-secret \
  -n gitops-dev
```

Then verify the Kubernetes Secret value:

```bash
kubectl get secret database-secret \
  -n gitops-dev \
  -o jsonpath='{.data.password}' | base64 --decode
```

The Kubernetes Secret should eventually contain:

```text
NewTemporaryPassword456
```

This demonstrates:

```text
Azure Key Vault
       │
       │ changed value
       ▼
External Secrets Operator
       │
       │ reconciliation
       ▼
Kubernetes Secret
       │
       │ updated value
       ▼
Application
```

---

# 34. GitOps and Secret Rotation

Notice something important.

We did NOT change Git when rotating the password.

Git still contains:

```yaml
remoteRef:
  key: database-password
```

Only Key Vault changed:

```text
Old password
     ↓
New password
```

ESO detected the change and updated the Kubernetes Secret.

Therefore:

```text
Application secret rotation
        ↓
Azure Key Vault
        ↓
ESO reconciliation
        ↓
Kubernetes Secret
```

No Git commit is required.

---

# 35. GitOps Repository Design

A Git repository can contain:

```text
gitops-kustomize-aks/
│
├── kustomize/
│   └── gitops-nginx/
│       ├── base/
│       └── overlays/
│           └── dev/
│
└── secrets/
    └── external-secret.yaml
```

The repository can contain:

```yaml
remoteRef:
  key: database-password
```

but should NOT contain:

```yaml
password: MyPassword123
```

---

# 36. Argo CD's Role

Argo CD manages the Kubernetes desired state.

For example:

```text
Git
 │
 │ ExternalSecret YAML
 ▼
Argo CD
 │
 ▼
AKS
 │
 ▼
ExternalSecret
```

Argo CD does NOT retrieve the Key Vault secret itself.

Instead:

```text
Argo CD
    │
    │ creates ExternalSecret
    ▼
External Secrets Operator
    │
    │ retrieves secret
    ▼
Azure Key Vault
```

This separation is important.

---

# 37. Argo CD vs External Secrets Operator

```text
Argo CD
=
Git → Kubernetes desired state
```

```text
External Secrets Operator
=
External Secret Provider → Kubernetes Secret
```

Therefore:

```text
Git
 │
 ▼
Argo CD
 │
 ▼
ExternalSecret
 │
 ▼
ESO
 │
 ▼
Azure Key Vault
 │
 ▼
Kubernetes Secret
```

---

# 38. Security Model

The complete security model is:

```text
                    Git
                     │
                     │ no password
                     ▼
                  Argo CD
                     │
                     ▼
              ExternalSecret
                     │
                     ▼
          External Secrets Operator
                     │
              ServiceAccount
                     │
                     ▼
          Azure Workload Identity
                     │
                     ▼
          Federated Identity Trust
                     │
                     ▼
          mi-external-secrets
                     │
             Key Vault Secrets User
                     │
                     ▼
          Azure Key Vault
                     │
                     ▼
            Kubernetes Secret
```

There are no long-lived Azure client secrets in Git.

---

# 39. Troubleshooting Guide

## Problem 1 — SecretStore not found

Error:

```text
SecretStore "azure-keyvault" not found
```

Check:

```bash
kubectl get secretstore -A
```

Create the SecretStore in the same namespace as the ExternalSecret when using a namespaced `SecretStore`.

---

## Problem 2 — SecretStore is not Ready

Run:

```bash
kubectl describe secretstore azure-keyvault \
  -n gitops-dev
```

Check:

```text
Conditions
Events
```

Common areas to verify:

```text
vaultUrl
tenantId
authType
serviceAccountRef
Workload Identity
Managed Identity
Key Vault RBAC
```

---

## Problem 3 — ExternalSecret is not Ready

Run:

```bash
kubectl describe externalsecret database-secret \
  -n gitops-dev
```

Look at:

```text
Status
Conditions
Events
```

---

## Problem 4 — Kubernetes Secret does not exist

Run:

```bash
kubectl get externalsecret database-secret \
  -n gitops-dev
```

If:

```text
Ready=True
```

then check:

```bash
kubectl get secret database-secret \
  -n gitops-dev
```

---

## Problem 5 — Workload Identity authentication failure

Verify:

```bash
kubectl get serviceaccount external-secrets-azure \
  -n gitops-dev \
  -o yaml
```

Check:

```yaml
annotations:
  azure.workload.identity/client-id: <managed-identity-client-id>

labels:
  azure.workload.identity/use: "true"
```

Also verify:

```text
AKS OIDC issuer = enabled
AKS Workload Identity = enabled
Federated credential = exists
```

---

## Problem 6 — Key Vault access denied

Check the Managed Identity's Key Vault role.

It should have:

```text
Key Vault Secrets User
```

on:

```text
kv-gitops-secrets-12345
```

Do not immediately grant:

```text
Owner
Contributor
Key Vault Administrator
```

Use the narrowest required permission.

---

# 40. Important Lesson Learned During the Lab

Azure permissions can be different for different operations.

For example, an identity may be able to:

```text
Create Key Vault
```

but not:

```text
Set Key Vault secret
```

Or it may be able to:

```text
Create Managed Identity
```

but not:

```text
Create federated credential
```

Or it may be able to:

```text
Read AKS configuration
```

but not:

```text
Modify AKS configuration
```

Examples of separate permissions:

```text
Microsoft.KeyVault/vaults/write

Microsoft.KeyVault/vaults/secrets/setSecret/action

Microsoft.ManagedIdentity/userAssignedIdentities/write

Microsoft.ManagedIdentity/userAssignedIdentities/federatedIdentityCredentials/write

Microsoft.ContainerService/managedClusters/write
```

This is why Azure authorization errors should be read carefully rather than repeatedly trying different commands.

---

# 41. Final Architecture

Our completed Lesson 19 architecture is:

```text
                         GitHub
                            │
                            │ ExternalSecret YAML
                            ▼
                         Argo CD
                            │
                            │ desired state
                            ▼
                         AKS
                            │
                            ▼
                  ExternalSecret
                            │
                            │ references
                            ▼
                     SecretStore
                            │
                            │ Workload Identity
                            ▼
                  Kubernetes ServiceAccount
                  external-secrets-azure
                            │
                            │ OIDC
                            ▼
                 Azure Workload Identity
                            │
                            │ Federation
                            ▼
                  mi-external-secrets
                            │
                            │ Key Vault Secrets User
                            ▼
               Azure Key Vault
               kv-gitops-secrets-12345
                            │
                            │ database-password
                            ▼
                External Secrets Operator
                            │
                            ▼
                 Kubernetes Secret
                    database-secret
                            │
                            ▼
                      Application
```

---

# 42. Core Mental Model

Remember these four statements:

```text
Git
=
Desired state
```

```text
Azure Key Vault
=
Secret storage
```

```text
External Secrets Operator
=
Synchronizes external secrets into Kubernetes
```

```text
Argo CD
=
Synchronizes Git desired state into Kubernetes
```

Therefore:

```text
Git
 ↓
Argo CD
 ↓
ExternalSecret
 ↓
External Secrets Operator
 ↓
Azure Key Vault
 ↓
Kubernetes Secret
 ↓
Application
```

---

# 43. Interview Discussion

## Question 1

### Why shouldn't we store passwords directly in Git?

Because Git history can retain previous secret values, making accidental exposure difficult to eliminate.

---

## Question 2

### Does Argo CD retrieve the Key Vault secret?

No.

Argo CD applies the `ExternalSecret` resource.

External Secrets Operator retrieves the actual secret from Azure Key Vault.

---

## Question 3

### What is the purpose of SecretStore?

It defines how ESO connects to the external secret provider.

---

## Question 4

### What is the purpose of ExternalSecret?

It defines which external secret should be synchronized into Kubernetes.

---

## Question 5

### Why use Azure Workload Identity?

It allows Kubernetes workloads to authenticate to Azure without storing long-lived Azure client secrets.

---

## Question 6

### What connects the Kubernetes ServiceAccount to the Azure Managed Identity?

The federated identity credential.

---

## Question 7

### What is stored in Git?

For example:

```yaml
remoteRef:
  key: database-password
```

Not:

```yaml
password: MyPassword123
```

---

## Question 8

### What happens when the Key Vault password changes?

ESO periodically reconciles the ExternalSecret and updates the Kubernetes Secret.

---

## Question 9

### What Azure role should the ESO Managed Identity have?

For this lab:

```text
Key Vault Secrets User
```

because ESO needs to read secrets.

---

## Question 10

### What is the difference between GitHub OIDC and AKS Workload Identity?

GitHub OIDC:

```text
GitHub Actions → Azure
```

AKS Workload Identity:

```text
Kubernetes ServiceAccount → Azure
```

They are separate authentication flows.

---

# 44. Final Verification Checklist

Before considering Lesson 19 complete:

```text
[ ] Azure Key Vault exists
[ ] database-password exists in Key Vault
[ ] External Secrets Operator is running
[ ] ESO v1 SecretStore CRD is available
[ ] azurekv provider is available
[ ] AKS OIDC issuer is enabled
[ ] AKS Workload Identity is enabled
[ ] User Assigned Managed Identity exists
[ ] Key Vault Secrets User assigned to Managed Identity
[ ] Federated Identity Credential exists
[ ] Kubernetes ServiceAccount exists
[ ] ServiceAccount has Workload Identity label
[ ] ServiceAccount has Managed Identity client ID annotation
[ ] SecretStore exists
[ ] SecretStore shows Ready=True
[ ] ExternalSecret exists
[ ] ExternalSecret shows SecretSynced=True
[ ] Kubernetes Secret exists
[ ] Application can consume the Kubernetes Secret
[ ] Secret rotation was tested
```

---

# 45. One-Line Summary

> **Git stores the secret reference, Argo CD deploys the ExternalSecret, External Secrets Operator authenticates through Azure Workload Identity, retrieves the secret from Azure Key Vault, and creates/updates the Kubernetes Secret consumed by the application.**
