# W9 — Deliver Smartly: Full Guide

## 0. Cluster
```bash
minikube delete -p w9
minikube start -p w9 --cpus=4 --memory=6g --kubernetes-version=v1.35.1
kubectl cluster-info
minikube ip -p w9
```
If `kubectl get nodes` shows `w9` Ready, you're good.

## 0.5 Recovery After Shutdown

If your laptop shuts down without stopping minikuke, the cluster is recoverable:

```bash
# Start the existing profile (reuses the same VM, images preserved)
minikube start -p w9

# Set docker env for image builds
eval $(minikube -p w9 docker-env)
```

Wait ~1 minute, then check pods:

```bash
# Watch until everything stabilizes
kubectl get pods -A
```

**Common issues:**

- **Frontend pods crash** with `host not found in upstream "expense-tracker-backend"` — nginx resolves DNS at startup and fails if CoreDNS isn't ready yet. Delete them to force a restart (DNS will be up by then):
  ```bash
  kubectl delete pods -n expense-tracker -l app=expense-tracker-frontend
  ```
- **ArgoCD, Grafana, or Prometheus pods stuck in Error** — same DNS race. Delete to restart:
  ```bash
  kubectl delete pods -n argocd --all
  kubectl delete pods -n monitoring --all
  ```
- **94% disk warning** — run `docker system prune` if you see "nearly out of disk space" during start.

All pods should settle to Running within 2 minutes.

**Restore port-forwards:**

```bash
# ArgoCD
kubectl port-forward -n argocd svc/argocd-server 8080:443 &
echo "https://localhost:8080  admin / <get password below>"
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo

# Grafana
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3001:80 &
echo "http://localhost:3001  admin / prom-operator"

# AlertManager
kubectl port-forward -n monitoring svc/kube-prometheus-stack-alertmanager 9093:9093 &
echo "http://localhost:9093"
```

**Check images are intact:**

```bash
minikube image ls -p w9 | grep -E "w9-api|expense-tracker"
```
Should list `w9-api:1`, `expense-tracker-backend:1`, `expense-tracker-frontend:1`.

## 1. ArgoCD
```bash
kubectl create ns argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd wait --for=condition=Ready pods --all --timeout=120s
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
kubectl patch svc -n argocd argocd-server -p '{"spec":{"type":"NodePort"}}'
ARGOCD_PORT=$(kubectl get svc -n argocd argocd-server -o jsonpath='{.spec.ports[0].nodePort}')
echo "http://$(minikube ip -p w9):$ARGOCD_PORT  admin / <password>"
```
Open that URL and login with admin + the password. Pro tip: the password can have special chars so paste it carefully (Ctrl+Shift+V in terminal).

## 2. root App-of-Apps
Create `argocd/root.yaml` — it watches `argocd/apps/` and auto-deploys anything you put there.
```bash
kubectl apply -f argocd/root.yaml
git add -A && git commit -m "root" && git push
```
`kubectl get app -n argocd root` should show Synced/Healthy. No child apps yet since `argocd/apps/` is empty.

## 3. Prometheus + Argo Rollouts
Create `argocd/apps/kube-prometheus-stack.yaml` and `argocd/apps/argo-rollouts.yaml`.
```bash
git add -A && git commit -m "lab1" && git push
kubectl annotate app -n argocd root argocd.argoproj.io/refresh=true --overwrite
kubectl -n argocd get applications -w
```
After a minute, both apps should appear as Synced/Healthy. Check pods in `monitoring` and `argo-rollouts` namespaces — everything should be Running.

**Known issue — CRDs:** If you see a sync error about "failed to install CRDs" (annotation exceeds 262KB), run this:
```bash
curl -sL "https://github.com/prometheus-community/helm-charts/releases/download/kube-prometheus-stack-65.1.1/kube-prometheus-stack-65.1.1.tgz" -o /tmp/kps.tgz
tar -xzf /tmp/kps.tgz -C /tmp/
kubectl apply --server-side -f /tmp/kube-prometheus-stack/charts/crds/crds/crd-prometheuses.yaml
kubectl apply --server-side -f /tmp/kube-prometheus-stack/charts/crds/crds/crd-alertmanagers.yaml
kubectl delete app -n argocd kube-prometheus-stack
kubectl annotate app -n argocd root argocd.argoproj.io/refresh=true --overwrite
```
The CRDs get installed outside Helm, then recreate the app — it'll work after that.

```bash
PROM_PORT=$(kubectl get svc -n monitoring kube-prometheus-stack-prometheus -o jsonpath='{.spec.ports[0].nodePort}')
echo "Prometheus: http://$(minikube ip -p w9):$PROM_PORT"
```

Install the argo-rollouts kubectl plugin:
```bash
curl -sSL --connect-timeout 30 --max-time 120 \
  https://github.com/argoproj/argo-rollouts/releases/download/v1.7.2/kubectl-argo-rollouts-linux-amd64 \
  -o /tmp/kubectl-argo-rollouts
chmod +x /tmp/kubectl-argo-rollouts && mv /tmp/kubectl-argo-rollouts ~/.local/bin/
```
Run `kubectl argo rollouts version` to confirm it works. If it says command not found, make sure `~/.local/bin` is in your PATH.

## 4. Flask API
Create `app/app.py` — a Flask app with prometheus_flask_exporter, reads ERROR_RATE and VERSION env vars.
Create `app/Dockerfile` — python:3.12-slim base, installs flask + prometheus_flask_exporter, runs on port 8080.
```bash
docker build -t w9-api:1 app/
minikube image load w9-api:1 -p w9
```
`minikube image ls -p w9 | grep w9-api` should show the image is loaded.

## 5. k8s Manifests + App
Create `k8s-api/api.yaml` (Rollout + Service), `k8s-api/servicemonitor.yaml`, `argocd/apps/api.yaml`.
```bash
git add -A && git commit -m "lab3" && git push
kubectl annotate app -n argocd root argocd.argoproj.io/refresh=true --overwrite
kubectl get pods -n demo -w
```
You should see 4 api pods spin up. Run `kubectl argo rollouts get rollout api -n demo` — it should say Healthy, 4/4 replicas, Step 5/5.

Generate some traffic so Prometheus has data:
```bash
kubectl -n demo run load --image=busybox --restart=Never -- sh -c "while true; do wget -qO- api:8080/; done"
curl -s "http://$(minikube ip -p w9):$PROM_PORT/api/v1/targets" | python3 -m json.tool
```
Look for `api` in the targets list — it should say `up`. Try querying `flask_http_request_total` in the Prometheus UI.

## 6. Canary
Edit `k8s-api/api.yaml` and bump VERSION to something new:
```bash
git commit -am "v2" && git push
kubectl annotate app -n argocd root argocd.argoproj.io/refresh=true --overwrite
kubectl argo rollouts get rollout api -n demo --watch
```
Watch what happens: the rollout creates a new canary pod, sets weight to 25%, then pauses. You'll see Step 1/5, SetWeight=25.

To move it forward:
```bash
kubectl argo rollouts promote api -n demo
```
It'll go 25% → 50% (after 2m) → 100%. Eventually the rollout shows Healthy.

Or to kill the canary:
```bash
kubectl argo rollouts abort api -n demo
```
The canary pods get torn down, stable stays. The rollout shows Degraded with "RolloutAborted" message.

Don't forget to reset VERSION back to normal when you're done testing.

## 7. Auto-Abort
This is where the magic happens. Create `k8s-api/analysis-template.yaml` that defines how to measure success rate.

**The tricky part — why result[0] matters:**
When Prometheus returns a query result, it comes back as an array (a "vector"), not a single number. So writing `result >= 0.95` crashes with "mismatched types []float64 and float64". You need to grab the first element: `result[0]`. And if the query returns nothing (e.g., no data yet), even `result[0]` fails, so guard it: `len(result) > 0 ? result[0] >= 0.95 : false`.

Also add `initialDelay: 120s` — otherwise the analysis runs immediately and there's no 2-minute window of data yet, causing error cascades.

Add the `analysis` block inside the Rollout's canary strategy referencing this template.

Now test it:
```bash
kubectl create deployment loader -n demo --image=busybox -- sh -c "while true; do wget -qO- http://api:8080/; done"
```
This creates traffic hitting the `/` endpoint (not `/healthz`). Without this, there's no errors and the "broken" version auto-promotes.

Set ERROR_RATE=0.3 in api.yaml (30% of responses will be 500) and bump VERSION:
```bash
git commit -am "test auto-abort" && git push
kubectl annotate app -n argocd api argocd.argoproj.io/refresh=true --overwrite
kubectl argo rollouts get rollout api -n demo --watch
```

**What should happen:**
1. Rollout starts canary at 25%
2. AnalysisRun starts after 2 minutes (initialDelay)
3. First measurement runs — PromQL sees ~70% success rate (30% injected errors)
4. Since 70% < 95%, the measurement counts as a failure
5. After 3 failures, the AnalysisRun fails
6. Rollout auto-aborts: "RolloutAborted: Metric success-rate assessed Failed"
7. Canary pods scale down, stable version stays

Clean up:
```bash
kubectl delete deployment -n demo loader
# Revert ERROR_RATE to 0
```

## 8. SLO Alert + Email
Create `k8s-api/slo-alert.yaml` — a PrometheusRule with:
- A recording rule `slo:success_rate:5m` that stores the 5-minute success rate
- An alert `APISuccessRateLow` that fires when that metric drops below 95% for 2 minutes

Add alertmanager config to `argocd/apps/kube-prometheus-stack.yaml`. You'll need:
- SMTP server (e.g., smtp.gmail.com:587)
- Your Gmail address as sender and receiver
- A Google App Password (generate at myaccount.google.com/apppasswords — needs 2FA)

```bash
git add -A && git commit -m "slo alert + email" && git push
kubectl annotate app -n argocd kube-prometheus-stack argocd.argoproj.io/refresh=true --overwrite
```

Check the Prometheus alerts page: `http://$(minikube ip -p w9):$PROM_PORT/alerts` — you should see `APISuccessRateLow` listed as inactive.

For the AlertManager UI:
```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-alertmanager 9093:9093 &
echo "http://localhost:9093"
```

**Testing the alert:**
Do the same thing as the auto-abort test — load generator + ERROR_RATE=0.3. After about 5 minutes:
1. `slo:success_rate:5m` drops below 0.95
2. 2 minutes later, the alert fires
3. Prometheus shows it as "firing" in the Alerts page
4. AlertManager picks it up
5. You get an email — check spam folder if you don't see it

Clean up: delete the loader, revert ERROR_RATE.

## 9. Grafana
```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3001:80 &
echo "http://localhost:3001  admin / prom-operator"
```
Login, then ☰ → Dashboards → import. Or browse the pre-loaded Kubernetes dashboards.

## 10. Expense-Tracker (2-Repo Pattern)
The idea: app code + k8s manifests live in one repo, GitOps config lives in another.

In the Expense-Tracker repo, you need:
- `backend/Dockerfile` — multi-stage Go build
- `frontend/Dockerfile` — node build + nginx serve
- `frontend/nginx.conf` — proxies `/api` to backend
- `k8s/backend.yaml` — Deployment (1 replica) + Service (with sessionAffinity)
- `k8s/frontend.yaml` — Deployment (2 replicas) + Service (NodePort)

Also update:
- `CategoryContext.tsx` and `TransactionContext.tsx`: change `const API = '/api'` (was localhost:8081)
- `main.go`: CORS origin to `"*"` (was localhost:5173)

Build and push:
```bash
docker build -t expense-tracker-backend:1 backend/
docker build -t expense-tracker-frontend:1 frontend/
minikube image load expense-tracker-backend:1 -p w9
minikube image load expense-tracker-frontend:1 -p w9
git add -A && git commit -m "k8s" && git push
```

In the GitOps repo, create `argocd/apps/expense-tracker.yaml` pointing to the Expense-Tracker repo's `k8s/` directory:
```bash
git add -A && git commit -m "expense-tracker" && git push
kubectl annotate app -n argocd root argocd.argoproj.io/refresh=true --overwrite
kubectl get pods -n expense-tracker -w
```
You should see 3 pods (1 backend + 2 frontend) all Running.

```bash
FRONTEND_PORT=$(kubectl get svc -n expense-tracker expense-tracker-frontend -o jsonpath='{.spec.ports[0].nodePort}')
echo "http://$(minikube ip -p w9):$FRONTEND_PORT"
```
Open that URL — you should see the Expense-Tracker UI. You can add transactions and categories.

**Heads up:** The backend uses an in-memory store. If the pod restarts, you lose your data. That's fine for a lab. If you want persistence, you'd add a database.

## URLs
| Service | URL |
|---------|-----|
| ArgoCD | `http://192.168.49.2:$ARGOCD_PORT` |
| Prometheus | `http://192.168.49.2:$PROM_PORT` |
| Grafana | `http://localhost:3001` (admin/prom-operator) |
| AlertManager | `http://localhost:9093` |
| Expense-Tracker | `http://192.168.49.2:<nodeport>` (run `echo $FRONTEND_PORT`) |

## Things That Went Wrong (so you don't hit them)
| Symptom | Why | Fix |
|---------|-----|-----|
| AnalysisRun error "[]float64" | Prom returns arrays, you compared it like a scalar | Use `result[0]` and `len(result) > 0 ? ... : false` |
| "Broken" canary auto-promotes | No real traffic — only health checks hit `/healthz` (always 200) | Run a load generator hitting `/` |
| AnalysisRun errors 5 times then fails | Query ran before 2 minutes of data existed | Add `initialDelay: 120s` |
| API pods crash-loop | Argo Rollouts chart v2.37.7 incompatible with K8s 1.35+ | Use v2.41.0+ |
| CRDs fail to install | Helm chart has CRD yaml >262KB annotation | Install CRDs manually with `--server-side` |
| Frontend can't reach API | CORS blocks the production URL | Set origin to `"*"` |
| Data disappears | 2 backend replicas with separate in-memory stores | Use 1 replica + `sessionAffinity: ClientIP` |

## How It All Fits Together
```
You commit → Git repo → ArgoCD syncs → k8s cluster

Prometheus scrapes api:8080/metrics → stores metrics
Prometheus rules → fires APISuccessRateLow → AlertManager → email
Prometheus query → AnalysisTemplate → auto-aborts bad canaries

Expense-Tracker: Browser → nginx (port 31279) → /api/ → Go backend (port 8081)
```
