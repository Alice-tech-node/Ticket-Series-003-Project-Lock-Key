# Sealed Secrets — Operator Workflow

## 1. Deploy the controller
```bash
kubectl apply -f ../argocd/sealed-secrets-app.yaml
kubectl -n kube-system rollout status deployment/sealed-secrets-controller
```

## 2. Install the `kubeseal` CLI (matches the controller version)
```bash
# macOS
brew install kubeseal

# Linux
KUBESEAL_VERSION='0.26.3'
curl -OL "https://github.com/bitnami-labs/sealed-secrets/releases/download/v${KUBESEAL_VERSION}/kubeseal-${KUBESEAL_VERSION}-linux-amd64.tar.gz"
tar -xvzf kubeseal-${KUBESEAL_VERSION}-linux-amd64.tar.gz kubeseal
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
```

## 3. Create the plaintext Secret LOCALLY — never apply it, never commit it
```bash
kubectl create secret generic db-credentials \
  --namespace asl-dev \
  --from-literal=POSTGRES_USER=asl_app \
  --from-literal=POSTGRES_PASSWORD='use-a-real-strong-password-here' \
  --dry-run=client -o yaml > /tmp/db-credentials-plaintext.yaml
```

## 4. Seal it into ciphertext that's safe for Git
```bash
kubeseal --format yaml \
  --controller-name sealed-secrets-controller \
  --controller-namespace kube-system \
  < /tmp/db-credentials-plaintext.yaml \
  > db-credentials-sealedsecret.yaml

# Immediately delete the plaintext — it must never touch disk in Git history
shred -u /tmp/db-credentials-plaintext.yaml   # or `rm -P` on macOS
```

## 5. Commit only the sealed file
```bash
git add sealed-secrets/db-credentials-sealedsecret.yaml
git commit -m "Add sealed db-credentials for asl-dev"
git push
```

Argo CD (or `kubectl apply -f`) applies the `SealedSecret`. The controller in
`kube-system` decrypts it in-cluster and materializes a normal `Secret`
named `db-credentials` in `asl-dev`, which your backend Deployment consumes
via `envFrom.secretRef` or `valueFrom.secretKeyRef` exactly as before.

## Key facts for the CTO write-up
- **Public repo safe**: encrypted values are asymmetrically encrypted against
  the controller's public key. Only the private key in-cluster can decrypt.
- **No manual rotation ceremony**: rotating the password just means re-running
  steps 3-5 and re-applying; Git history keeps old ciphertext, which is
  harmless since it's undecryptable outside the cluster.
- **Backup the controller's private key** (`kubectl get secret -n kube-system
  -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml`) somewhere secure —
  losing it means losing the ability to decrypt existing SealedSecrets.
- **Scope**: by default a SealedSecret is bound to its target namespace+name,
  so a sealed file can't be copy-pasted into another namespace and decrypted
  there. Good — that's the intended blast-radius limit.
