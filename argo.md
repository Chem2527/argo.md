# Argo CD + Immutable Tags — LMB Production (Option 1)

**Audience:** Platform / DevOps / app owners (forwardable)  
**Chosen approach:** **Option 1** (recommended) — GitHub Actions pushes immutable image to ACR, then updates a **GitOps repo**; **Argo CD** syncs that Git state into Kubernetes.  
**Not chosen for pilot:** Option 2 (Argo CD Image Updater) — defer until many repos make CI→GitOps steps heavy.  
**Pilot service:** `lmb_statement_generator`  
**Cluster:** PRD (access from laptop via **FortiClient VPN**)  
**Argo CD location:** Runs **inside the cluster** (`argocd` ns). Laptop only opens UI via VPN + port-forward (not a local Argo install).  
**Date:** 2026-07-21  

**Status:** Proposal / design only. No production branches were modified in this review.

---

## 1. Problem (today)

| Today | Impact |
|---|---|
| CI pushes mutable tag `pddlmbcr.azurecr.io/statementgenerator:prd` | Every push overwrites the same tag |
| Deployment uses `image: ...:prd` + `imagePullPolicy: Always` | Cluster always pulls whatever `:prd` points to **now** |
| `kubectl rollout undo` / Deployment revision rollback | Spec still says `:prd` → pods come back on the **new** image, not the old one |

**Rollback of the old stable build is not possible with mutable `:prd` alone.**

---

## 2. What we will do (Option 1 — chosen)

```text
Merge to production
        │
        ▼
GitHub Actions (prodacr.yml)  [app repo]
  1. Build image
  2. Push to ACR:  .../statementgenerator:prd-<github.run_number>
     (optional also keep :prd as a floating pointer for humans)
  3. On SUCCESS only → update GitOps repo values:
        image.tag: prd-<run_number>
        │
        ▼
GitOps repo (e.g. lmb-gitops)   ← source of truth for “what prod should run”
        │
        ▼
Argo CD (watches GitOps repo)
  → updates Deployment in middleware-prd-ns
  → nodes PULL image from ACR (Argo does not push images)
```

### Mental model (one line)

> **CI builds `prd-<run_number>` → CI updates GitOps tag → Argo CD syncs Deployment → cluster pulls that tag from ACR.**

Argo CD does **not** watch GitHub Actions and does **not** “fetch images from GHA.” It only applies what is in the GitOps repo.

### Decision: Option 1 vs Option 2

| | **Option 1 (CHOSEN)** | Option 2 (later / optional) |
|---|---|---|
| How new tag reaches Argo | **GitHub Actions** updates GitOps after green build | **Image Updater** watches ACR and writes Git |
| Extra cluster component | No | Yes (Image Updater) |
| Clear “CI passed ⇒ deploy this tag” | **Yes** | Indirect |
| Recommendation | **Use this for pilot and rollout** | Revisit only if CI→GitOps steps become too heavy across many repos |

**Team decision: proceed with Option 1.**

---

## 3. Accessing Argo CD from your laptop (FortiClient VPN)

Argo CD is **not** installed on your PC. It runs **inside the PRD cluster** (namespace `argocd`). Your laptop only opens the UI/API over VPN.

### Local expose (recommended for PRD — no public URL needed)

```bash
# 1) Connect FortiClient VPN
# 2) Confirm cluster context
kubectl config current-context
kubectl get ns argocd

# 3) Port-forward Argo CD server to localhost
kubectl -n argocd port-forward svc/argocd-server 8080:443

# 4) Browser: https://localhost:8080
#    User: admin
#    Password:
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

You can do the same port-forward from **Lens** (forward `argocd-server` service → local port).

| Method | When to use |
|---|---|
| `kubectl port-forward` / Lens forward | **Default for PRD** (VPN-only, simple) |
| Internal Ingress / private LB | Later, for shared team URL (still VPN / private network) |
| Public Internet Ingress | **Not recommended** for production Argo without strong SSO + IP allowlists |

Without FortiClient VPN, port-forward to PRD will fail or hang.

---

## 4. Argo CD vs Lens — same Deployments (this is the requirement)

**Yes — the requirement is that Argo CD updates the live Deployment objects in the cluster** (the same ones you see and edit today in Lens).

| Tool | What it does |
|---|---|
| **Lens** | UI to **view** (and today, sometimes manually edit) Deployments, Pods, Secrets, ConfigMaps |
| **Argo CD** | Reads GitOps → **applies/syncs** the Deployment in `middleware-prd-ns` via Kubernetes API |
| **GitHub Actions** | Builds/pushes image to ACR + writes `image.tag` into GitOps (Option 1) |

Example for pilot:

1. CI pushes `pddlmbcr.azurecr.io/statementgenerator:prd-87`  
2. CI sets GitOps `tag: prd-87`  
3. Argo CD syncs → live Deployment `statementgenerator` image = `...:prd-87`  
4. Open **Lens** → same Deployment → you **see** `prd-87`  

Argo CD does **not** “push an image” and does **not** edit Lens. It updates Kubernetes; Lens shows the result.

**After Argo manages the app:** do **not** keep changing that Deployment’s image by hand in Lens. With `selfHeal: true`, Argo will overwrite manual edits to match Git. Change the tag in **GitOps** (or roll back in Argo UI) instead.

---

## 5. Access / ownership (important for install)

| Action | Who |
|---|---|
| Create namespace `argocd` | **Cluster admin** (we do **not** have namespace-create) |
| Install Argo CD (CRDs, ClusterRoles) | **Cluster admin** (or grant one-time rights) |
| Day-2: edit Deployments, Secrets, ConfigMaps in app namespaces | **We have access** |
| Day-2: manage Argo Applications (after install + RBAC) | Us + admin agreement |
| Create / write GitOps repo + update `prodacr.yml` | Us / GitHub owners |
| FortiClient VPN | Required for any laptop `kubectl` / Argo UI port-forward to PRD |

**Ask admin once:**

1. Create namespace `argocd`  
2. Install Argo CD (official manifest or Helm) **or** grant temporary rights to apply install (needs CRDs + ClusterRoles)  
3. Allow Argo CD service account to manage `middleware-prd-ns`  
4. Grant our group access to Argo UI/CLI  

After that, we can operate with existing rights (Deployments / Secrets / ConfigMaps) plus Argo.

---

## 6. Pilot example — `lmb_statement_generator`

### 6.1 Current state (production branch + live cluster)

**Repo:** `lmb_statement_generator`  
**Branch:** `production`  
**Workflow today:** `.github/workflows/prodacr.yml`

```yaml
# TODAY (mutable) — production branch
on:
  push:
    branches:
      - production

# builds with Dockerfile.Dev and pushes:
#   pddlmbcr.azurecr.io/statementgenerator:prd
```

**Live Deployment (from prod backup):**

| Field | Value |
|---|---|
| Namespace | `middleware-prd-ns` |
| Deployment | `statementgenerator` |
| Helm release | `lmb-statementgenerator` |
| Image | `pddlmbcr.azurecr.io/statementgenerator:prd` |
| imagePullPolicy | `Always` |
| Port | `6103` |

### 6.2 Target state

| Item | Target |
|---|---|
| Immutable ACR tag | `pddlmbcr.azurecr.io/statementgenerator:prd-<github.run_number>` |
| Example | `pddlmbcr.azurecr.io/statementgenerator:prd-87` |
| Optional floating tag | `...:prd` (pointer only — **not** used in Deployment) |
| GitOps path | `lmb-gitops/apps/middleware-prd/statementgenerator/values.yaml` |
| Deployment image | Must come from GitOps tag via Argo CD |
| imagePullPolicy | `IfNotPresent` (safe with immutable tags) |

---

## 7. Concrete changes for the pilot

### 7.1 New GitOps repo: `lmb-gitops`

```text
lmb-gitops/
  apps/
    middleware-prd/
      statementgenerator/
        values.yaml          # image.repository + image.tag
        # chart or kustomize — match how Helm is managed today
  argocd/
    application-statementgenerator.yaml
```

**Example `values.yaml`:**

```yaml
image:
  repository: pddlmbcr.azurecr.io/statementgenerator
  tag: prd-87                    # written by CI after successful push
  pullPolicy: IfNotPresent
```

**Do not put secrets into this repo.** Keep using existing K8s Secrets / ConfigMaps (we can create/update those).

### 7.2 Update `prodacr.yml` in `lmb_statement_generator` (production)

Conceptual workflow (team to implement when approved):

```yaml
name: prod acr

on:
  push:
    branches:
      - production

concurrency:
  group: deploy-production-acr-${{ github.ref_name }}
  cancel-in-progress: true

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Login to ACR
        run: |
          echo "${{ secrets.ACR_PASSWORD }}" | docker login pddlmbcr.azurecr.io \
            -u "${{ secrets.ACR_USERNAME }}" --password-stdin

      - name: Build and push immutable tag
        env:
          IMAGE: pddlmbcr.azurecr.io/statementgenerator
          TAG: prd-${{ github.run_number }}
        run: |
          docker build -f Dockerfile.Dev -t "${IMAGE}:${TAG}" -t "${IMAGE}:prd" .
          docker push "${IMAGE}:${TAG}"
          docker push "${IMAGE}:prd"
          echo "Pushed ${IMAGE}:${TAG}"

      # Only after successful push — update GitOps (Option 1)
      - name: Update GitOps image tag
        if: success()
        env:
          GITOPS_TOKEN: ${{ secrets.GITOPS_TOKEN }}
          TAG: prd-${{ github.run_number }}
        run: |
          git clone "https://x-access-token:${GITOPS_TOKEN}@github.com/<ORG>/lmb-gitops.git"
          cd lmb-gitops
          # prefer yq in real implementation
          sed -i "s/^  tag: .*/  tag: ${TAG}/" \
            apps/middleware-prd/statementgenerator/values.yaml
          git config user.name "lmb-ci-bot"
          git config user.email "ci-bot@users.noreply.github.com"
          git add apps/middleware-prd/statementgenerator/values.yaml
          git commit -m "prod(statementgenerator): ${TAG}" || echo "No change"
          git push
```

**Secrets needed in the app repo:** existing `ACR_*` + new `GITOPS_TOKEN` (PAT/GitHub App with write to `lmb-gitops`).

**Safer prod variant:** CI opens a PR to `lmb-gitops` instead of direct push; merge = approve deploy. Start with manual Argo sync if preferred.

### 7.3 Argo CD Application (created in `argocd` ns after admin install)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: lmb-statementgenerator
  namespace: argocd
spec:
  project: default   # or dedicated AppProject middleware-prd
  source:
    repoURL: https://github.com/<ORG>/lmb-gitops.git
    targetRevision: main
    path: apps/middleware-prd/statementgenerator
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: middleware-prd-ns
  syncPolicy:
    automated:
      prune: false      # keep false for pilot
      selfHeal: true
    syncOptions:
      - CreateNamespace=false
```

### 7.4 What the team will see end-to-end (example run)

1. Developer merges to `production` on `lmb_statement_generator`.  
2. Actions run #87 succeeds.  
3. ACR shows: `pddlmbcr.azurecr.io/statementgenerator:prd-87` (and optionally `:prd`).  
4. CI updates `lmb-gitops` → `tag: prd-87`.  
5. Argo CD syncs → Deployment `statementgenerator` image becomes `...:prd-87`.  
6. Pods roll; nodes pull `prd-87` from ACR.  
7. In **Lens**, open Deployment `statementgenerator` → image shows `...:prd-87` (same object Argo updated).  

**Rollback:** set GitOps tag back to `prd-86` (or Argo History → Rollback) → sync → pods run build 86 again (tag must still exist in ACR).

---

## 8. Why not Image Updater (Option 2) for now?

| | Option 1 (CI → GitOps) | Option 2 (Image Updater) |
|---|---|---|
| Who writes the tag into Git? | GitHub Actions after green build | Separate Image Updater watching ACR |
| Clear “CI passed ⇒ this tag”? | Yes | Indirect (ACR tag appears) |
| Extra cluster component? | No | Yes |
| Best for first pilot? | **Yes** | Later, if many repos |

**Decision: Option 1 for `lmb_statement_generator`, then reuse the same pattern for the other services.**

---

## 9. Other services (same pattern later)

| App repo | ACR image (today `:prd`) |
|---|---|
| `lmb_super_admin` | `backofficeapp:prd` |
| `lmb_documents` | `documents:prd` |
| `lmb_notifications` | `notifications:prd` |
| `lmb_business_management` | `businessmanagement:prd` |
| `lmb_corporate_app` | `corporateapp:prd` |
| `lmb_settings` | `settings:prd` |
| **`lmb_statement_generator` (pilot)** | **`statementgenerator:prd`** |
| `lmb_onboarding_app` | `onboardingapp:prd` |
| `lmb_integrations` | `integrations:prd` |
| `lmb_migrations` | `migrations:prd` |
| `lmb_processing_scripts` | `processingscripts:prd` |
| `lmb_kafka_connector` | `kafkaconnector:prd` |

**Note:** `lmb_processing_scripts` and `lmb_kafka_connector` currently trigger on **PR → production** (pipeline runs on PR open). Align them to **push → production** when migrating, same as statement generator.

All live under Helm in **`middleware-prd-ns`**.

---

## 10. Rollout steps (team checklist)

### Phase 0 — Admin / access

- [ ] FortiClient VPN documented for PRD kubectl  
- [ ] Admin creates namespace `argocd`  
- [ ] Admin installs Argo CD (CRDs + control plane)  
- [ ] Argo CD can manage `middleware-prd-ns`  
- [ ] Our team can use Argo UI via **VPN + port-forward** (or Lens forward) + existing Deploy/Secret/CM rights  
- [ ] Confirm **Option 1** (not Image Updater) for pilot

### Phase 1 — GitOps + pilot wiring

- [ ] Create `lmb-gitops` repo  
- [ ] Export current Helm values for `lmb-statementgenerator`  
- [ ] Add `statementgenerator` path + Argo Application  
- [ ] Add `GITOPS_TOKEN` secret to `lmb_statement_generator`  

### Phase 2 — CI tag change (statement generator only)

- [ ] Change `prodacr.yml` to push `prd-${{ github.run_number }}`  
- [ ] Add GitOps update step on success  
- [ ] Keep floating `:prd` optional; **Deployment must use `prd-<n>` only**  

### Phase 3 — Validate

- [ ] Confirm ACR has `statementgenerator:prd-<n>`  
- [ ] Confirm GitOps `tag:` matches  
- [ ] Confirm Deployment image matches (kubectl **and** Lens)  
- [ ] **Rollback test:** previous `prd-<n-1>` restores old pods  

### Phase 4 — Expand

- [ ] Repeat for remaining repos  
- [ ] Fix PR triggers on kafka / processing_scripts  
- [ ] Enable auto-sync carefully; keep `prune: false` until stable  

---

## 11. FAQ

**Q: Option 1 or Option 2?**  
A: **Option 1** for this project (CI → GitOps → Argo). Option 2 (Image Updater) later only if needed.

**Q: How do we open Argo UI if we only have FortiClient VPN?**  
A: Argo runs in-cluster. Use `kubectl -n argocd port-forward svc/argocd-server 8080:443` (or Lens port-forward), then `https://localhost:8080`. No public expose required for PRD.

**Q: Is Argo installed on my laptop?**  
A: No. Only the UI/CLI talk to the in-cluster Argo over VPN.

**Q: Will Argo update the same Deployments we see in Lens?**  
A: **Yes — that is the requirement.** Argo syncs the live Kubernetes Deployment; Lens shows the updated image/tag.

**Q: Should we still edit the image in Lens after Argo owns the app?**  
A: No. Change GitOps (or Argo rollback). Manual Lens image edits will fight `selfHeal`.

**Q: Does Argo CD pick the latest image from ACR automatically?**  
A: No. Option 1: **CI writes the new tag into GitOps**. Argo CD syncs Git → Deployment.

**Q: Does Argo CD push images?**  
A: No. CI pushes to ACR. Argo CD only updates the Deployment spec. Nodes pull from ACR.

**Q: Can we install Argo CD without creating a namespace?**  
A: Namespace must exist first (admin). We can manage Deployments/Secrets/ConfigMaps; we still need admin for `argocd` ns + CRDs/ClusterRoles.

**Q: Is `statementgenerator:prd:87` valid?**  
A: No. Use `statementgenerator:prd-87`.

**Q: Why `github.run_number`?**  
A: Stable, immutable per workflow run; easy to read in ACR and GitOps. Optional later: `prd-<run_number>-<shortsha>`.

---

## 12. Ask for team agreement

1. Adopt **Option 1** (CI → GitOps → Argo CD) — **not** Image Updater for the pilot.  
2. Pilot on **`lmb_statement_generator`**: move from `:prd` to `:prd-<github.run_number>`.  
3. Admin creates **`argocd`** namespace and installs Argo CD.  
4. Team accesses Argo via **FortiClient VPN + local port-forward** (Lens optional).  
5. Argo updates the **same live Deployments** visible in Lens; GitOps becomes source of truth for image tags.  
6. Create **`lmb-gitops`** as the only source of truth for prod image tags.  
7. After successful pilot + rollback test, roll out to the other LMB services.

---

## 13. Reference — current statement generator workflow (production)

```yaml
name: prod acr
on:
  push:
    branches:
      - production
# ...
# docker build -f Dockerfile.Dev -t pddlmbcr.azurecr.io/statementgenerator:prd .
# docker push pddlmbcr.azurecr.io/statementgenerator:prd
```

**Target image examples after change:**

```text
pddlmbcr.azurecr.io/statementgenerator:prd-87
pddlmbcr.azurecr.io/statementgenerator:prd-88
```

Registry remains: **`pddlmbcr.azurecr.io`**.  
Auth remains: **`ACR_USERNAME` / `ACR_PASSWORD`**.
