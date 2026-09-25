# Lesson 16 — GitHub Actions + Argo CD

## GitOps CI/CD — Build Once, Promote DEV → QA → PROD

In this lesson we connect:

- GitHub
- GitHub Actions
- Azure Container Registry (ACR)
- Kustomize
- Argo CD
- Azure Kubernetes Service (AKS)

The final goal is:

```text
Developer
    |
    | git push
    v
GitHub
    |
    v
GitHub Actions
    |
    | Build Docker image
    | Push image
    v
Azure Container Registry
    |
    | Immutable image SHA
    v
DEV GitOps Manifest
    |
    v
Argo CD
    |
    v
AKS DEV
    |
    | Promotion PR
    v
QA GitOps Manifest
    |
    v
Argo CD
    |
    v
AKS QA
    |
    | Promotion PR
    v
PROD GitOps Manifest
    |
    v
Argo CD
    |
    v
AKS PROD
```

---

# Lesson 16A — GitHub Actions CI

## 1. What are we building?

The CI pipeline performs:

```text
Developer
    |
    | git push
    v
GitHub
    |
    v
GitHub Actions
    |
    +--> Checkout source code
    |
    +--> Validate application files
    |
    +--> Validate Dockerfile
    |
    +--> Build Docker image
    |
    +--> Login to Azure
    |
    +--> Login to ACR
    |
    +--> Push Docker image
    |
    +--> Update DEV Kustomize manifest
    |
    +--> Commit manifest change
    |
    v
Git repository
```

Argo CD then detects the Git change and deploys the desired state to AKS.

---

# 2. CI vs CD

## GitHub Actions = CI

GitHub Actions is responsible for:

- Source checkout
- Validation
- Docker build
- Image push
- Updating GitOps manifests
- Creating promotion PRs

GitHub Actions does **not** directly deploy to Kubernetes.

There is no:

```bash
kubectl apply
```

in the CI pipeline.

---

## Argo CD = CD

Argo CD is responsible for:

```text
Git desired state
       |
       v
Argo CD
       |
       v
Kubernetes / AKS
```

Argo CD continuously compares the Git desired state with the Kubernetes actual state and reconciles the cluster.

---

# 3. Environment and Azure resources

We reused the existing AKS environment.

## Resource Group

```text
rg-gitops-aks
```

## AKS

```text
aks-gitops-lab
```

## Azure Container Registry

```text
acrgitopslab11068
```

## ACR Login Server

```text
acrgitopslab11068.azurecr.io
```

## Application Image

```text
acrgitopslab11068.azurecr.io/aks-demo
```

---

# 4. Git Branch Strategy

We use three environment branches:

```text
dev
qa
main
```

Environment mapping:

```text
dev   → DEV
qa    → QA
main  → PROD
```

Promotion direction:

```text
dev
 ↓
qa
 ↓
main
```

---

# 5. Application

The application is located under:

```text
app/
```

Important files:

```text
app/
├── Dockerfile
└── index.html
```

---

# 6. Immutable Image Tagging

The Docker image is tagged using the Git commit SHA.

Example:

```text
a945cd82fe08ad0e1f7e16006337cb5733d7e6d9
```

The resulting image is:

```text
acrgitopslab11068.azurecr.io/aks-demo:a945cd82fe08ad0e1f7e16006337cb5733d7e6d9
```

This is an immutable image reference.

We do not use:

```text
latest
```

for environment promotion.

---

# 7. Why use Git SHA?

The Git SHA provides traceability.

We can determine:

```text
Git commit
    ↓
Docker image
    ↓
DEV
    ↓
QA
    ↓
PROD
```

For example:

```text
Git SHA:
a945cd82fe08ad0e1f7e16006337cb5733d7e6d9

Docker image:
acrgitopslab11068.azurecr.io/aks-demo:a945cd82fe08ad0e1f7e16006337cb5733d7e6d9
```

---

# 8. DEV Kustomize Overlay

The DEV overlay is:

```text
kustomize/gitops-nginx/overlays/dev/kustomization.yaml
```

The important section is:

```yaml
images:
  - name: nginx
    newName: acrgitopslab11068.azurecr.io/aks-demo
    newTag: "<DEV IMAGE SHA>"
```

Example:

```yaml
images:
  - name: nginx
    newName: acrgitopslab11068.azurecr.io/aks-demo
    newTag: "a945cd82fe08ad0e1f7e16006337cb5733d7e6d9"
```

---

# 9. Why `newName` and `newTag` are both important

This is an important Kustomize lesson.

If we only have:

```yaml
images:
  - name: nginx
    newTag: "a945cd82..."
```

Kustomize can render:

```text
nginx:a945cd82...
```

which means Kubernetes may try:

```text
docker.io/library/nginx:a945cd82...
```

That is not our application image.

The correct configuration is:

```yaml
images:
  - name: nginx
    newName: acrgitopslab11068.azurecr.io/aks-demo
    newTag: "a945cd82..."
```

This renders:

```text
acrgitopslab11068.azurecr.io/aks-demo:a945cd82...
```

---

# 10. Final CI Workflow

File:

```text
.github/workflows/ci.yml
```

Final workflow:

```yaml
name: CI Pipeline

on:
  push:
    branches:
      - dev
    paths:
      - 'app/**'
      - '.github/workflows/ci.yml'

  pull_request:
    branches:
      - dev
    paths:
      - 'app/**'
      - '.github/workflows/ci.yml'

permissions:
  id-token: write
  contents: write

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout source code
        uses: actions/checkout@v4

      - name: Verify application files
        run: |
          test -f app/Dockerfile
          test -f app/index.html

      - name: Validate Dockerfile
        run: |
          docker build --check ./app

      - name: Build Docker image
        run: |
          docker build \
            -t aks-demo:${{ github.sha }} \
            ./app

      - name: Azure login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Login to ACR
        run: |
          az acr login --name acrgitopslab11068

      - name: Tag Docker image
        run: |
          docker tag \
            aks-demo:${{ github.sha }} \
            acrgitopslab11068.azurecr.io/aks-demo:${{ github.sha }}

      - name: Push Docker image
        run: |
          docker push \
            acrgitopslab11068.azurecr.io/aks-demo:${{ github.sha }}

      - name: Update GitOps image tag
        run: |
          sed -i "s/newTag: \".*\"/newTag: \"${{ github.sha }}\"/" \
            kustomize/gitops-nginx/overlays/dev/kustomization.yaml

      - name: Commit GitOps manifest
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"

          git add kustomize/gitops-nginx/overlays/dev/kustomization.yaml

          git commit \
            -m "Update DEV image to ${{ github.sha }}" \
            || echo "No changes to commit"

          git push
```

---

# 11. CI Trigger

A developer changes the application:

```bash
vi app/index.html
```

Then:

```bash
git add app/index.html
git commit -m "Release Version 5"
git push origin dev
```

This triggers GitHub Actions.

---

# 12. Complete CI Flow

```text
Developer
    |
    | git push origin dev
    v
GitHub
    |
    v
GitHub Actions
    |
    +--> Checkout
    |
    +--> Validate files
    |
    +--> Validate Dockerfile
    |
    +--> docker build
    |
    +--> Azure login
    |
    +--> ACR login
    |
    +--> docker tag
    |
    +--> docker push
    |
    v
Azure Container Registry
    |
    | aks-demo:<Git SHA>
    v
Update DEV Kustomization
    |
    v
Git push
    |
    v
Argo CD
    |
    v
AKS DEV
```

---

# 13. Important Interview Question

## Why doesn't GitHub Actions run `kubectl apply`?

Because we are using GitOps.

GitHub Actions performs CI:

```text
Build
Validate
Push image
Update Git desired state
```

Argo CD performs CD:

```text
Git desired state
        ↓
Argo CD
        ↓
Kubernetes
```

This separates CI from deployment.

---

# Lesson 16B — Manual Promotion

Before automating promotion, we manually performed the promotion process.

The purpose was to understand exactly what happens to:

- Git branches
- Kustomize manifests
- Image tags
- Argo CD
- Kubernetes

---

# 14. Manual Promotion Model

The promotion model is:

```text
DEV
 |
 | Same immutable image SHA
 v
QA
 |
 | Same immutable image SHA
 v
PROD
```

Example:

```text
acrgitopslab11068.azurecr.io/aks-demo:a945cd82fe08ad0e1f7e16006337cb5733d7e6d9
```

The same image is used in:

```text
DEV
QA
PROD
```

---

# 15. Build Once, Promote Many

We do not rebuild the application for every environment.

Bad approach:

```text
DEV  → Build image
QA   → Build another image
PROD → Build another image
```

Better approach:

```text
Build once
    |
    v
Immutable image
    |
    +--> DEV
    |
    +--> QA
    |
    +--> PROD
```

This is commonly described as:

> Build once, promote many.

---

# 16. DEV → QA Manual Promotion

Suppose DEV contains:

```yaml
images:
  - name: nginx
    newName: acrgitopslab11068.azurecr.io/aks-demo
    newTag: "a945cd82fe08ad0e1f7e16006337cb5733d7e6d9"
```

QA should contain:

```yaml
images:
  - name: nginx
    newName: "acrgitopslab11068.azurecr.io/aks-demo"
    newTag: "a945cd82fe08ad0e1f7e16006337cb5733d7e6d9"
```

The important part is that the image SHA remains the same.

---

# 17. QA → PROD Manual Promotion

The same immutable image is promoted from QA to PROD.

```text
QA
 |
 | Same SHA
 v
main / PROD
```

Example:

```yaml
images:
  - name: nginx
    newName: "acrgitopslab11068.azurecr.io/aks-demo"
    newTag: "a945cd82fe08ad0e1f7e16006337cb5733d7e6d9"
```

Then:

```text
main
 ↓
Argo CD
 ↓
PROD
```

---

# 18. Why manual promotion first?

Automation can hide the actual GitOps process.

Manual promotion helped us understand:

```text
Which branch changes?

Which manifest changes?

What does Kustomize render?

What does Argo CD detect?

When does Kubernetes pull the image?
```

Once the process was understood, we automated it.

---

# Lesson 16C — Automated Promotion

The final automated architecture is:

```text
Developer
    |
    | Push to dev
    v
GitHub Actions CI
    |
    | Build once
    v
Azure Container Registry
    |
    v
DEV Manifest
    |
    v
Argo CD
    |
    v
AKS DEV
    |
    | Promotion workflow
    v
QA Promotion PR
    |
    | Merge
    v
QA Manifest
    |
    v
Argo CD
    |
    v
AKS QA
    |
    | Promotion workflow
    v
PROD Promotion PR
    |
    | Merge
    v
PROD Manifest
    |
    v
Argo CD
    |
    v
AKS PROD
```

---

# 19. Why use PR-based Promotion?

The promotion workflow does not directly modify the destination environment.

Instead:

```text
Automation
    |
    v
Create PR
    |
    v
Human review
    |
    v
Merge
    |
    v
Argo CD
    |
    v
Environment
```

This provides:

- Review
- Approval
- Audit trail
- Git history
- Controlled promotion
- Separation between environments

---

# 20. DEV → QA Automated Promotion

Workflow file:

```text
.github/workflows/promote-qa.yml
```

Final validated workflow:

```yaml
name: Promote DEV to QA

on:
  workflow_dispatch:

permissions:
  contents: write
  pull-requests: write

jobs:
  promote-to-qa:
    runs-on: ubuntu-latest

    steps:

      # --------------------------------------------------
      # Step 1: Read image SHA from DEV
      # --------------------------------------------------
      - name: Checkout DEV
        uses: actions/checkout@v4
        with:
          ref: dev
          fetch-depth: 0

      - name: Read DEV image tag
        id: image
        run: |
          IMAGE_TAG=$(grep 'newTag:' \
            kustomize/gitops-nginx/overlays/dev/kustomization.yaml \
            | sed 's/.*newTag: "\(.*\)"/\1/')

          echo "DEV image tag: $IMAGE_TAG"
          echo "tag=$IMAGE_TAG" >> "$GITHUB_OUTPUT"

      # --------------------------------------------------
      # Step 2: Checkout QA branch
      # --------------------------------------------------
      - name: Checkout QA
        uses: actions/checkout@v4
        with:
          ref: qa
          fetch-depth: 0

      # --------------------------------------------------
      # Step 3: Replace the complete QA images section
      # --------------------------------------------------
      - name: Update QA image
        env:
          IMAGE_TAG: ${{ steps.image.outputs.tag }}
        run: |
          python - <<'PY'
          from pathlib import Path
          import os
          import re

          path = Path(
              "kustomize/gitops-nginx/overlays/qa/kustomization.yaml"
          )

          text = path.read_text()

          new_tag = os.environ["IMAGE_TAG"]

          pattern = r'(?ms)^images:\n.*?(?=^labels:)'

          replacement = f'''images:
            - name: nginx
              newName: "acrgitopslab11068.azurecr.io/aks-demo"
              newTag: "{new_tag}"

          '''

          updated, count = re.subn(pattern, replacement, text)

          if count != 1:
              raise SystemExit(
                  f"Expected exactly one images section, found {count}"
              )

          path.write_text(updated)
          PY

      # --------------------------------------------------
      # Step 4: Verify the generated QA manifest
      # --------------------------------------------------
      - name: Show QA manifest
        run: |
          echo "======================================"
          echo "QA Kustomization"
          echo "======================================"

          cat kustomize/gitops-nginx/overlays/qa/kustomization.yaml

      # --------------------------------------------------
      # Step 5: Create QA promotion PR
      # --------------------------------------------------
      - name: Create QA promotion PR
        uses: peter-evans/create-pull-request@v7
        with:
          branch: promote-to-qa
          base: qa
          title: "Promote ${{ steps.image.outputs.tag }} to QA"
          commit-message: "Promote image ${{ steps.image.outputs.tag }} to QA"
          body: |
            ## QA Promotion

            Promoting the DEV image to QA.

            **Image:**

            `acrgitopslab11068.azurecr.io/aks-demo:${{ steps.image.outputs.tag }}`

            This PR updates the QA GitOps manifest.

            After merge, Argo CD will reconcile the QA environment.

          delete-branch: true
```

---

# 21. DEV → QA Flow

Suppose DEV contains:

```text
a945cd82fe08ad0e1f7e16006337cb5733d7e6d9
```

The workflow performs:

```text
Read DEV SHA
      |
      v
Checkout QA
      |
      v
Update QA manifest
      |
      v
Create promote-to-qa branch
      |
      v
Create PR
      |
      v
qa
      |
      v
Argo CD
      |
      v
QA
```

The PR contains:

```diff
 images:
   - name: nginx
     newName: "acrgitopslab11068.azurecr.io/aks-demo"
-    newTag: "old-SHA"
+    newTag: "a945cd82fe08ad0e1f7e16006337cb5733d7e6d9"
```

---

# 22. Important DEV → QA Design

The source of the image SHA is:

```text
dev
```

The destination manifest is:

```text
qa
```

So:

```text
Source:
dev

Destination:
qa
```

This is important because we initially encountered an issue when the workflow checked out DEV and modified the QA overlay from the DEV branch.

The correct approach is:

```text
Read SHA from DEV
        |
        v
Checkout actual QA branch
        |
        v
Update QA manifest
        |
        v
Create PR → QA
```

---

# 23. QA → PROD Automated Promotion

Workflow file:

```text
.github/workflows/promote-prod.yml
```

Final validated workflow:

```yaml
name: Promote QA to PROD

on:
  workflow_dispatch:

permissions:
  contents: write
  pull-requests: write

jobs:
  promote-to-prod:
    runs-on: ubuntu-latest

    steps:

      # --------------------------------------------------
      # Step 1: Read image SHA from QA
      # --------------------------------------------------
      - name: Checkout QA
        uses: actions/checkout@v4
        with:
          ref: qa
          fetch-depth: 0

      - name: Read QA image tag
        id: image
        run: |
          IMAGE_TAG=$(grep 'newTag:' \
            kustomize/gitops-nginx/overlays/qa/kustomization.yaml \
            | sed 's/.*newTag: "\(.*\)"/\1/')

          echo "QA image tag: $IMAGE_TAG"
          echo "tag=$IMAGE_TAG" >> "$GITHUB_OUTPUT"

      # --------------------------------------------------
      # Step 2: Checkout PROD / main
      # --------------------------------------------------
      - name: Checkout PROD
        uses: actions/checkout@v4
        with:
          ref: main
          fetch-depth: 0

      # --------------------------------------------------
      # Step 3: Update PROD image
      # --------------------------------------------------
      - name: Update PROD image
        env:
          IMAGE_TAG: ${{ steps.image.outputs.tag }}
        run: |
          python - <<'PY'
          from pathlib import Path
          import os
          import re

          path = Path(
              "kustomize/gitops-nginx/overlays/prod/kustomization.yaml"
          )

          text = path.read_text()

          new_tag = os.environ["IMAGE_TAG"]

          image_block = f'''images:
            - name: nginx
              newName: "acrgitopslab11068.azurecr.io/aks-demo"
              newTag: "{new_tag}"

          '''

          # If PROD already has an images section,
          # replace the entire section.
          if re.search(r'(?ms)^images:\n.*?(?=^patches:|^labels:)', text):

              pattern = r'(?ms)^images:\n.*?(?=^patches:|^labels:)'

              updated, count = re.subn(
                  pattern,
                  image_block,
                  text
              )

              if count != 1:
                  raise SystemExit(
                      f"Expected exactly one images section, found {count}"
                  )

              text = updated

          # Otherwise insert images before patches.
          else:
              if "\npatches:" in text:
                  text = text.replace(
                      "\npatches:",
                      "\n" + image_block + "patches:",
                      1
                  )
              else:
                  text = text.rstrip() + "\n\n" + image_block

          path.write_text(text)
          PY

      # --------------------------------------------------
      # Step 4: Show PROD manifest
      # --------------------------------------------------
      - name: Show PROD manifest
        run: |
          echo "======================================"
          echo "PROD Kustomization"
          echo "======================================"

          cat kustomize/gitops-nginx/overlays/prod/kustomization.yaml

      # --------------------------------------------------
      # Step 5: Create PROD promotion PR
      # --------------------------------------------------
      - name: Create PROD promotion PR
        uses: peter-evans/create-pull-request@v7
        with:
          branch: promote-to-prod
          base: main
          title: "Promote ${{ steps.image.outputs.tag }} to PROD"
          commit-message: "Promote image ${{ steps.image.outputs.tag }} to PROD"
          body: |
            ## PROD Promotion

            Promoting the QA-approved image to PROD.

            **Image:**

            `acrgitopslab11068.azurecr.io/aks-demo:${{ steps.image.outputs.tag }}`

            This PR updates the PROD GitOps manifest.

            After merge, Argo CD will reconcile the PROD environment.

          delete-branch: true
```

---

# 24. Why PROD Workflow Is Different

The original PROD overlay did not contain an `images:` section.

Original structure:

```yaml
replicas:
  - name: gitops-nginx
    count: 6

patches:
  - path: service-patch.yaml

labels:
  - pairs:
      environment: prod
```

The workflow therefore inserts:

```yaml
images:
  - name: nginx
    newName: "acrgitopslab11068.azurecr.io/aks-demo"
    newTag: "<QA IMAGE SHA>"
```

before:

```yaml
patches:
```

---

# 25. Final PROD Manifest

After promotion, PROD contains:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: gitops-prod

resources:
  - ../../base
  - namespace.yaml

replicas:
  - name: gitops-nginx
    count: 6

images:
  - name: nginx
    newName: "acrgitopslab11068.azurecr.io/aks-demo"
    newTag: "a945cd82fe08ad0e1f7e16006337cb5733d7e6d9"

patches:
  - path: service-patch.yaml

labels:
  - pairs:
      environment: prod
```

---

# 26. Complete Final Architecture

```text
                         DEVELOPER
                             |
                             | git push
                             v
                       GitHub - dev
                             |
                             v
                   +-------------------+
                   |   GitHub Actions  |
                   |       CI          |
                   +-------------------+
                             |
                             | Build image
                             | Push image
                             v
                   +-------------------+
                   |       ACR         |
                   |                   |
                   | aks-demo:<SHA>    |
                   +-------------------+
                             |
                             | Update DEV manifest
                             v
                       GitHub - dev
                             |
                             v
                          Argo CD
                             |
                             v
                         AKS DEV
                             |
                             |
                     DEV → QA Promotion
                             |
                             v
                     GitHub Actions
                             |
                             v
                    promote-to-qa
                             |
                             v
                         PR → qa
                             |
                         Human merge
                             |
                             v
                        GitHub - qa
                             |
                             v
                          Argo CD
                             |
                             v
                          AKS QA
                             |
                             |
                    QA → PROD Promotion
                             |
                             v
                     GitHub Actions
                             |
                             v
                   promote-to-prod
                             |
                             v
                        PR → main
                             |
                         Human merge
                             |
                             v
                       GitHub - main
                             |
                             v
                          Argo CD
                             |
                             v
                         AKS PROD
```

---

# 27. One Image Through All Environments

Example:

```text
acrgitopslab11068.azurecr.io/aks-demo:a945cd82fe08ad0e1f7e16006337cb5733d7e6d9
```

Promotion:

```text
                         ACR
                          |
                          v
                 a945cd82fe08...
                          |
              +-----------+-----------+
              |           |           |
              v           v           v
             DEV         QA          PROD
```

The image is built once.

It is not rebuilt for QA.

It is not rebuilt for PROD.

Only the Git desired state changes.

---

# 28. GitHub Actions Responsibilities

GitHub Actions handles:

```text
CI
 |
 +-- Checkout source
 +-- Validate files
 +-- Validate Dockerfile
 +-- Build image
 +-- Login to Azure
 +-- Login to ACR
 +-- Push image
 +-- Update GitOps manifest
 +-- Create promotion PR
```

---

# 29. Argo CD Responsibilities

Argo CD handles:

```text
CD
 |
 +-- Watch Git
 +-- Compare desired state
 +-- Detect OutOfSync
 +-- Reconcile Kubernetes
 +-- Deploy application
 +-- Report Sync status
 +-- Report Health status
```

GitHub Actions does not replace Argo CD.

Argo CD does not build Docker images.

---

# 30. GitOps Principle

Git is the source of truth for the desired state.

For example:

```yaml
newTag: "a945cd82fe08ad0e1f7e16006337cb5733d7e6d9"
```

means:

> The desired application image for this environment is the image identified by this SHA.

Argo CD reads that desired state and reconciles Kubernetes.

---

# 31. Troubleshooting Lessons

## Problem 1 — Docker Hub image was used

We encountered:

```text
Failed to pull image:

docker.io/library/nginx:<SHA>
```

Cause:

```yaml
images:
  - name: nginx
    newTag: "<SHA>"
```

without:

```yaml
newName: "acrgitopslab11068.azurecr.io/aks-demo"
```

Correct:

```yaml
images:
  - name: nginx
    newName: "acrgitopslab11068.azurecr.io/aks-demo"
    newTag: "<SHA>"
```

---

# 32. Problem 2 — Duplicate `newTag`

We encountered:

```yaml
newTag: "old-SHA"
newTag: "new-SHA"
```

Cause:

The promotion workflow used simple text replacement and did not safely replace the entire `images:` section.

Solution:

Replace the complete `images:` section deterministically.

---

# 33. Problem 3 — No PR Created

GitHub Actions reported:

```text
Branch 'promote-to-qa' no longer differs from base branch 'qa'
```

This means:

```text
Source desired state
        ==
Destination desired state
```

There is nothing to promote.

This is normal and safe behavior.

---

# 34. Problem 4 — Non-fast-forward promotion branch

We encountered:

```text
! [rejected] promote-to-qa -> promote-to-qa
(non-fast-forward)
```

Cause:

The remote temporary promotion branch already existed.

Cleanup:

```bash
git push origin --delete promote-to-qa
```

For PROD:

```bash
git push origin --delete promote-to-prod
```

The workflows use:

```yaml
delete-branch: true
```

for promotion branch cleanup.

---

# 35. Problem 5 — GitHub Actions Could Not Create PR

We encountered:

```text
GitHub Actions is not permitted to create or approve pull requests.
```

The repository setting had to allow GitHub Actions to create and approve pull requests.

GitHub repository:

```text
Settings
  → Actions
    → General
      → Workflow permissions
```

Enable:

```text
Allow GitHub Actions to create and approve pull requests
```

The workflows also require:

```yaml
permissions:
  contents: write
  pull-requests: write
```

---

# 36. Problem 6 — Workflow Not Visible in Actions

For:

```yaml
on:
  workflow_dispatch:
```

we kept the workflow on the default branch (`main`) so it could be manually executed from GitHub Actions.

Workflow maintenance pattern:

```text
Develop/update workflow
        |
        v
dev
        |
        v
Copy workflow to main
        |
        v
main
        |
        v
Run workflow
```

---

# 37. PR Creation vs Deployment

Creating a promotion PR does not deploy the application.

```text
Promotion workflow
       |
       v
PR created
       |
       X
       |
       v
No Kubernetes deployment yet
```

After merge:

```text
PR merged
   |
   v
Git branch updated
   |
   v
Argo CD detects change
   |
   v
Argo reconciliation
   |
   v
Kubernetes
```

---

# 38. Interview Scenario

## Question

How would you implement CI/CD for Kubernetes using GitHub Actions and Argo CD?

## Answer

I would separate CI and CD responsibilities.

GitHub Actions would:

1. Checkout source code.
2. Validate the application.
3. Build the Docker image.
4. Tag the image with an immutable Git SHA.
5. Push the image to ACR.
6. Update the DEV GitOps manifest.

Argo CD would watch the GitOps repository and reconcile the DEV environment.

For promotion, I would promote the same immutable image SHA through QA and PROD using Git pull requests.

The promotion workflow would read the image SHA from the source environment, update the destination Kustomize overlay, and create a PR.

After approval and merge, Argo CD would reconcile the destination environment.

The result is:

```text
Build once
Promote many
```

---

# 39. Interview Question — Why not build separately for PROD?

Because rebuilding can produce a different artifact.

Instead:

```text
Build once
    |
    v
Immutable image SHA
    |
    v
DEV
    |
    v
QA
    |
    v
PROD
```

The artifact tested in QA is the same artifact deployed to PROD.

---

# 40. Interview Question — What is the role of GitHub Actions?

GitHub Actions performs CI and Git-based automation.

In this architecture:

```text
GitHub Actions
    |
    +-- Build image
    +-- Push image
    +-- Update Git
    +-- Create promotion PR
```

It does not directly deploy Kubernetes resources.

---

# 41. Interview Question — What is the role of Argo CD?

Argo CD is the GitOps CD tool.

It continuously compares:

```text
Git desired state
        vs
Kubernetes actual state
```

and reconciles Kubernetes toward the Git-defined desired state.

---

# 42. Interview Question — Why use PR-based promotion?

PR-based promotion provides:

- Review
- Approval
- Audit trail
- Change history
- Environment separation
- Controlled production deployment

For example:

```text
QA-approved image
      |
      v
PROD PR
      |
      v
Human approval
      |
      v
main
      |
      v
Argo CD
      |
      v
PROD
```

---

# 43. Final Mental Model

Remember:

```text
GitHub Actions = Build and automate Git changes

ACR = Store immutable container images

Git = Desired state

Argo CD = Deploy and reconcile desired state

AKS = Run the application
```

---

# 44. Final Lesson 16 Summary

## Lesson 16A — CI

```text
Application code
      ↓
GitHub Actions
      ↓
Docker image
      ↓
ACR
      ↓
DEV manifest
      ↓
Argo CD
      ↓
DEV
```

---

## Lesson 16B — Manual Promotion

```text
DEV
 ↓
Manual promotion
 ↓
QA
 ↓
Manual promotion
 ↓
PROD
```

Purpose:

> Understand exactly what changes during environment promotion.

---

## Lesson 16C — Automated Promotion

```text
DEV
 ↓
GitHub Actions
 ↓
QA Promotion PR
 ↓
QA
 ↓
GitHub Actions
 ↓
PROD Promotion PR
 ↓
PROD
```

Purpose:

> Automate environment promotion while retaining PR-based approval and GitOps deployment.

---

# 45. Final Lesson 16 Principle

The most important lesson is:

```text
CI builds the artifact.

ACR stores the artifact.

Git stores the desired state.

PRs control promotion.

Argo CD performs deployment.

Kubernetes runs the application.
```

The complete flow is:

```text
Application Change
        |
        v
GitHub Actions
        |
        v
Build Docker Image
        |
        v
Push to ACR
        |
        v
Update DEV Manifest
        |
        v
Argo CD → DEV
        |
        v
DEV → QA Promotion PR
        |
        v
Argo CD → QA
        |
        v
QA → PROD Promotion PR
        |
        v
Argo CD → PROD
```

The core principle:

```text
Build Once
    ↓
Promote the Same Immutable Image
    ↓
DEV → QA → PROD
```
