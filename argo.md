# Argo CD Implementation Guide — Lulu Middleware (LMB)

**Scope:** Production ACR + Kubernetes deployments for LMB services  
**Local code reviewed (read-only):** `C:\Users\nxtge\Downloads\Lulu-Repos`  
**Prod manifests referenced (read-only):** `C:\Users\nxtge\Downloads\k8s\prd-backup-20260717-113423`  
**Cluster access:** Connected to the **Kubernetes PRD cluster** after connecting **FortiClient VPN** (required for `kubectl` / Argo CD install & day-2 ops from your machine)  
**Date:** 2026-07-21  

**Important:** This document is guidance only. No GitHub branches or workflows were modified.

---

## 1. Direct answers

### 1.1 Can Argo CD work if someone else creates the namespace and you install into it?

**Short answer: partially yes — but a namespace alone is not enough for a normal cluster-wide Argo CD install.**

| What you need | Why |
|---|---|
| Namespace `argocd` (or similar) | Where Argo CD pods/Services run |
| Permission to create **CRDs** (`applications.argoproj.io`, etc.) | Argo CD Application CRDs are cluster-scoped |
| Permission to create **ClusterRoles / ClusterRoleBindings** | Default Argo CD watches/manages other namespaces |
| Permission to create Deployments/Services/Secrets/ConfigMaps **in** `argocd` | Install the control plane |
| Ability for Argo CD SA to manage target ns (e.g. `middleware-prd-ns`) | Sync apps into prod |

**Practical options:**

1. **Recommended (full install):** Cluster admin creates `argocd` **and** either:
   - installs Argo CD for you, **or**
   - grants you temporary rights to apply the official install manifest (CRDs + ClusterRoles), then you operate day-to-day inside `argocd`.

2. **Namespace-scoped Argo CD:** Possible if admin creates the namespace, installs/allows CRDs once, and configures Argo CD with `--namespaced` / Application controller scoped to specific namespaces. More limited (harder multi-ns / AppProject patterns).

3. **Will NOT work:** Only creating empty `argocd` namespace while your RBAC is limited to that namespace and you **cannot** create CRDs / ClusterRoles. `kubectl apply -n argocd -f install.yaml` will fail on cluster-scoped objects.

**Ask your cluster admin for one of these packages:**

```text
A) Full: create ns argocd + apply official Argo CD install (or give you ClusterRole to do it once)
B) Scoped: create ns argocd + install CRDs cluster-wide + ClusterRole limited to middleware-prd-ns + argocd
C) Least privilege day-2: after install, your user only needs access to argocd UI/CLI + gitops repo
```

### 1.2 Is `website:prd:<incremental>` a valid tag?

**No.** Docker/OCI image references allow **one** tag after the last colon in the name (ignoring registry port).  

Invalid: `myacr.azurecr.io/website:prd:001`  
Valid: `myacr.azurecr.io/website:prd-001` or `website:prd-20260721-a1b2c3d` or digest `website@sha256:...`

Saikrishna’s suggestion (`website:prd-001`) is correct. Prefer **immutable** tags (git SHA or build number), not only increments.

### 1.3 Does Argo CD alone fix rollback?

**No.** Argo CD rollbacks Git desired state. If Git still says `image: ...:prd` and ACR `:prd` was overwritten, sync/rollback still deploys the **latest** digest behind `:prd` (especially with `imagePullPolicy: Always`, which your prod Deployments use today).

**You need both:**

1. Immutable image tags (or digests) in Git  
2. Argo CD (or Helm) deploying that exact tag from Git  

---

## 2. Read-only review of current production flow

### 2.1 What `prodacr.yml` does today

Across the 12 repos on branch **`production`**, GitHub Actions only **build + push** to ACR. They do **not** apply Kubernetes manifests.

| Repo | Workflow file | Trigger | Image pushed |
|---|---|---|---|
| `lmb_super_admin` | `prodacr.yml` | push → `production` | `pddlmbcr.azurecr.io/backofficeapp:prd` |
| `lmb_documents` | `prodacr.yml` | push → `production` | `pddlmbcr.azurecr.io/documents:prd` |
| `lmb_notifications` | `prodacr.yml` | push → `production` | `pddlmbcr.azurecr.io/notifications:prd` |
| `lmb_business_management` | `prodacr.yml` | push → `production` + `workflow_dispatch` | `pddlmbcr.azurecr.io/businessmanagement:prd` |
| `lmb_corporate_app` | `prodacr.yml` | push → `production` | `pddlmbcr.azurecr.io/corporateapp:prd` |
| `lmb_settings` | `prodacr.yml` | push → `production` | `pddlmbcr.azurecr.io/settings:prd` |
| `lmb_statement_generator` | `prodacr.yml` | push → `production` | `pddlmbcr.azurecr.io/statementgenerator:prd` |
| `lmb_onboarding_app` | `prodacr.yml` | push → `production` | `pddlmbcr.azurecr.io/onboardingapp:prd` |
| `lmb_integrations` | `prodacr.yml` | push → `production` + `workflow_dispatch` | `pddlmbcr.azurecr.io/integrations:prd` |
| `lmb_migrations` | `Prodacr.yml` | push → `production` | `pddlmbcr.azurecr.io/migrations:prd` |
| `lmb_processing_scripts` | `prodacr.yml` | **pull_request → `production`** + dispatch | `pddlmbcr.azurecr.io/processingscripts:prd` |
| `lmb_kafka_connector` | `prodacr.yml` | **pull_request → `production`** | `pddlmbcr.azurecr.io/kafkaconnector:prd` |

**Common patterns:**

- Registry: `pddlmbcr.azurecr.io`
- Auth: `secrets.ACR_USERNAME` / `secrets.ACR_PASSWORD`
- Dockerfile: almost always **`Dockerfile.Dev`** for production builds
- Tag: always mutable **`:prd`**
- Some repos notify Teams on success/failure (`lmb_super_admin`, `lmb_documents`)

### 2.2 How workloads run in the cluster (from prod backup)

- Namespace: **`middleware-prd-ns`**
- Mostly **Helm-managed** (`app.kubernetes.io/managed-by: Helm`, `meta.helm.sh/release-name: lmb-*`)
- Images: same `:prd` tags as CI
- **`imagePullPolicy: Always`** → every new pod pulls whatever `:prd` points to **now**

Example mapping (apps ↔ images):

| Helm release / Deployment | Image |
|---|---|
| `lmb-backoffice-app` / `backoffice-app` | `.../backofficeapp:prd` |
| `lmb-documents` / `documents` | `.../documents:prd` |
| `lmb-notifications` / `notifications` | `.../notifications:prd` |
| `lmb-businessmanagement` | `.../businessmanagement:prd` |
| `lmb-corporateapp` | `.../corporateapp:prd` |
| `lmb-settings` | `.../settings:prd` |
| `lmb-statementgenerator` | `.../statementgenerator:prd` |
| `lmb-onboarding` | `.../onboardingapp:prd` |
| `lmb-integrations` | `.../integrations:prd` |
| `lmb-migrations` | `.../migrations:prd` |
| `lmb-processingscripts` | `.../processingscripts:prd` |
| `lmb-kafkaconnector` | `.../kafkaconnector:prd` |

Also present in cluster (not in your 12-repo list): website, backoffice web, corporate web, onboarding web, logger, cardsapplications, serviceapplications.

### 2.3 Verdict: is it good to move to Argo CD?

**Yes — with prerequisites. Do not “just install Argo CD” on top of mutable `:prd` tags.**

| Ready today? | Item |
|---|---|
| Yes | Clear list of services, ACR, prod namespace, Helm already in use |
| Yes | CI already centralized on `production` branch |
| **No** | Immutable tags / digests in deploy specs |
| **No** | Single GitOps repo (or chart repo) as source of truth for prod desired state |
| **No** | Consistent CI triggers (PR vs push to production) |
| Risk | Secrets currently appear in plain Deployment env (must move to Sealed Secrets / External Secrets before GitOps) |
| Risk | Prod builds use `Dockerfile.Dev`; prod Deployments even show odd env like `ENVIRONMENT: STAGING` on backoffice — clean before declaring GitOps “source of truth” |

**Recommendation:** Move to Argo CD **after** (or in parallel with) immutable tagging + a dedicated GitOps repo. Argo CD then gives real rollback, audit trail, and controlled sync. Without that, you only move the same broken rollback into a nicer UI.

---

## 3. Target architecture

```text
Developer PR → merge to production
        │
        ▼
GitHub Actions (prodacr.yml)
  - build Dockerfile (prefer Dockerfile.Prod later)
  - push TWO tags:
      pddlmbcr.azurecr.io/<svc>:prd-<sha>
      pddlmbcr.azurecr.io/<svc>:prd          (optional floating pointer)
  - optionally commit/PR image tag into gitops repo
        │
        ▼
GitOps repo (e.g. lmb-gitops)
  apps/middleware-prd/<svc>/values.yaml
    image.tag: prd-<sha>
        │
        ▼
Argo CD Application (auto or manual sync)
        │
        ▼
Helm release in middleware-prd-ns
```

**Rollback:** revert GitOps commit (previous `prd-<sha>`) → Argo CD sync → Kubernetes pulls the **old immutable** image. Optional: keep floating `:prd` for humans, but **never** deploy only `:prd` from Git.

---

## 4. Permissions checklist for your admin

Ask for:

1. Namespace `argocd` created  
2. Argo CD installed **or** rights to apply [official install](https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml)  
3. Your kubeconfig / AAD group can:
   - get/list pods, logs, secrets in `argocd`
   - create Application / AppProject CRs
4. Argo CD service account can manage `middleware-prd-ns` (and optionally staging)  
5. Network: Argo CD → GitHub (HTTPS or SSH), Argo CD → ACR pull via cluster imagePullSecrets (already expected today)  
6. Optional: Ingress / LoadBalancer + Azure AD OIDC for Argo CD UI  

Minimum ClusterRole ideas (admin to refine):

```yaml
# Illustrative only — admin should tighten
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: argocd-application-controller-middleware
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]
# Prefer scoping to middleware-prd-ns via Role + RoleBinding instead of ClusterRole where possible
```

For **namespace-scoped** Argo CD, admin still must install CRDs once at cluster scope.

---

## 5. Step-by-step: install Argo CD (when you have rights)

### 5.1 Prerequisites

**Network:** You must be on **FortiClient VPN** before talking to the PRD cluster. Without VPN, `kubectl`, Helm, and Argo CD CLI/port-forward from your laptop will fail or time out.

```bash
# 1) Connect FortiClient VPN first, then:
kubectl version --client
helm version   # optional if using Helm chart
# Confirm context points to PRD AKS (or correct cluster)
kubectl config current-context
kubectl get ns middleware-prd-ns
kubectl get nodes   # sanity check: VPN + kubeconfig both OK
```

If `kubectl get nodes` hangs or returns connection errors, reconnect FortiClient and re-check kubeconfig context before installing Argo CD.

### 5.2 Create namespace (if not done)

```bash
kubectl create namespace argocd
```

### 5.3 Install (official manifests)

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd rollout status deployment/argocd-server
```

Or Helm:

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm upgrade --install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --set configs.params."server\.insecure"=false
```

### 5.4 Access UI (temporary port-forward)

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
echo
kubectl -n argocd port-forward svc/argocd-server 8080:443
# Open https://localhost:8080  user: admin
```

Install CLI: https://argo-cd.readthedocs.io/en/stable/cli_installation/

```bash
argocd login localhost:8080 --username admin
```

### 5.5 Register cluster (usually in-cluster is automatic)

```bash
argocd cluster list
```

---

## 6. Create a GitOps repository (required)

Suggested new repo (org-level): `lmb-gitops` (or `lmb-k8s-prd`).

```text
lmb-gitops/
  apps/
    middleware-prd/
      backofficeapp/
        Chart.yaml          # or use umbrella + dependency
        values.yaml         # image.repository / image.tag
      documents/
      notifications/
      businessmanagement/
      corporateapp/
      settings/
      statementgenerator/
      onboardingapp/
      integrations/
      migrations/
      processingscripts/
      kafkaconnector/
  projects/
    middleware-prd.yaml     # AppProject
  root/
    middleware-prd-app-of-apps.yaml
```

**Do not commit raw secrets** from the prod backup (integrations currently has plaintext API secrets in Deployment env). Use:

- Existing K8s Secrets already in cluster + `ignoreDifferences` / not managed by Argo, **or**
- External Secrets Operator / Sealed Secrets / Azure Key Vault CSI  

Start by exporting **Helm charts** you already use for `lmb-*` releases (source of truth today). If charts live only on someone’s laptop, recover them from Helm:

```bash
helm list -n middleware-prd-ns
helm get values lmb-integrations -n middleware-prd-ns
helm get manifest lmb-integrations -n middleware-prd-ns
```

---

## 7. Example Argo CD Application (one service)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: lmb-integrations
  namespace: argocd
spec:
  project: middleware-prd
  source:
    repoURL: https://github.com/<org>/lmb-gitops.git
    targetRevision: main
    path: apps/middleware-prd/integrations
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: middleware-prd-ns
  syncPolicy:
    automated:
      prune: false          # start false in prod
      selfHeal: true
    syncOptions:
      - CreateNamespace=false
```

App-of-apps pattern: one root Application that points at a folder of Application manifests.

---

## 8. Change GitHub Actions for immutable tags (all 12 repos)

**Do this in a planned change window — not done in this review.**

Recommended tag scheme:

```text
IMAGE=pddlmbcr.azurecr.io/<repo-name>
TAG_IMMUTABLE=prd-${GITHUB_SHA::7}          # or prd-${{ github.run_number }}
TAG_FLOATING=prd                            # optional

docker build -f Dockerfile.Dev -t $IMAGE:$TAG_IMMUTABLE -t $IMAGE:$TAG_FLOATING .
docker push $IMAGE:$TAG_IMMUTABLE
docker push $IMAGE:$TAG_FLOATING
```

Then update GitOps `values.yaml`:

```yaml
image:
  repository: pddlmbcr.azurecr.io/integrations
  tag: prd-a1b2c3d
  pullPolicy: IfNotPresent   # safe with immutable tags
```

Ways to update GitOps:

1. **Manual PR** into `lmb-gitops` (safest start)  
2. **CI bot** opens PR / commits tag after successful ACR push (Image Updater or custom step)  
3. **Argo CD Image Updater** watches ACR and writes tags back to Git  

Align triggers while you are there:

- Prefer **`push` to `production`** (not PR) for ACR publish — today `lmb_kafka_connector` and `lmb_processing_scripts` publish on PR, which can push “prod” images before merge.

---

## 9. Migration plan (low risk)

### Phase 0 — Access (this week)

1. Confirm **FortiClient VPN** is connected whenever you run `kubectl` / Argo CD against PRD  
2. Confirm admin creates `argocd` + install rights (section 4)  
3. Confirm who owns Helm charts for `lmb-*`  
4. Inventory secrets: which must not enter Git  

### Phase 1 — Install Argo CD (non-prod first if possible)

1. Install on **staging** cluster or a sandbox namespace policy first  
2. Deploy **one** non-critical app with Argo CD in parallel (observe only / manual sync)  
3. Compare live Helm release vs Argo desired state  

### Phase 2 — Immutable tags (CI)

1. Update one repo’s `prodacr.yml` to push `prd-<sha>` + keep `:prd`  
2. Manually set Deployment/Helm values to `prd-<sha>` once  
3. Test rollback: change tag back to previous SHA → pods return to old build  

### Phase 3 — GitOps + Argo for all middleware apps

1. Import charts/values into `lmb-gitops`  
2. Create AppProject `middleware-prd`  
3. Onboard services one-by-one; leave `prune: false` initially  
4. Disable conflicting manual Helm upgrades / Lens edits for onboarded apps  

### Phase 4 — Harden

1. Automated sync + selfHeal  
2. RBAC / SSO on Argo UI  
3. Notifications (Teams) from Argo instead of only CI  
4. Remove reliance on floating `:prd` in Deployments  
5. Switch builds from `Dockerfile.Dev` to a real prod Dockerfile when ready  

---

## 10. Rollback runbook (after immutable tags + Argo)

```bash
# Find previous good tag in gitops history
cd lmb-gitops
git log -p -- apps/middleware-prd/integrations/values.yaml

# Revert values to previous tag (or revert commit)
git revert <bad-commit>
git push

# Sync
argocd app sync lmb-integrations
argocd app wait lmb-integrations --health
```

Or in UI: Application → History → Rollback → Sync.

This works **only** if the old tag still exists in ACR and is immutable.

---

## 11. Suggested reply to the email thread (optional)

> The rollback concern is valid: mutable `:prd` plus `imagePullPolicy: Always` means Deployment revision rollback cannot restore the previous image digest.  
> Tag format `website:prd:001` is invalid; use `website:prd-001` or better `website:prd-<gitsha>`.  
> We should (1) push immutable tags from `prodacr.yml`, (2) store that tag in a GitOps repo, (3) let Argo CD sync Helm releases in `middleware-prd-ns`.  
> Argo CD install needs more than an empty namespace (CRDs + cluster RBAC). Please create `argocd` and either install Argo CD or grant rights to apply the official manifest.  
> I will not change production branches until we agree on tag scheme + GitOps ownership.

---

## 12. What was / was not done in this review

**Done (read-only):**

- Inspected `production` branch `prodacr.yml` / `Prodacr.yml` for all 12 listed repos under `Downloads/Lulu-Repos`
- Correlated with prod Deployment backup under `Downloads/k8s/prd-backup-20260717-113423`
- Wrote this guide to `Downloads/argo.md`

**Not done (per your request):**

- No commits, pushes, or edits to any GitHub repo / branch  
- No cluster changes  
- No workflow updates yet  

---

## 13. Immediate next actions for you

1. Use **FortiClient VPN** whenever accessing the PRD cluster (`kubectl`, Argo install, UI port-forward)  
2. Forward **section 4** permissions ask to cluster admin  
3. Confirm ownership of Helm charts for `middleware-prd-ns`  
4. Agree tag format: `prd-<7-char-sha>` (preferred) or `prd-<run_number>`  
5. Create empty `lmb-gitops` repo (when approved)  
6. Pilot: one service (e.g. `settings` or `statementgenerator`) on staging/prod with Argo + immutable tag  
7. Only after pilot success: roll out remaining services + update all `prodacr.yml` files  

If you want, next session can draft the exact `prodacr.yml` patch and sample GitOps values **as files under Downloads only** (still without touching any repo branch).
