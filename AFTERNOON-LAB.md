# AFTERNOON LAB — Observability + Canary

Based on slide "W9 — Deliver Smartly (Kiến thức + Ví dụ)"

**Prereq:** Cluster `w9` running, ArgoCD installed, root Application watching `argocd/apps/`, repo at `https://github.com/zacknguyn/GitOps-Lab-Simple.git`.

---

## Lab 1: Install Prometheus + Argo Rollouts (via GitOps)

Create 2 files in `argocd/apps/`, push → root auto-deploys.

### File 1: `argocd/apps/kube-prometheus-stack.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kube-prometheus-stack
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://prometheus-community.github.io/helm-charts
    chart: kube-prometheus-stack
    targetRevision: 65.1.1
    helm:
      values: |
        prometheus:
          prometheusSpec:
            serviceMonitorSelectorNilUsesHelmValues: false
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

### File 2: `argocd/apps/argo-rollouts.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argo-rollouts
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://argoproj.github.io/argo-helm
    chart: argo-rollouts
    targetRevision: 2.37.7
    helm:
      values: |
        controller:
          metrics:
            enabled: true
        dashboard:
          enabled: true
          service:
            type: NodePort
  destination:
    server: https://kubernetes.default.svc
    namespace: argo-rollouts
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Commit & push

```bash
cd /home/quang/Documents/GitHub/GitOps-lab-morning
git add argocd/apps/
git commit -m "obs+rollouts"
git push
```

Wait for ArgoCD auto-sync (or force refresh root):

```bash
kubectl annotate app -n argocd root argocd.argoproj.io/refresh=true --overwrite
sleep 20
kubectl -n argocd get applications
# Should see: kube-prometheus-stack, argo-rollouts
```

### Verify

```bash
kubectl -n monitoring get pods -w   # wait for Running
kubectl -n argo-rollouts get pods   # Running
```

If CRDs fail to install, run manually:

```bash
# Download chart and extract CRDs
curl -sL "https://github.com/prometheus-community/helm-charts/releases/download/kube-prometheus-stack-65.1.1/kube-prometheus-stack-65.1.1.tgz" -o /tmp/kps.tgz
tar -xzf /tmp/kps.tgz -C /tmp/
kubectl apply --server-side -f /tmp/kube-prometheus-stack/charts/crds/crds/crd-prometheuses.yaml
kubectl apply --server-side -f /tmp/kube-prometheus-stack/charts/crds/crds/crd-alertmanagers.yaml
# Then delete and let root recreate the app fresh
kubectl delete app -n argocd kube-prometheus-stack
kubectl annotate app -n argocd root argocd.argoproj.io/refresh=true --overwrite
```

**Checkpoint:** 2 apps Synced; pods in `monitoring` + `argo-rollouts` Running.

---

## Lab 2: Write Flask API + Build Image

Create `app/app.py`:

```python
import os, random
from flask import Flask, jsonify
from prometheus_flask_exporter import PrometheusMetrics

app = Flask(__name__)
PrometheusMetrics(app)

ERR = float(os.getenv("ERROR_RATE", "0"))
VER = os.getenv("VERSION", "v1")

@app.get("/")
def index():
    if random.random() < ERR:
        return jsonify(error="injected", version=VER), 500
    return jsonify(ok=True, version=VER)

@app.get("/healthz")
def healthz():
    return "ok", 200
```

Create `app/Dockerfile`:

```dockerfile
FROM python:3.12-slim
RUN pip install flask prometheus-flask-exporter
COPY app.py /app/app.py
WORKDIR /app
ENV FLASK_APP=app.py
EXPOSE 8080
CMD ["flask", "run", "--host=0.0.0.0", "--port=8080"]
```

### Build & load

```bash
docker build -t w9-api:1 app/
minikube image load w9-api:1 -p w9
minikube image ls -p w9 | grep w9-api
# Should show: docker.io/library/w9-api:1
```

**Checkpoint:** `w9-api:1` image available in minikube.

---

## Lab 3: Write k8s manifests + Application → Prometheus scrapes

### File 1: `k8s-api/api.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: api
  namespace: demo
  labels:
    app: api
spec:
  replicas: 4
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: w9-api:1
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8080
          env:
            - name: ERROR_RATE
              value: "0"
            - name: VERSION
              value: "v1"
          readinessProbe:
            httpGet:
              path: /healthz
              port: 8080
  strategy:
    canary:
      steps:
        - setWeight: 25
        - pause: {}
        - setWeight: 50
        - pause:
            duration: 30s
        - setWeight: 100
---
apiVersion: v1
kind: Service
metadata:
  name: api
  namespace: demo
  labels:
    app: api
spec:
  selector:
    app: api
  ports:
    - name: http
      port: 8080
      targetPort: 8080
  type: NodePort
```

### File 2: `k8s-api/servicemonitor.yaml`

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: api
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: api
  namespaceSelector:
    any: true
  endpoints:
    - port: http
      path: /metrics
      interval: 15s
```

### File 3: `argocd/apps/api.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: api
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/zacknguyn/GitOps-Lab-Simple.git
    targetRevision: HEAD
    path: k8s-api/
  destination:
    server: https://kubernetes.default.svc
    namespace: demo
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### Commit & push

```bash
git add k8s-api/ argocd/apps/api.yaml
git commit -m "api"
git push
kubectl annotate app -n argocd root argocd.argoproj.io/refresh=true --overwrite
```

### Verify metric scraping

Wait for sync, then generate traffic and check:

```bash
# Generate load
kubectl -n demo run load --image=busybox --restart=Never -- sh -c "while true; do wget -qO- api:8080/; done"

# Check Prometheus targets
minikube service -n monitoring kube-prometheus-stack-prometheus -p w9 --url
# Open the first URL:9090 → Targets → find "api" → UP

# Or via curl:
curl -s http://$(minikube service -n monitoring kube-prometheus-stack-prometheus -p w9 --url | head -1)/api/v1/targets | python3 -c "import json,sys; data=json.load(sys.stdin); [print(t['labels']['job'], '→', t['health']) for t in data['data']['activeTargets'] if 'api' in t['labels'].get('job','')]"
```

**Checkpoint:** Prometheus Targets shows `api` UP; query `flask_http_request_total` shows data.

---

## Lab 4: Canary — manual promote/abort

### Step 1: Install argo-rollouts kubectl plugin

```bash
curl -sSL --connect-timeout 30 --max-time 120 \
  https://github.com/argoproj/argo-rollouts/releases/download/v1.7.2/kubectl-argo-rollouts-linux-amd64 \
  -o /tmp/kubectl-argo-rollouts
chmod +x /tmp/kubectl-argo-rollouts
mv /tmp/kubectl-argo-rollouts ~/.local/bin/
kubectl argo rollouts version
```

### Step 2: Trigger a canary

Edit `k8s-api/api.yaml`: change `VERSION: "v1"` → `VERSION: "v2"` (and optionally build a new image):

```bash
git commit -am "api v2"
git push
kubectl annotate app -n argocd root argocd.argoproj.io/refresh=true --overwrite
```

Watch the rollout:

```bash
kubectl argo rollouts get rollout api -n demo --watch
```

The Rollout pauses at 25% (because `pause: {}` is infinite).

### Step 3: Promote or abort

If it looks good — promote:

```bash
kubectl argo rollouts promote api -n demo
```

If something is wrong — abort:

```bash
kubectl argo rollouts abort api -n demo
```

**Checkpoint:** Canary promotes to 100% or aborts back to old version on demand.

---

## Challenge: Auto-abort + SLO alert

See slide "★ Challenge: Ship Smartly". Requirements:
1. All changes through Git, `git revert` rollback < 5 min
2. 1 SLO + 1 alert firing to email when error injected
3. Canary auto-aborts via AnalysisTemplate (bad version → auto rollback)

### AnalysisTemplate

Create `k8s-api/analysis-template.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: error-rate
  namespace: demo
spec:
  metrics:
    - name: success-rate
      interval: 30s
      successCondition: result >= 0.95
      failureLimit: 3
      provider:
        prometheus:
          address: http://kube-prometheus-stack-prometheus.monitoring:9090
          query: |
            1 - (sum(rate(flask_http_request_total{status=~"5..",app="api"}[2m]))
                 / sum(rate(flask_http_request_total{app="api"}[2m])))
```

Add `analysis` to the Rollout's canary strategy in `k8s-api/api.yaml`:

```yaml
  strategy:
    canary:
      steps:
        - setWeight: 25
        - pause: { duration: 2m }
        - setWeight: 50
        - pause: { duration: 2m }
        - setWeight: 100
      analysis:
        templates:
          - templateName: error-rate
        startingStep: 0
```

Then test by deploying a version with `ERROR_RATE=0.1` and watch it auto-abort.
