#  SSL Truststore Workaround

## Problem

Node apps calling `https://emw.ruyabank.ae` fail with:

```text
unable to verify the first certificate; if the root CA is installed locally, try running Node.js with --use-system-ca
```

**Cause:** Ruya’s server sends only the leaf certificate, not the GlobalSign intermediate (`GlobalSign GCC R3 DV TLS CA 2020`).

**Proper fix:** Ruya should serve the full TLS chain.

**Workaround (our side):** Add the missing intermediate via `NODE_EXTRA_CA_CERTS` on the `integrations` deployment.

---

## Scope

| Item | Value |
|------|--------|
| Cluster context | `infra:PDD-LMB-PRD` |
| Namespace | `middleware-prd-ns` |
| Deployment | `integrations` |
| Container | `lmb-integrations` |
| Secret name | `ruya-ca-chain` |
| Secret / file key | `ruya_globalsign_chain.pem` |
| Mount path | `/etc/ssl/certs/custom` |
| Env var | `NODE_EXTRA_CA_CERTS=/etc/ssl/certs/custom/ruya_globalsign_chain.pem` |

---

## Step 1 — Download and convert the intermediate cert

```bash
curl -o gsgccr3dvtlsca2020.crt https://secure.globalsign.com/cacert/gsgccr3dvtlsca2020.crt
openssl x509 -inform DER -in gsgccr3dvtlsca2020.crt -out ruya_globalsign_chain.pem
```

Confirm the PEM starts with `-----BEGIN CERTIFICATE-----`.

---

## Step 2 — Create (or update) the Kubernetes Secret

```bash
kubectl create secret generic ruya-ca-chain \
  -n middleware-prd-ns \
  --from-file=ruya_globalsign_chain.pem=/Users/nxtge/Downloads/ruya_globalsign_chain.pem
```

If the Secret already exists and you need to replace the cert:

```bash
kubectl create secret generic ruya-ca-chain \
  -n middleware-prd-ns \
  --from-file=ruya_globalsign_chain.pem=/Users/nxtge/Downloads/ruya_globalsign_chain.pem \
  --dry-run=client -o yaml | kubectl apply -f -
```

Change the path on the right if the PEM is in another folder.  
Left side key name must stay: `ruya_globalsign_chain.pem`.

---

## Step 3 — Mount the Secret into `integrations`

**Skip if already present** (checked: volume + mount already exist on prod).

Check first:

```bash
kubectl get deploy integrations -n middleware-prd-ns \
  -o jsonpath='{range .spec.template.spec.volumes[*]}{.name} secret={.secret.secretName}{"\n"}{end}'

kubectl get deploy integrations -n middleware-prd-ns \
  -o jsonpath='{range .spec.template.spec.containers[0].volumeMounts[*]}{.name} -> {.mountPath}{"\n"}{end}'
```

Expected:

```text
custom-ca-cert secret=ruya-ca-chain
custom-ca-cert -> /etc/ssl/certs/custom
```

Only if missing, run:

```bash
kubectl patch deploy integrations -n middleware-prd-ns --type='json' -p='[
  {"op":"add","path":"/spec/template/spec/volumes/-","value":{"name":"custom-ca-cert","secret":{"secretName":"ruya-ca-chain"}}},
  {"op":"add","path":"/spec/template/spec/containers/0/volumeMounts/-","value":{"name":"custom-ca-cert","mountPath":"/etc/ssl/certs/custom","readOnly":true}}
]'
```

---

## Step 4 — Set `NODE_EXTRA_CA_CERTS`

**Skip if already present** (checked: already set on prod).

Check first:

```bash
kubectl get deploy integrations -n middleware-prd-ns \
  -o jsonpath='{range .spec.template.spec.containers[0].env[*]}{.name}={.value}{"\n"}{end}' \
  | grep NODE_EXTRA_CA_CERTS
```

Expected:

```text
NODE_EXTRA_CA_CERTS=/etc/ssl/certs/custom/ruya_globalsign_chain.pem
```

Only if missing, run:

```bash
kubectl set env deploy/integrations -n middleware-prd-ns \
  NODE_EXTRA_CA_CERTS=/etc/ssl/certs/custom/ruya_globalsign_chain.pem
```

---

## Step 5 — Restart (if Secret was created/updated) and verify

If you created or updated the Secret, restart so pods pick up the new data:

```bash
kubectl rollout restart deploy/integrations -n middleware-prd-ns
kubectl rollout status deploy/integrations -n middleware-prd-ns
kubectl get pods -n middleware-prd-ns | grep integrations
```

Replace `<POD>` with a running pod name:

```bash
kubectl exec -n middleware-prd-ns <POD> -- printenv NODE_EXTRA_CA_CERTS
kubectl exec -n middleware-prd-ns <POD> -- ls -la /etc/ssl/certs/custom
```

Expected:

```text
NODE_EXTRA_CA_CERTS=/etc/ssl/certs/custom/ruya_globalsign_chain.pem
... ruya_globalsign_chain.pem ...
```

Optional TLS smoke test from the pod:

```bash
kubectl exec -n middleware-prd-ns <POD> -- node -e "require('https').get('https://emw.ruyabank.ae/',r=>{console.log('OK',r.statusCode,'auth',r.socket.authorized);r.resume()}).on('error',e=>console.log('FAIL',e.message))"
```

Success looks like `OK ... auth true` (not `unable to verify the first certificate`).

---

## Current prod status (as of investigation)

- Secret `ruya-ca-chain` exists
- Volume mount `custom-ca-cert` → `/etc/ssl/certs/custom` exists
- `NODE_EXTRA_CA_CERTS` is already set

So typically you only need **Step 1 + Step 2 (update Secret if needed) + Step 5 (restart + verify)**.

---

## Notes

- Prefer applying this in Helm/GitOps if that manages `integrations`, or a live patch may be overwritten later.
- Do **not** set `HTTP_AGENT_REJECT_UNAUTHORIZED=false` in production.
- Long-term: ask Ruya to fix the full certificate chain on `emw.ruyabank.ae`.
