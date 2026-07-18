# NetworkPolicy — Apply & Verify

## Prerequisite
Your CNI must enforce NetworkPolicy (Calico, Cilium, WeaveNet, etc.).
Plain flannel does **not** enforce these — `kubectl apply` will succeed
silently but nothing will actually be blocked. Check with:
```bash
kubectl get pods -n kube-system | grep -Ei 'calico|cilium|weave'
```
For local dev clusters (kind/minikube), you may need to explicitly enable
a NetworkPolicy-capable CNI at cluster creation.

## Apply, in order (order doesn't strictly matter, but this reads clearly)
```bash
kubectl apply -f 00-default-deny-all.yaml
kubectl apply -f 01-allow-dns.yaml
kubectl apply -f 02-allow-frontend-to-backend.yaml
kubectl apply -f 03-allow-backend-to-postgres.yaml
```

## Required labels
These policies assume your Deployments/Pods carry:
- `app: frontend`
- `app: backend`
- `app: postgres`
Adjust `matchLabels` in the YAML if your actual labels differ.

## Verify the DB is now unreachable from the frontend (this is the test that matters)
```bash
# Should FAIL / time out:
kubectl exec -n asl-dev deploy/frontend -- \
  sh -c 'nc -zv -w3 postgres.asl-dev.svc.cluster.local 5432 || echo "BLOCKED as expected"'
```

## Verify backend -> postgres still works
```bash
# Should SUCCEED:
kubectl exec -n asl-dev deploy/backend -- \
  sh -c 'nc -zv -w3 postgres.asl-dev.svc.cluster.local 5432'
```

## Verify frontend -> backend still works
```bash
kubectl exec -n asl-dev deploy/frontend -- \
  sh -c 'nc -zv -w3 backend.asl-dev.svc.cluster.local 8080'
```

## If frontend needs external ingress (e.g. from an Ingress controller)
Add a dedicated policy allowing ingress to `app: frontend` from the
ingress-nginx namespace/pods — don't loosen the default-deny baseline.
Example:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-to-frontend
  namespace: asl-dev
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - protocol: TCP
          port: 80
```
