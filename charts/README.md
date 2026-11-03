# Observability Helm umbrella chart (laboratory)

Installs Prometheus, Loki, Grafana and Promtail (log collection) through upstream Helm dependencies.

## Important
- This is a **lab configuration**, not production-ready. No HA, no persistent data, short retention, no external authentication/TLS. Pod restarts may lose metrics/logs.
- **Promtail is legacy/EOL**. For a durable installation migrate log collection to Grafana Alloy. This chart includes Promtail only to keep a simple lab deployment.
- Check chart versions/values compatibility before deploying. Dependency ranges are examples; commit `Chart.lock` and vendored charts if Argo CD cannot reach public Helm repositories.
- `observability` is the Helm release name, which the datasource URLs assume.
- Loki filesystem mode is for one replica only; do not scale without object storage and appropriate architecture.

## Setup on kube01
```bash
kubectl get nodes
kubectl get pods -n argocd
kubectl create namespace monitoring --dry-run=client -o yaml | kubectl apply -f -
# Enter a strong password interactively; do not commit or echo it.
read -rsp 'Grafana admin password: ' GRAFANA_PASSWORD; echo
kubectl -n monitoring create secret generic grafana-admin \
  --from-literal=admin-user=admin \
  --from-literal=admin-password="$GRAFANA_PASSWORD" \
  --dry-run=client -o yaml | kubectl apply -f -
unset GRAFANA_PASSWORD

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
cd helm/observability
helm dependency update
helm lint .
helm template observability . -n monitoring > /tmp/observability-rendered.yaml
helm upgrade --install observability . -n monitoring --create-namespace --wait --timeout 15m
```

## GitOps alternative
Commit `helm/observability` including `Chart.lock` to `https://github.com/souzaal/argocd.git`, and optionally vendor `charts/*.tgz` if dependency fetching is unreliable. Create the Grafana secret separately, then apply `argocd-application.yaml` from the repository root (or adjust its path).
```bash
kubectl apply -f argocd-application.yaml
kubectl -n argocd get applications observability
```
**Choose either initial Helm CLI management or Argo CD management**; do not have both manage the same release/resources simultaneously.

## Access and checks
```bash
kubectl -n monitoring get pods,svc
kubectl -n monitoring port-forward svc/observability-grafana 3000:80
```
Open http://localhost:3000 from the machine where port-forward runs. If you run it on kube01, use an SSH tunnel from your desktop, e.g. `ssh -L 3000:localhost:3000 souza@192.168.0.119` while port-forward is active.

If the pods are Pending, inspect `kubectl -n monitoring describe pod <name>` and node resources. If Loki fails to start, inspect `kubectl -n monitoring logs <loki-pod>`. Check that Grafana's datasource service names match `kubectl -n monitoring get svc`.
