# Lesson 17 — Image Tagging Strategy

## Objective

In this lesson, we will understand how container image tags affect:

- Deployment traceability
- Rollbacks
- GitOps
- CI/CD
- Production releases
- DEV → QA → PROD promotion

We will use our existing `gitops-kustomize-aks` project.

We will **not create a new AKS cluster, ACR, VM, or other Azure resources**.

---

# 1. The Problem With `latest`

Consider this Kubernetes Deployment:

```yaml
containers:
  - name: nginx
    image: acrgitopslab11068.azurecr.io/aks-demo:latest
```

Initially:

```text
ACR

aks-demo:latest
       ↓
   Version 1
```

Later, a new build is pushed:

```text
ACR

aks-demo:latest
       ↓
   Version 2
```

The Kubernetes manifest hasn't changed:

```yaml
image: acrgitopslab11068.azurecr.io/aks-demo:latest
```

But the image behind the tag has changed.

Therefore:

```text
Yesterday:

latest → Version 1


Today:

latest → Version 2
```

This is the fundamental problem with using `latest` as a production deployment reference.

---

# 2. Why `latest` Is Dangerous

## Problem 1 — Release identification

If PROD contains:

```yaml
image: aks-demo:latest
```

we cannot determine the application release simply from the manifest.

Compare:

```text
aks-demo:latest
```

with:

```text
aks-demo:1.0.23
```

The second one clearly identifies the intended application release.

Or:

```text
aks-demo:a945cd82fe08ad0e1f7e16006337cb5733d7e6d9
```

which identifies the source revision.

---

# 3. Rollback Problem

Imagine:

```text
PROD
  ↓
latest
  ↓
Version 2
```

Version 2 has a production issue.

We want:

```text
Version 1
```

But `latest` may now point to Version 2.

We have to determine:

- Which image was previously deployed?
- Which digest represented that image?
- Does the old image still exist?
- What did `latest` point to previously?
- Is the old image still available?

With an immutable/versioned tag:

```text
PROD
  ↓
aks-demo:1.0.2
```

Rollback can be:

```text
1.0.2
  ↓
rollback
  ↓
1.0.1
```

The desired image is explicit.

---

# 4. Tagging Strategies

There are several common image tagging strategies.

## 4.1 `latest`

```text
aks-demo:latest
```

Meaning:

```text
"Use the image currently referenced by latest."
```

Problem:

```text
latest
  ↓
can change over time
```

It is therefore a poor choice for controlled production releases.

---

## 4.2 Build Number

Example:

```text
aks-demo:1001
aks-demo:1002
aks-demo:1003
```

Advantages:

- Unique build identifier
- Easy to generate automatically
- Easy to track CI builds

Disadvantage:

```text
1003
```

doesn't tell us much about the application release itself.

---

## 4.3 Semantic Version

Example:

```text
aks-demo:1.0.23
```

Common structure:

```text
MAJOR.MINOR.PATCH
```

Example:

```text
1.0.23
│ │ │
│ │ └── PATCH
│ └──── MINOR
└────── MAJOR
```

---

## 4.4 Git Commit SHA

Example:

```text
aks-demo:a945cd82fe08ad0e1f7e16006337cb5733d7e6d9
```

This gives direct traceability:

```text
Container Image
      ↓
Git Commit
      ↓
Exact Source Revision
```

This is the strategy we already implemented in Lesson 16.

---

## 4.5 Version + Git SHA

Another possible strategy is:

```text
aks-demo:1.0.23-a945cd82
```

This provides:

```text
1.0.23
   ↓
Application release

a945cd82
   ↓
Source revision
```

The exact tagging convention is an organizational decision.

---

# 5. Semantic Versioning

A semantic version normally follows:

```text
MAJOR.MINOR.PATCH
```

For example:

```text
1.0.23
```

### PATCH

Bug fix:

```text
1.0.23
   ↓
1.0.24
```

### MINOR

Backward-compatible feature:

```text
1.0.23
   ↓
1.1.0
```

### MAJOR

Breaking change:

```text
1.0.23
   ↓
2.0.0
```

The exact meaning and release policy should be defined by the development team.

---

# 6. Tag vs Digest

This is an important concept.

## Tag

Example:

```text
aks-demo:1.0.23
```

A tag is a human-readable reference.

## Digest

Example:

```text
sha256:abcdef123456...
```

A digest identifies the exact image content.

Conceptually:

```text
Tag
 ↓
1.0.23
 ↓
Image
 ↓
Digest
 ↓
sha256:...
```

The digest is the strongest identifier of the actual container image content.

---

# 7. Important: A Version Tag Is Not Automatically Immutable

Consider:

```text
aks-demo:1.0.23
```

Someone could technically push another image using the same tag:

```text
1.0.23 → Image A
```

and later:

```text
1.0.23 → Image B
```

Therefore:

> A tag becomes part of an immutable tagging strategy only when the release tag is never reused or overwritten.

The production rule should be:

```text
1.0.23
   ↓
ONE IMAGE
```

not:

```text
1.0.23
   ↓
Image A
   ↓
Image B
```

---

# 8. Our Lesson 16 Image Tagging Strategy

In Lesson 16, our GitHub Actions workflow used:

```yaml
docker build \
  -t aks-demo:${{ github.sha }} \
  ./app
```

Then:

```yaml
docker tag \
  aks-demo:${{ github.sha }} \
  acrgitopslab11068.azurecr.io/aks-demo:${{ github.sha }}
```

Then:

```yaml
docker push \
  acrgitopslab11068.azurecr.io/aks-demo:${{ github.sha }}
```

Therefore, if the Git commit is:

```text
a945cd82fe08ad0e1f7e16006337cb5733d7e6d9
```

the image becomes:

```text
acrgitopslab11068.azurecr.io/aks-demo:a945cd82fe08ad0e1f7e16006337cb5733d7e6d9
```

---

# 9. Why Git SHA Is Useful

Suppose PROD contains:

```text
acrgitopslab11068.azurecr.io/aks-demo:a945cd82...
```

We can trace:

```text
PROD
 ↓
Image
 ↓
Git SHA
 ↓
Git commit
 ↓
Source code change
```

This helps with:

- Troubleshooting
- Auditing
- Incident investigation
- Rollbacks
- Release tracking
- Deployment traceability

---

# 10. Build Once, Promote Many

This is the most important concept of this lesson.

A bad approach is:

```text
Source
  ↓
Build
  ↓
DEV image

Source
  ↓
Build
  ↓
QA image

Source
  ↓
Build
  ↓
PROD image
```

We are building the application multiple times.

A better approach is:

```text
Source
  ↓
ONE BUILD
  ↓
ONE IMMUTABLE IMAGE
  ↓
ACR
  ↓
DEV
  ↓
QA
  ↓
PROD
```

This is commonly described as:

```text
Build Once
Promote Many
```

---

# 11. Why Rebuilding for PROD Can Be Dangerous

Suppose we build:

```text
DEV image
```

Then later build again for PROD.

The environment may have changed because of:

- Base image changes
- Dependency changes
- Package repository changes
- Build tooling changes
- Build environment changes

Therefore:

```text
DEV image ≠ necessarily PROD image
```

Instead:

```text
DEV image
   ↓
tested
   ↓
same image
   ↓
QA
   ↓
same image
   ↓
PROD
```

This gives us much stronger artifact consistency.

---

# 12. Our GitOps Promotion Flow

Our current architecture is:

```text
Developer
    ↓
GitHub
    ↓
GitHub Actions
    ↓
Docker Build
    ↓
ACR
    ↓
GitOps Repository
    ↓
Argo CD
    ↓
AKS
```

Environment promotion:

```text
DEV
 │
 │ same image
 ↓
QA
 │
 │ same image
 ↓
PROD
```

For example:

```text
acrgitopslab11068.azurecr.io/aks-demo:
a945cd82fe08ad0e1f7e16006337cb5733d7e6d9
```

The same image is promoted through all environments.

---

# 13. Practice Environment

We will use the existing environment:

```text
Resource Group:
rg-gitops-aks

AKS:
aks-gitops-lab

ACR:
acrgitopslab11068

Repository:
aks-demo

GitHub Repository:
gitops-kustomize-aks
```

Do not create another AKS cluster or ACR.

---

# 14. Practice 1 — Inspect `latest`

Run on the existing Ubuntu VM:

```bash
docker --version
```

Then:

```bash
docker pull nginx:latest
```

Inspect the image:

```bash
docker image inspect nginx:latest \
  --format '{{index .RepoDigests 0}}'
```

You should see something similar to:

```text
nginx@sha256:xxxxxxxxxxxxxxxxxxxxxxxx
```

The important relationship is:

```text
nginx:latest
      ↓
specific image digest
```

The tag is a reference to the image.

---

# 15. Practice 2 — Inspect a Versioned Image

Pull a versioned image:

```bash
docker pull nginx:1.27
```

Then:

```bash
docker image inspect nginx:1.27 \
  --format '{{index .RepoDigests 0}}'
```

Now compare:

```text
nginx:latest
nginx:1.27
```

Conceptually:

```text
latest
   ↓
moving reference

1.27
   ↓
version reference
```

Remember:

> A version tag is only part of an immutable strategy if that tag is never reused or overwritten.

---

# 16. Practice 3 — Inspect Our ACR Tags

Now inspect the actual ACR repository used by our project:

```bash
az acr repository show-tags \
  --name acrgitopslab11068 \
  --repository aks-demo \
  --output table
```

You should see the image tags generated during our previous lessons.

For example:

```text
a945cd82fe08ad0e1f7e16006337cb5733d7e6d9
fc9119d81705440bc76a93222f041ab7fed918cb
26a905a91b611cd1c8919daaba7a9ce56368a302
```

The important observation is:

```text
Each build
    ↓
different Git SHA
    ↓
different image tag
```

---

# 17. Practice 4 — Inspect ACR Manifest Information

Run:

```bash
az acr repository show-manifests \
  --name acrgitopslab11068 \
  --repository aks-demo \
  --output table
```

The exact output format can vary with Azure CLI/API versions.

Look for the relationship between:

```text
Tag
Digest
```

Conceptually:

```text
Tag
 ↓
a945cd82...
 ↓
Digest
 ↓
sha256:...
```

The tag gives us a convenient reference.

The digest identifies the actual image content.

---

# 18. Practice 5 — Check GitOps Image Tags

Check DEV:

```bash
grep 'newTag:' \
  kustomize/gitops-nginx/overlays/dev/kustomization.yaml
```

Check QA:

```bash
grep 'newTag:' \
  kustomize/gitops-nginx/overlays/qa/kustomization.yaml
```

Check PROD:

```bash
grep 'newTag:' \
  kustomize/gitops-nginx/overlays/prod/kustomization.yaml
```

We expect the promoted release to look conceptually like:

```text
DEV  → a945cd82...
QA   → a945cd82...
PROD → a945cd82...
```

The exact SHA depends on the current repository state.

The important point is:

```text
ONE IMAGE
   ↓
DEV
   ↓
QA
   ↓
PROD
```

---

# 19. Practice 6 — Trace Image Back to Git

Suppose the deployed image is:

```text
a945cd82fe08ad0e1f7e16006337cb5733d7e6d9
```

If that commit exists in your local repository:

```bash
git show a945cd82fe08ad0e1f7e16006337cb5733d7e6d9
```

Or:

```bash
git log --oneline --all | grep a945cd82
```

The trace becomes:

```text
Git Commit
    ↓
Docker Image Tag
    ↓
ACR
    ↓
DEV
    ↓
QA
    ↓
PROD
```

This is one of the biggest advantages of using Git SHA image tags.

---

# 20. Practice 7 — Verify "Build Once, Promote Many"

Compare:

```text
DEV
```

```bash
grep 'newTag:' \
  kustomize/gitops-nginx/overlays/dev/kustomization.yaml
```

Then:

```text
QA
```

```bash
grep 'newTag:' \
  kustomize/gitops-nginx/overlays/qa/kustomization.yaml
```

Then:

```text
PROD
```

```bash
grep 'newTag:' \
  kustomize/gitops-nginx/overlays/prod/kustomization.yaml
```

If the same release was promoted:

```text
DEV  → SHA-X
QA   → SHA-X
PROD → SHA-X
```

That demonstrates:

```text
Build Once
     ↓
Same Image
     ↓
Promote
     ↓
DEV → QA → PROD
```

---

# 21. Practice 8 — Think About a Production Release

Suppose the team wants release:

```text
1.0.23
```

A possible image could be:

```text
acrgitopslab11068.azurecr.io/aks-demo:1.0.23
```

If additional Git traceability is required:

```text
acrgitopslab11068.azurecr.io/aks-demo:1.0.23-a945cd82
```

Now we have:

```text
1.0.23
   ↓
Application release

a945cd82
   ↓
Git source revision
```

We are not changing our existing pipeline yet.

The purpose here is to understand the design options.

---

# 22. Production Tagging Rules

A production-oriented tagging strategy should generally follow these rules:

### Rule 1

Do not use:

```text
latest
```

as the production release identifier.

### Rule 2

Give every build/release a unique identifier.

Examples:

```text
1.0.23
```

or:

```text
a945cd82...
```

### Rule 3

Never reuse a production release tag for different image content.

Bad:

```text
1.0.23 → Image A
1.0.23 → Image B
```

Good:

```text
1.0.23 → Image A
1.0.24 → Image B
```

### Rule 4

Promote the same tested image.

```text
DEV
 ↓
QA
 ↓
PROD
```

not:

```text
DEV → rebuild
QA  → rebuild
PROD → rebuild
```

---

# 23. Interview Discussion

## Question 1

### Why is `latest` dangerous?

Answer:

> `latest` is a mutable tag. It can point to different image content over time, so the Kubernetes manifest does not clearly identify which application version is running. This makes auditing, troubleshooting and rollback more difficult.

---

## Question 2

### Is `1.0.23` automatically immutable?

Answer:

> No. A registry can technically allow the same tag to be overwritten. An immutable tagging strategy requires release tags to be unique and never reused for different image content.

---

## Question 3

### Why use Git SHA as an image tag?

Answer:

> A Git SHA provides direct traceability between the container image and the exact source revision that produced it.

---

## Question 4

### Why shouldn't we rebuild the image for PROD?

Answer:

> The image tested in lower environments should be the same artifact promoted to production. Rebuilding can introduce differences in dependencies, base images or build environments.

---

## Question 5

### What does "Build Once, Promote Many" mean?

Answer:

> Build the container once, store that immutable artifact in the registry, and promote the same image through DEV, QA and PROD.

---

## Question 6

### What is the difference between an image tag and image digest?

Answer:

> A tag is a human-readable reference such as `1.0.23`. A digest such as `sha256:...` identifies the exact image content.

---

# 24. Final Architecture

Our desired image lifecycle is:

```text
                    SOURCE CODE
                         │
                         ↓
                    Git Commit
                         │
                         ↓
                  GitHub Actions
                         │
                         ↓
                    Docker Build
                         │
                         ↓
                 Immutable Image
                         │
                         ↓
                        ACR
                         │
              ┌──────────┼──────────┐
              ↓          ↓          ↓
             DEV         QA        PROD
              │          │          │
              └──────── SAME ───────┘
                       IMAGE
```

The key principle:

```text
Build Once
    ↓
Store Immutable Image
    ↓
Promote Same Image
    ↓
DEV → QA → PROD
```

---

# 25. Tagging Strategy Summary

```text
┌──────────────────────┬─────────────────────────────┐
│ Tag Strategy         │ Example                     │
├──────────────────────┼─────────────────────────────┤
│ latest               │ aks-demo:latest             │
│ Build number         │ aks-demo:1003               │
│ Semantic version     │ aks-demo:1.0.23             │
│ Git SHA              │ aks-demo:a945cd82...        │
│ Version + Git SHA    │ aks-demo:1.0.23-a945cd82   │
└──────────────────────┴─────────────────────────────┘
```

For our current GitOps implementation:

```text
Git SHA
   ↓
ACR
   ↓
DEV
   ↓
QA
   ↓
PROD
```

---

# 26. Key Takeaways

Remember these six points:

```text
1. latest is a moving target.

2. Production should use a unique release identifier.

3. Tags are references; digests identify exact image content.

4. Git SHA provides strong source-to-image traceability.

5. Never reuse an immutable production release tag.

6. Build once and promote the same image through DEV → QA → PROD.
```

The most important mental model:

```text
Source Code
    ↓
Git Commit
    ↓
ONE Docker Build
    ↓
ONE Immutable Image
    ↓
ACR
    ↓
DEV
    ↓
QA
    ↓
PROD
```

---

# 27. Practice Checklist

Complete these before moving to Lesson 18:

```text
[ ] Understand why latest is dangerous

[ ] Understand mutable vs immutable tags

[ ] Understand semantic versioning

[ ] Understand Git SHA image tags

[ ] Understand image tags vs image digests

[ ] Inspect nginx:latest

[ ] Inspect nginx:1.27

[ ] List aks-demo tags in ACR

[ ] Inspect ACR manifests/digests

[ ] Check DEV image tag

[ ] Check QA image tag

[ ] Check PROD image tag

[ ] Verify the same image is promoted DEV → QA → PROD

[ ] Understand Build Once, Promote Many

[ ] Be able to explain the strategy in an interview
```

---

# Lesson 17 — Final Mental Model

```text
              Git Commit
                   │
                   ↓
             GitHub Actions
                   │
                   ↓
              Docker Build
                   │
                   ↓
         Immutable Image Tag
                   │
                   ↓
                  ACR
                   │
          ┌────────┼────────┐
          ↓        ↓        ↓
         DEV       QA      PROD
          │        │        │
          └────────┼────────┘
                   ↓
              SAME IMAGE
```

```text
❌ latest

aks-demo:latest
      ↓
moving target


✅ Version

aks-demo:1.0.23
      ↓
specific release


✅ Git SHA

aks-demo:a945cd82...
      ↓
exact source revision


✅ Build Once, Promote Many

ONE IMAGE
   ↓
DEV → QA → PROD
```

**Lesson 17 complete when you can explain not only "what tag should I use?" but also "why does the same immutable image need to be promoted from DEV to QA to PROD?"**
