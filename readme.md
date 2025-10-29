# EKS CSR Demo with cert-manager (Teaching Lab)

This guide shows, step by step, how to issue a client certificate through Kubernetes CertificateSigningRequests (CSR) on Amazon EKS using cert-manager as the signer. Every step includes a short explanation and a dedicated Copy block for commands.

Important: These certs are for learning the CSR lifecycle. Managed EKS does not trust this lab CA for API authentication. Use IAM or ServiceAccount tokens for real access.

---

## 0) Reset (safe to run before starting)
Why: start from a clean slate so previous attempts don’t interfere.

Copy (Git Bash)
```bash
mkdir -p ./demo
rm -f ./demo/john.key ./demo/john.csr ./demo/john.crt ./demo/csr.yaml clusterissuer.yaml
kubectl delete csr john --ignore-not-found
kubectl delete csr john-2 --ignore-not-found
```

---

## 1) Install cert-manager with CSR controllers
Why: cert-manager’s CSR controllers must be enabled to sign Kubernetes CSRs.

Copy (Git Bash / any shell)
```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm upgrade --install cert-manager jetstack/cert-manager \
  -n cert-manager --create-namespace \
  --set installCRDs=true \
  --set extraArgs[0]=--feature-gates=ExperimentalCertificateSigningRequestControllers=true
```

Wait and verify deploys and feature-gate:

Copy
```bash
kubectl wait --for=condition=Established crd/issuers.cert-manager.io --timeout=300s
kubectl -n cert-manager rollout status deploy/cert-manager
kubectl -n cert-manager rollout status deploy/cert-manager-webhook
kubectl -n cert-manager rollout status deploy/cert-manager-cainjector
```

Copy
```bash
kubectl -n cert-manager get deploy cert-manager \
  -o jsonpath='{.spec.template.spec.containers[0].args}' | tr ' ' '\n' | \
  grep -i ExperimentalCertificateSigningRequestControllers || echo MISSING
```

If you see MISSING, patch and roll out:

Copy
```bash
kubectl -n cert-manager patch deploy cert-manager \
  --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--feature-gates=ExperimentalCertificateSigningRequestControllers=true"}]'
kubectl -n cert-manager rollout status deploy/cert-manager
```

---

## 2) Create a lab CA Secret
Why: cert-manager needs a CA to sign certs. For the lab, we bring our own CA keypair.

Copy (Git Bash)
```bash
MSYS_NO_PATHCONV=1 openssl req -x509 -newkey rsa:2048 -days 3650 -nodes \
  -subj "/CN=Workshop User CA" \
  -keyout ca.key -out ca.crt
```

Copy (PowerShell)
```powershell
& 'C:\\Program Files\\Git\\usr\\bin\\openssl.exe' req -x509 -newkey rsa:2048 -days 3650 -nodes -subj "/CN=Workshop User CA" -keyout .\\ca.key -out .\\ca.crt
```

Create the Secret in cert-manager namespace:

Copy
```bash
kubectl -n cert-manager create secret generic user-ca \
  --from-file=tls.crt=ca.crt --from-file=tls.key=ca.key 2>/dev/null || true
```

---

## 3) Create a ClusterIssuer (recommended)
Why: ClusterIssuer is cluster-scoped and avoids namespace ambiguity.

Copy
```bash
cat > clusterissuer.yaml <<'YAML'
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: user-ca-cluster
spec:
  ca:
    secretName: user-ca
YAML
kubectl apply -f clusterissuer.yaml
kubectl wait --for=condition=Ready clusterissuer/user-ca-cluster --timeout=300s
```

---

## 4) Grant cert-manager CSR status permissions (one-time)
Why: cert-manager must be able to patch `.status.certificate` on CSR objects.

Copy
```bash
cat <<'YAML' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cert-manager-csr-writer
rules:
- apiGroups: ["certificates.k8s.io"]
  resources: ["certificatesigningrequests", "certificatesigningrequests/status"]
  verbs: ["get","list","watch","update","patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cert-manager-csr-writer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cert-manager-csr-writer
subjects:
- kind: ServiceAccount
  name: cert-manager
  namespace: cert-manager
YAML
```

Sanity check:

Copy
```bash
kubectl auth can-i patch certificatesigningrequests/status --as system:serviceaccount:cert-manager:cert-manager
```

---

## 5) Generate user key + CSR (john)
Why: create a private key and certificate request for user john.

Copy (Git Bash)
```bash
MSYS_NO_PATHCONV=1 openssl req -new -newkey rsa:2048 -nodes \
  -keyout ./demo/john.key -out ./demo/john.csr -subj "/CN=john"
```

Copy (PowerShell)
```powershell
& 'C:\\Program Files\\Git\\usr\\bin\\openssl.exe' req -new -newkey rsa:2048 -nodes -keyout .\\demo\\john.key -out .\\demo\\john.csr -subj "/CN=john"
```

---

## 6) Create, approve, and fetch the Kubernetes CSR
Why: submit CSR to Kubernetes, approve it, and fetch the signed certificate when ready.

Copy (Git Bash)
```bash
REQ=$(base64 -w0 ./demo/john.csr 2>/dev/null || base64 ./demo/john.csr | tr -d '\n')
cat > ./demo/csr.yaml <<YAML
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: john-2
spec:
  request: $REQ
  signerName: clusterissuers.cert-manager.io/user-ca-cluster
  usages:
    - digital signature
    - key encipherment
    - client auth
  expirationSeconds: 31536000
YAML
kubectl apply -f ./demo/csr.yaml
kubectl certificate approve john-2
for i in {1..60}; do c=$(kubectl get csr john-2 -o jsonpath='{.status.certificate}'); if [ -n "$c" ]; then echo "$c" | base64 --decode > ./demo/john.crt; echo "Issued"; break; fi; echo "waiting for certificate..."; sleep 5; done
openssl x509 -in ./demo/john.crt -noout -subject -issuer
```

Copy (PowerShell)
```powershell
$REQ = [Convert]::ToBase64String([IO.File]::ReadAllBytes('demo\\john.csr'))
@"
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: john-2
spec:
  request: $REQ
  signerName: clusterissuers.cert-manager.io/user-ca-cluster
  usages:
    - digital signature
    - key encipherment
    - client auth
  expirationSeconds: 31536000
"@ | Set-Content -Path 'demo\\csr.yaml' -NoNewline
kubectl apply -f .\\demo\\csr.yaml
kubectl certificate approve john-2
$b64 = ""; for ($i=0; $i -lt 60 -and [string]::IsNullOrEmpty($b64); $i++) { $b64 = kubectl get csr john-2 -o jsonpath='{.status.certificate}'; if (-not $b64) { Start-Sleep -Seconds 5 } }
if (-not $b64) { throw "certificate not issued" }
[IO.File]::WriteAllBytes('demo\\john.crt',[Convert]::FromBase64String($b64))
& 'C:\\Program Files\\Git\\usr\\bin\\openssl.exe' x509 -in .\\demo\\john.crt -noout -subject -issuer
```

---

## 7) Quick checks
Why: validate the components and the signed certificate.

Copy
```bash
kubectl get crd issuers.cert-manager.io clusterissuers.cert-manager.io
kubectl wait --for=condition=Ready clusterissuer/user-ca-cluster --timeout=300s
```

Copy
```bash
kubectl get csr john-2 -o jsonpath='{.spec.signerName}{"\n"}{.spec.usages}{"\n"}{.status.conditions[*].type}{"\n"}'
```

Copy
```bash
kubectl get certificaterequests.cert-manager.io -A | grep -i john-2 || echo "no CR found"
```

---

## Troubleshooting scenarios
- CSR stays Pending; no certificate populated
  - Feature gate missing on cert-manager; enable `ExperimentalCertificateSigningRequestControllers`
  - Wrong signer; use `clusterissuers.cert-manager.io/user-ca-cluster`
  - RBAC missing; ensure controller can patch CSR status (command above should say `yes`)
  - No CertificateRequest; check logs: `kubectl -n cert-manager logs deploy/cert-manager --tail=200 | grep -Ei 'csr|certificaterequest|john'`
- “resource name may not be empty” when applying YAML
  - Your heredoc or indentation is wrong. Ensure 2 spaces under `metadata:`, `spec:`, `ca:` and that `YAML` ends the heredoc on its own line.
- Git Bash rewrites `-subj` into a path
  - Use `MSYS_NO_PATHCONV=1` before `openssl ... -subj "/CN=..."`
- Fast fallback (keep class moving)
  - Sign directly with the lab CA:

Copy
```bash
openssl x509 -req -in ./demo/john.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out ./demo/john.crt -days 365 -sha256
openssl x509 -in ./demo/john.crt -noout -subject -issuer
```

---

## Files produced
- `demo/john.key` — private key (keep secret)
- `demo/john.csr` — certificate signing request
- `demo/csr.yaml` — Kubernetes CSR manifest
- `demo/john.crt` — issued certificate (after approval + signing)

---

DevOps Academy — Learn by doing. Labs, guides, and real‑world patterns for Kubernetes, Cloud, and Automation.
