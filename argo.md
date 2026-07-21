# Argo CD + Immutable Tags — LMB Production

**Example repo:** `lmb_statement_generator`  
**Approach:** GitHub Actions pushes `prd-<github.run_number>` to ACR → updates **GitOps repo** → **Argo CD** syncs Deployment in `middleware-prd-ns`  

---

## 1. Problem

Today CI pushes mutable `pddlmbcr.azurecr.io/statementgenerator:prd`. Deployment uses that tag with `imagePullPolicy: Always`, so rollback cannot restore the old image.

**Fix:** immutable tags `...:prd-<run_number>` + GitOps + Argo CD.

---

## 2. Flow

```text
Merge to production
  → GitHub Actions (prodacr.yml)
      1. Push ACR: statementgenerator:prd-<github.run_number>
      2. On success → update GitOps values.yaml  image.tag: prd-<run_number>
  → Argo CD watches GitOps
  → Updates Deployment in middleware-prd-ns
  → Nodes pull image from ACR
```

Argo CD runs **in-cluster** (`argocd` ns). It does not watch GitHub Actions; it syncs Git → Kubernetes.

---

## 3. Admin-only — install Argo CD

App team **cannot** create namespaces / CRDs / ClusterRoles. Admin must run this on PRD.

```bash
# 0) Confirm PRD context
kubectl config current-context
kubectl cluster-info

# 1) Create namespace
kubectl create namespace argocd

# 2) Install Argo CD (CRDs + ClusterRoles + control plane)
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 3) Wait until ready
kubectl -n argocd get pods
kubectl -n argocd rollout status deployment/argocd-server --timeout=300s
kubectl -n argocd rollout status deployment/argocd-repo-server --timeout=300s
kubectl -n argocd rollout status statefulset/argocd-application-controller --timeout=300s

# 4) Verify Argo can manage middleware-prd-ns
kubectl auth can-i create deployments -n middleware-prd-ns \
  --as=system:serviceaccount:argocd:argocd-application-controller
kubectl auth can-i update deployments -n middleware-prd-ns \
  --as=system:serviceaccount:argocd:argocd-application-controller
kubectl auth can-i patch deployments -n middleware-prd-ns \
  --as=system:serviceaccount:argocd:argocd-application-controller
# Expect: yes

# 5) Grant  access to  me/sunil for argocd ns (replace placeholders)
kubectl create rolebinding argocd-team-admin \
  --clusterrole=admin \
  --user=<USER_EMAIL_OR_NAME> \
  -n argocd


# 6) Share initial admin password securely
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```
**Admin checklist**

- [ ] `argocd` namespace created  
- [ ] Install applied; pods Ready  
- [ ] Argo SA can update Deployments in `middleware-prd-ns`  
- [ ] App team RoleBinding in `argocd`  
- [ ] Admin password shared  

---

## 4. Example Repo — `lmb_statement_generator`

### Today (production)

| Item | Value |
|---|---|
| Workflow | `.github/workflows/prodacr.yml` on push → `production` |
| Image | `pddlmbcr.azurecr.io/statementgenerator:prd` |
| Namespace | `middleware-prd-ns` |
| Deployment | `statementgenerator` |
| Helm release | `lmb-statementgenerator` |

### Target

| Item | Value |
|---|---|
| Image | `pddlmbcr.azurecr.io/statementgenerator:prd-<github.run_number>` |
| Example | `...:prd-87` |
| GitOps | `lmb-gitops/apps/middleware-prd/statementgenerator/values.yaml` |

### GitOps repo (`lmb-gitops`)

```text
lmb-gitops/
  apps/middleware-prd/statementgenerator/values.yaml
  argocd/application-statementgenerator.yaml
```

```yaml
# values.yaml
image:
  repository: pddlmbcr.azurecr.io/statementgenerator
  tag: prd-87
  pullPolicy: IfNotPresent
```

Do not commit secrets. Keep existing K8s Secrets/ConfigMaps.

### `prodacr.yml` (after approval)

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

      - name: Update GitOps image tag
        if: success()
        env:
          GITOPS_TOKEN: ${{ secrets.GITOPS_TOKEN }}
          TAG: prd-${{ github.run_number }}
        run: |
          git clone "https://x-access-token:${GITOPS_TOKEN}@github.com/<ORG>/lmb-gitops.git"
          cd lmb-gitops
          sed -i "s/^  tag: .*/  tag: ${TAG}/" \
            apps/middleware-prd/statementgenerator/values.yaml
          git config user.name "lmb-ci-bot"
          git config user.email "ci-bot@users.noreply.github.com"
          git add apps/middleware-prd/statementgenerator/values.yaml
          git commit -m "prod(statementgenerator): ${TAG}" || echo "No change"
          git push
```

Secrets: existing `ACR_*` + `GITOPS_TOKEN` (write to `lmb-gitops`).

### Argo CD Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: lmb-statementgenerator
  namespace: argocd
spec:
  project: default
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
      prune: false
      selfHeal: true
    syncOptions:
      - CreateNamespace=false
```

### End-to-end check

1. Merge to `production` → Actions run #87 succeeds  
2. ACR has `statementgenerator:prd-87`  
3. GitOps `tag: prd-87`  
4. Argo syncs → Deployment image = `...:prd-87`  
5. Rollback: set GitOps tag to previous `prd-<n>` → sync  

After Argo owns the app, do not change the image manually in Lens; change GitOps or use Argo rollback.

---

## 5. Later (same pattern)

| Repo | Image today |
|---|---|
| `lmb_super_admin` | `backofficeapp:prd` |
| `lmb_documents` | `documents:prd` |
| `lmb_notifications` | `notifications:prd` |
| `lmb_business_management` | `businessmanagement:prd` |
| `lmb_corporate_app` | `corporateapp:prd` |
| `lmb_settings` | `settings:prd` |
| `lmb_statement_generator` | `statementgenerator:prd` **(pilot)** |
| `lmb_onboarding_app` | `onboardingapp:prd` |
| `lmb_integrations` | `integrations:prd` |
| `lmb_migrations` | `migrations:prd` |
| `lmb_processing_scripts` | `processingscripts:prd` |
| `lmb_kafka_connector` | `kafkaconnector:prd` |

---
