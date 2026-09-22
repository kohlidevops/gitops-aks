# Lesson 15 — Git Branching Strategy for GitOps

## 🎯 Goal

In this lesson, we learn how Git branching can be used with GitOps to promote application changes through:

```text
Feature → DEV → QA → PROD
```

We will practice:

- Feature branches
- DEV branch
- QA branch
- Production (`main`) branch
- Pull Requests
- Compare vs Base branches
- Code review
- Merge
- Promotion
- Argo CD branch tracking
- GitOps OutOfSync
- Manual Sync
- Environment verification
- GitOps promotion flow

---

# 1. Why Git Branching Strategy Matters in GitOps

In GitOps, Git contains the desired Kubernetes configuration.

If we use different branches for environments, each branch can represent the configuration that should be deployed to a particular environment.

Our learning strategy:

```text
Feature Branch
      │
      │ PR
      ▼
     dev
      │
      │ PR
      ▼
      qa
      │
      │ PR
      ▼
     main
```

Argo CD connects these branches to Kubernetes environments:

```text
Git Branch                  Argo CD Application              Kubernetes

dev ─────────────────────→ gitops-nginx-dev ─────────────→ gitops-dev

qa ──────────────────────→ gitops-nginx-qa ──────────────→ gitops-qa

main ────────────────────→ gitops-nginx-prod ────────────→ gitops-prod
```

---

# 2. Important Concept — Feature Branch Has No Environment

The feature branch is temporary.

For example:

```text
feature/app-release-v2
```

It is not watched by Argo CD.

There is no:

```text
feature branch → Kubernetes
```

Instead:

```text
feature/app-release-v2
        │
        │ Pull Request
        ▼
       dev
        │
        ▼
      Argo CD
        │
        ▼
       DEV
```

The feature branch is used to develop and review the change before it enters an environment branch.

---

# 3. Our Branch Strategy

We use:

```text
feature/app-release-v2
        │
        │ PR
        ▼
       dev
        │
        │ PR
        ▼
       qa
        │
        │ PR
        ▼
      main
```

Environment mapping:

```text
dev  → DEV
qa   → QA
main → PROD
```

---

# 4. The Most Important PR Concept

When creating a GitHub Pull Request:

```text
COMPARE = where the change currently exists

BASE = where the change should go
```

Think:

```text
COMPARE ─────────→ BASE
```

Examples:

### Feature → DEV

```text
Compare: feature/app-release-v2
Base:    dev
```

Meaning:

```text
feature/app-release-v2
          │
          ▼
         dev
```

---

### DEV → QA

```text
Compare: dev
Base:    qa
```

Meaning:

```text
dev
 │
 ▼
qa
```

---

### QA → PROD

```text
Compare: qa
Base:    main
```

Meaning:

```text
qa
 │
 ▼
main
```

### Easy memory rule

```text
BASE = destination

COMPARE = source
```

---

# 5. Final Repository Structure

Repository:

```text
gitops-kustomize-aks
```

Kustomize structure:

```text
gitops-kustomize-aks/
└── kustomize/
    └── gitops-nginx/
        ├── base/
        │   ├── deployment.yaml
        │   ├── service.yaml
        │   └── kustomization.yaml
        │
        └── overlays/
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

# 6. Clean Starting Point

Before starting the lesson, we cleaned the old Lesson 15 branches.

We kept:

```text
main
```

and recreated:

```text
dev
qa
```

The starting point was:

```text
main
  │
  ├── dev
  │
  └── qa
```

Initially all three branches contained the same configuration.

This is useful because we can clearly see each promotion.

---

# 7. Creating the DEV Branch

Start from `main`:

```bash
git checkout main
git pull origin main
```

Create DEV branch:

```bash
git checkout -b dev
```

Push:

```bash
git push -u origin dev
```

Check:

```bash
git branch
```

Example:

```text
* dev
  main
```

---

# 8. Creating the QA Branch

Start from clean `main`:

```bash
git checkout main
git pull origin main
```

Create:

```bash
git checkout -b qa
```

Push:

```bash
git push -u origin qa
```

Now the branches are:

```text
main
dev
qa
```

---

# 9. Argo CD Branch Mapping

We configured Argo CD so that each environment watches a specific branch.

## DEV

Argo CD Application:

```text
gitops-nginx-dev
```

Source:

```text
Repository:
gitops-kustomize-aks

Revision:
dev

Path:
kustomize/gitops-nginx/overlays/dev
```

Architecture:

```text
dev
 │
 ▼
gitops-nginx-dev
 │
 ▼
kustomize/gitops-nginx/overlays/dev
 │
 ▼
gitops-dev
```

---

## QA

Argo CD Application:

```text
gitops-nginx-qa
```

Source:

```text
Repository:
gitops-kustomize-aks

Revision:
qa

Path:
kustomize/gitops-nginx/overlays/qa
```

Architecture:

```text
qa
 │
 ▼
gitops-nginx-qa
 │
 ▼
kustomize/gitops-nginx/overlays/qa
 │
 ▼
gitops-qa
```

---

## PROD

Argo CD Application:

```text
gitops-nginx-prod
```

Source:

```text
Repository:
gitops-kustomize-aks

Revision:
main

Path:
kustomize/gitops-nginx/overlays/prod
```

Architecture:

```text
main
 │
 ▼
gitops-nginx-prod
 │
 ▼
kustomize/gitops-nginx/overlays/prod
 │
 ▼
gitops-prod
```

---

# 10. Why We Used a Common Base Change

For this lesson, we wanted to demonstrate true promotion:

```text
DEV → QA → PROD
```

Therefore, we changed the common Kustomize base instead of changing only the DEV overlay.

File:

```text
kustomize/gitops-nginx/base/deployment.yaml
```

We added:

```yaml
env:
  - name: APP_RELEASE
    value: "v2"
```

Result:

```yaml
containers:
  - name: nginx
    image: nginx:1.27
    ports:
      - containerPort: 80
    env:
      - name: APP_RELEASE
        value: "v2"
```

Because all environments use the same base:

```text
base
 ├── DEV overlay
 ├── QA overlay
 └── PROD overlay
```

the same change can be promoted through all environments.

---

# 11. Create Feature Branch

Start from `dev`:

```bash
git checkout dev
git pull origin dev
```

Create feature branch:

```bash
git checkout -b feature/app-release-v2
```

Check:

```bash
git branch
```

Example:

```text
  dev
* feature/app-release-v2
  main
  qa
```

The feature branch starts from DEV.

---

# 12. Make the Feature Change

Edit:

```text
kustomize/gitops-nginx/base/deployment.yaml
```

Add:

```yaml
env:
  - name: APP_RELEASE
    value: "v2"
```

---

# 13. Validate Kustomize

Validate DEV:

```bash
kubectl kustomize kustomize/gitops-nginx/overlays/dev | grep -A3 APP_RELEASE
```

Validate QA:

```bash
kubectl kustomize kustomize/gitops-nginx/overlays/qa | grep -A3 APP_RELEASE
```

Validate PROD:

```bash
kubectl kustomize kustomize/gitops-nginx/overlays/prod | grep -A3 APP_RELEASE
```

All should contain:

```text
APP_RELEASE
v2
```

---

# 14. Check the Feature Change

Run:

```bash
git status
```

Then:

```bash
git diff
```

Verify that the intended file is:

```text
kustomize/gitops-nginx/base/deployment.yaml
```

and that only the intended change exists.

---

# 15. Commit the Feature

```bash
git add kustomize/gitops-nginx/base/deployment.yaml
```

Commit:

```bash
git commit -m "Add application release v2"
```

Push:

```bash
git push -u origin feature/app-release-v2
```

At this point:

```text
feature/app-release-v2
```

contains the change.

But DEV has not received it yet.

QA has not received it.

PROD has not received it.

---

# 16. Create Feature → DEV Pull Request

Go to GitHub:

```text
Pull requests
    ↓
New pull request
```

Select:

```text
Base:
dev

Compare:
feature/app-release-v2
```

Remember:

```text
COMPARE = source
BASE    = destination
```

Therefore:

```text
feature/app-release-v2
          │
          ▼
         dev
```

---

# 17. Review Files Changed

Before merging the PR, select:

```text
Files changed
```

Check that the expected file is:

```text
kustomize/gitops-nginx/base/deployment.yaml
```

Check that the change contains:

```yaml
env:
  - name: APP_RELEASE
    value: "v2"
```

The reviewer should verify:

- Correct file
- Correct configuration
- No accidental changes
- No unrelated changes

---

# 18. Merge Feature → DEV

After review:

```text
Merge pull request
```

Now:

```text
feature/app-release-v2
          │
          ▼
         dev
```

The change is now in the `dev` branch.

It is not yet in:

```text
qa
```

or:

```text
main
```

---

# 19. Which Environment Should Be Checked?

Use this simple rule:

```text
PR merged into dev  → Check DEV

PR merged into qa   → Check QA

PR merged into main → Check PROD
```

Because the first PR was merged into `dev`, check:

```text
gitops-nginx-dev
```

---

# 20. Argo CD DEV — OutOfSync

`gitops-nginx-dev` watches:

```text
Revision: dev
```

After the Git change reaches `dev`, Argo CD detects:

```text
Git desired state
       ≠
Kubernetes actual state
```

Therefore:

```text
Sync Status: OutOfSync
```

Because this lesson uses Manual Sync, click:

```text
SYNC
```

Then:

```text
SYNCHRONIZE
```

Expected final status:

```text
Synced
Healthy
```

---

# 21. Verify DEV

Run:

```bash
kubectl get deployment gitops-nginx -n gitops-dev \
  -o jsonpath='{.spec.template.spec.containers[0].env}'
```

Expected:

```text
APP_RELEASE
v2
```

At this point:

```text
DEV = v2
QA  = old version
PROD = old version
```

---

# 22. DEV → QA Promotion

After DEV validation, promote the same change to QA.

Do not create another feature branch.

The change already exists in:

```text
dev
```

Create a Pull Request:

```text
Base:
qa

Compare:
dev
```

Therefore:

```text
dev
 │
 ▼
qa
```

Remember:

```text
Compare = dev
Base    = qa
```

---

# 23. Review DEV → QA PR

Open:

```text
Files changed
```

Verify that the expected change is being promoted:

```text
kustomize/gitops-nginx/base/deployment.yaml
```

with:

```yaml
APP_RELEASE: v2
```

This PR means:

> Take the already tested DEV change and promote it to QA.

Merge the PR.

Now:

```text
dev
 │
 ▼
qa
```

---

# 24. Check QA

Because the PR was merged into:

```text
qa
```

we check:

```text
gitops-nginx-qa
```

Argo CD watches:

```text
Revision: qa
```

It detects the Git change.

Expected:

```text
OutOfSync
```

Because:

```text
Git QA:
APP_RELEASE=v2

Kubernetes QA:
old configuration
```

Click:

```text
SYNC
```

Then:

```text
SYNCHRONIZE
```

Wait for:

```text
Synced
Healthy
```

---

# 25. Verify QA

Run:

```bash
kubectl get deployment gitops-nginx -n gitops-qa \
  -o jsonpath='{.spec.template.spec.containers[0].env}'
```

Expected:

```text
APP_RELEASE
v2
```

Now:

```text
DEV  = v2
QA   = v2
PROD = old version
```

---

# 26. QA → PROD Promotion

After QA validation, promote the change to production.

Create a Pull Request:

```text
Base:
main

Compare:
qa
```

Therefore:

```text
qa
 │
 ▼
main
```

Remember:

```text
Compare = qa
Base    = main
```

---

# 27. Review QA → PROD PR

Open:

```text
Files changed
```

Verify:

```text
kustomize/gitops-nginx/base/deployment.yaml
```

contains:

```yaml
APP_RELEASE: v2
```

Review carefully because this PR targets the production branch.

---

# 28. Merge QA → MAIN

Merge the PR.

Now:

```text
qa
 │
 ▼
main
```

The change is now in the production branch.

---

# 29. Check PROD

Because the PR was merged into:

```text
main
```

check:

```text
gitops-nginx-prod
```

Argo CD watches:

```text
Revision: main
```

Expected:

```text
OutOfSync
```

Why?

```text
Git main:
APP_RELEASE=v2

Kubernetes PROD:
old configuration
```

Click:

```text
SYNC
```

Then:

```text
SYNCHRONIZE
```

Expected:

```text
Synced
Healthy
```

---

# 30. Verify PROD

Run:

```bash
kubectl get deployment gitops-nginx -n gitops-prod \
  -o jsonpath='{.spec.template.spec.containers[0].env}'
```

Expected:

```text
APP_RELEASE
v2
```

Final state:

```text
DEV  = v2
QA   = v2
PROD = v2
```

---

# 31. Complete GitOps Promotion Flow

The complete flow we practiced:

```text
                Developer
                    │
                    ▼
          feature/app-release-v2
                    │
                    │ PR
                    │
                    ▼
                   dev
                    │
                    │ Argo CD
                    ▼
                  DEV
                    │
                    │
                    │ PR
                    ▼
                   qa
                    │
                    │ Argo CD
                    ▼
                   QA
                    │
                    │
                    │ PR
                    ▼
                  main
                    │
                    │ Argo CD
                    ▼
                  PROD
```

---

# 32. GitHub PR Cheat Sheet

| Promotion | Compare | Base | Environment to Check |
|---|---|---|---|
| Feature → DEV | `feature/app-release-v2` | `dev` | DEV |
| DEV → QA | `dev` | `qa` | QA |
| QA → PROD | `qa` | `main` | PROD |

Remember:

```text
Compare = SOURCE
Base    = DESTINATION
```

---

# 33. Argo CD Cheat Sheet

| Git Branch | Argo CD Application | Kubernetes Namespace |
|---|---|---|
| `dev` | `gitops-nginx-dev` | `gitops-dev` |
| `qa` | `gitops-nginx-qa` | `gitops-qa` |
| `main` | `gitops-nginx-prod` | `gitops-prod` |

---

# 34. OutOfSync Cheat Sheet

When a Git change reaches an environment branch:

```text
Git changed
    ↓
Argo CD detects change
    ↓
Desired state != Live state
    ↓
OutOfSync
    ↓
Manual Sync
    ↓
Kubernetes updated
    ↓
Synced
```

For our lab:

```text
PR → branch
     ↓
Argo CD detects Git change
     ↓
OutOfSync
     ↓
SYNC
     ↓
Healthy
```

---

# 35. Why OutOfSync May Take Some Time

Argo CD may not detect a Git change immediately if a webhook is not configured.

For example:

```text
GitHub PR merged
      ↓
QA branch updated
      ↓
Argo CD repository refresh
      ↓
OutOfSync
```

There can be a delay between the Git merge and Argo CD detecting the new commit.

This is normal.

A GitHub webhook can provide faster notification.

---

# 36. Important Distinction — Merge vs Deploy

A GitHub PR merge does not directly deploy to Kubernetes.

For example:

```text
PR merged into qa
```

means:

```text
QA Git desired state changed
```

Then:

```text
Argo CD detects the change
        ↓
OutOfSync
        ↓
Sync
        ↓
Kubernetes changes
```

Therefore:

```text
Git Merge ≠ Kubernetes Deployment
```

Argo CD performs the GitOps deployment/reconciliation.

---

# 37. Promotion Does Not Mean Copying Files Manually

We don't do:

```text
DEV YAML
   ↓
copy/paste
   ↓
QA YAML
```

Instead:

```text
Feature
   ↓
dev
   ↓
qa
   ↓
main
```

Git history records the promotion.

This gives us:

- Review
- Approval workflow
- Audit history
- Controlled promotion
- Easy rollback
- Traceability

---

# 38. Rollback Concept

Suppose:

```text
main
```

contains:

```text
APP_RELEASE=v2
```

and production has a problem.

We can revert the Git change.

Example:

```text
main
 │
 ├── v1
 │
 └── v2  ← current
```

Create a revert through Git:

```text
v2
 ↓
revert
 ↓
v1
```

Then Argo CD detects the Git change:

```text
Git
 ↓
OutOfSync
 ↓
Sync
 ↓
PROD returns toward v1
```

Git becomes the record of the rollback.

---

# 39. Production Branching Model

The model used in this lesson is:

```text
feature
   ↓
 dev
   ↓
 qa
   ↓
main
```

This is only one possible GitOps strategy.

Organizations can also use:

- Feature → main
- Environment directories
- Git tags
- Release branches
- Image promotion
- GitOps repositories separated by environment
- Pull-request based environment promotion

The important GitOps principles remain:

```text
Git = Desired State

Argo CD = Reconciliation

Kubernetes = Actual Runtime
```

---

# 40. Interview Questions

## Q1. What is a feature branch?

A temporary Git branch used to develop a change independently before merging it into another branch.

Example:

```text
feature/app-release-v2
```

---

## Q2. What is environment promotion?

Moving an already developed and reviewed change from one environment stage to another.

Example:

```text
DEV → QA → PROD
```

In our Git strategy:

```text
dev → qa → main
```

---

## Q3. What is the purpose of Pull Requests?

Pull Requests provide a controlled mechanism to review and merge changes.

They can provide:

- Code review
- Approval
- Automated validation
- Audit history
- Controlled promotion

---

## Q4. What is Base in a GitHub PR?

Base is the destination branch.

Example:

```text
dev → qa
```

The PR should have:

```text
Base: qa
```

---

## Q5. What is Compare in a GitHub PR?

Compare is the source branch containing the changes.

Example:

```text
dev → qa
```

The PR should have:

```text
Compare: dev
```

---

## Q6. What happens after merging a PR into `dev`?

The `dev` branch changes.

If Argo CD DEV watches `dev`, Argo CD detects the desired-state change.

It may show:

```text
OutOfSync
```

After synchronization:

```text
Synced
Healthy
```

---

## Q7. Does merging a PR automatically deploy to Kubernetes?

Not necessarily.

The merge changes Git.

Argo CD must detect the Git change and reconcile Kubernetes.

With manual sync:

```text
Git merge
   ↓
OutOfSync
   ↓
Manual Sync
   ↓
Kubernetes
```

With automatic sync:

```text
Git merge
   ↓
OutOfSync/detection
   ↓
Automatic Sync
   ↓
Kubernetes
```

---

## Q8. Why does DEV watch `dev` instead of `main`?

Because we want DEV to consume changes that have been promoted into the DEV branch.

```text
dev → DEV
qa → QA
main → PROD
```

---

## Q9. Why did we use a common base change?

Because we wanted the same change to move through:

```text
DEV → QA → PROD
```

The common base is inherited by all overlays.

---

## Q10. What is the GitOps promotion flow?

```text
Feature
   ↓ PR
dev
   ↓ PR
qa
   ↓ PR
main
```

Argo CD maps those branches to:

```text
DEV
QA
PROD
```

---

# 41. Troubleshooting

## PR merged but Argo CD still says Synced

Check:

```bash
kubectl get application gitops-nginx-qa -n argocd \
  -o jsonpath='{.spec.source.targetRevision}{"\n"}'
```

Expected:

```text
qa
```

Check the branch:

```bash
git checkout qa
git pull origin qa
git log --oneline -5
```

You can request a hard refresh:

```bash
kubectl annotate application gitops-nginx-qa \
  -n argocd \
  argocd.argoproj.io/refresh=hard --overwrite
```

---

## Argo CD is OutOfSync but Sync fails

Check:

```bash
kubectl describe application gitops-nginx-qa -n argocd
```

Then inspect Kubernetes:

```bash
kubectl get pods -n gitops-qa
```

```bash
kubectl get deployment -n gitops-qa
```

```bash
kubectl get events -n gitops-qa --sort-by=.lastTimestamp
```

---

# 42. Final Mental Model

Remember these three layers:

```text
              GIT
         Desired State
              │
              │
              ▼
           Argo CD
       GitOps Controller
              │
              │ Reconcile
              ▼
        KUBERNETES / AKS
          Actual State
```

And our branch strategy:

```text
feature
   │
   │ PR
   ▼
 dev ─────────────→ DEV
   │
   │ PR
   ▼
 qa ──────────────→ QA
   │
   │ PR
   ▼
main ─────────────→ PROD
```

Most important rule:

```text
COMPARE = SOURCE
BASE    = DESTINATION
```

Most important environment rule:

```text
Merged to dev  → Check DEV

Merged to qa   → Check QA

Merged to main → Check PROD
```

---

# 43. Lesson 15 Checklist

- [x] Understand feature branch
- [x] Understand DEV branch
- [x] Understand QA branch
- [x] Understand main/production branch
- [x] Understand branch-to-environment mapping
- [x] Create feature branch
- [x] Make a feature change
- [x] Validate Kustomize
- [x] Commit feature change
- [x] Push feature branch
- [x] Create Feature → DEV PR
- [x] Understand Compare vs Base
- [x] Review Files Changed
- [x] Merge Feature → DEV
- [x] Observe DEV OutOfSync
- [x] Sync DEV
- [x] Verify DEV
- [x] Create DEV → QA PR
- [x] Review DEV → QA changes
- [x] Merge DEV → QA
- [x] Observe QA OutOfSync
- [x] Sync QA
- [x] Verify QA
- [ ] Create QA → PROD PR
- [ ] Review QA → PROD changes
- [ ] Merge QA → main
- [ ] Observe PROD OutOfSync
- [ ] Sync PROD
- [ ] Verify PROD
- [ ] Understand GitOps rollback

---

# Final Interview Statement

> In our GitOps workflow, feature branches are used for development, while environment branches represent the desired state for DEV, QA, and PROD. A feature is promoted using Pull Requests from feature → dev → qa → main. GitHub Pull Requests provide review and controlled promotion, while Argo CD watches the corresponding environment branch and reconciles Kubernetes whenever the desired state changes. After a merge, Argo CD detects the difference as OutOfSync, and synchronization updates the target Kubernetes environment.

```
