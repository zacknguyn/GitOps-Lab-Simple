# How Everything Flows

## GitOps Flow (app-of-apps)

```
You push to GitHub
       │
       ▼
ArgoCD detects change (polls every 3min or manual refresh)
       │
       ▼
root Application sees new/updated files in argocd/apps/
       │
       ▼
Creates/updates child Applications (kube-prometheus-stack, argo-rollouts, api, expense-tracker)
       │
       ▼
Each child Application syncs its source to the cluster
       │
       ├── Helm charts (kube-prometheus-stack, argo-rollouts) → Helm release in cluster
       │
       └── Plain YAML (api, expense-tracker) → kubectl apply to cluster
```

The root Application watches only the `argocd/apps/` folder. Everything in that folder becomes an ArgoCD Application. Each Application points to either a Helm chart or a YAML directory in a Git repo.

---

## How a Canary Deploys

```
You change VERSION in k8s-api/api.yaml → git commit → git push
       │
       ▼
ArgoCD syncs → updates the Rollout resource in the cluster
       │
       ▼
Argo Rollouts controller sees pod template changed
       │
       ▼
Creates new ReplicaSet (canary) with 1 pod
       
       ▼
Updates Service to send 25% traffic to canary, 75% to stable
       │
       ▼
AnalysisRun starts (after 120s initialDelay)
       │
       ▼
Every 30s: PromQL query → Prometheus computes success rate
            if result[0] >= 0.95 → measurement passes
            if result[0] < 0.95  → measurement fails
       │
       ├── 3 failures → AnalysisRun fails → Rollout auto-aborts
       │                     └── Canary pods deleted, stable stays
       │
       └── All pass → Rollout proceeds to next step
                            │
                    25% → wait 2m → 50% → wait 2m → 100%
                            │
                     Rollout Healthy, canary becomes new stable
```

### Why the AnalysisTemplate uses result[0]
```
Prometheus API always returns data as an array (called a "vector"):
    [{"metric": {...}, "value": [timestamp, "0.97"]}]

If you write result >= 0.95, you're comparing:
    [{metric:..., value:...}] >= 0.95
Which is: []float64 >= float64 → CRASH

So you write result[0] >= 0.95:
    [{metric:..., value:...}] [0] >= 0.95
Which is: 0.97 >= 0.95 → TRUE

Plus the len() guard for when there's no data yet:
    len(result) > 0 ? result[0] >= 0.95 : false
```

---

## Prometheus → AlertManager → Email

```
Flask API exposes /metrics
       │
       ▼
ServiceMonitor tells Prometheus: "scrape api:8080/metrics every 15s"
       │
       ▼
Prometheus stores flask_http_request_total counter
       │
       ▼
PrometheusRule defines:
  - Recording rule: slo:success_rate:5m = 1 - (5xx rate / total rate)
  - Alert rule:     if slo < 0.95 for 2 minutes → fire APISuccessRateLow
       │
       ▼
Prometheus evaluates rules every ~30s
       │
       ├── if condition met for 2 minutes → sends alert to AlertManager
       │                                     │
       │                                     ▼
       │                              AlertManager matches route
       │                              → sends to email-alerts receiver
       │                                     │
       │                                     ▼
       │                              Sends email via Gmail SMTP
       │
       └── if condition not met → alert stays inactive
```

---

## Load Generator Flow (why we need it)

```
Browser / User
       │
       ├── /healthz → Flask → always returns 200 (no error injection)
       │
       ├── /metrics → Flask → always returns 200 (no error injection)
       │
       └── /         → Flask → has ERROR_RATE logic
                            │
                    random.random() < ERROR_RATE ?
                       │                │
                     yes               no
                       │                │
                       ▼                ▼
                  500 error          200 OK

Without load generator: only health checks + Prometheus hit the API
→ All requests go to /healthz and /metrics → always 200 → success rate 100%
→ "Broken" version passes analysis and auto-promotes

With load generator: hits / endpoint directly
→ With ERROR_RATE=0.3: 30% of total requests return 500
→ Success rate ≈ 70% → below 95% → auto-abort works
```

---

## Expense-Tracker 2-Repo Pattern

### Why 2 repos?
```
Expense-Tracker repo (app code)          GitOps-Lab-Simple repo (config)
─────────────────────────────            ──────────────────────────────
backend/main.go                          argocd/apps/expense-tracker.yaml
backend/store.go                                   │
backend/Dockerfile                                  │ references Expense-Tracker repo
frontend/src/*                                     │ path: k8s/
frontend/Dockerfile                                │
frontend/nginx.conf                                ▼
k8s/backend.yaml                     ArgoCD sees the Application
k8s/frontend.yaml                              │
                                     syncs k8s/ folder from
                                     Expense-Tracker repo
                                              │
                                              ▼
                                     Creates resources in cluster
```

The separation means:
- App devs work in the Expense-Tracker repo, never touch GitOps
- Platform team approves the ArgoCD Application once, it auto-follows the k8s/ folder
- No need to copy k8s manifests between repos

### How frontend talks to backend
```
Browser loads expense-tracker-frontend:80
       │
       ▼
React app makes fetch('/api/transactions')  ← relative URL, not http://localhost:8081
       │
       ▼
Nginx receives request at /api/transactions
       │
       ▼
Nginx proxy_pass → http://expense-tracker-backend:8081/api/transactions
       │
       ▼
Go API processes request, returns JSON
       │
       ▼
Nginx returns response to browser
```

The key change: we updated `const API = '/api'` instead of `http://localhost:8081/api`. This lets nginx handle the routing — the browser talks to nginx (same origin), nginx proxies to the backend. No CORS issues.

---

## What's In-Memory vs Persistent

| Component | Storage | Data survives pod restart? |
|-----------|---------|---------------------------|
| Flask API (w9-api) | In-memory counter | No (but it's just metrics) |
| Expense-Tracker backend | In-memory Store struct | No |
| Prometheus | PersistentVolume (in minikube) | Yes (within cluster lifetime) |
| AlertManager | PersistentVolume | Yes |
| Grafana | PersistentVolume + sqlite | Yes |

The Expense-Tracker could use a database, but for a lab it's fine. Just know that if the backend pod restarts, you start fresh.

---

## Namespaces Overview

```
argocd           → ArgoCD itself (server, repo-server, controller, redis, dex)
argo-rollouts    → Argo Rollouts controller (manages Rollout resources)
monitoring       → Prometheus stack (prometheus, alertmanager, grafana, operator)
demo             → Flask API (Rollout + Service + pods)
expense-tracker  → Expense-Tracker (backend + frontend pods)
```
