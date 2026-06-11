# Commands — Observability + Canary

Run in order. Copy-paste each block.

---

## Prereq: cluster + ArgoCD

```bash
# Already done if you're reading this:
# minikube start -p w9 --cpus=4 --memory=6g --driver=docker
# kubectl create ns argocd
# kubectl apply --server-side -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
# kubectl -n argocd wait --for=condition=ready pods --all --timeout=180s
# kubectl apply -f argocd/root.yaml
```

---

## Lab 1 — Prometheus + Argo Rollouts

```bash
# 1a. Create kube-prometheus-stack Application
cat <<'EOF' > argocd/apps/kube-prometheus-stack.yaml
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
EOF

# 1b. Create argo-rollouts Application
cat <<'EOF' > argocd/apps/argo-rollouts.yaml
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
EOF

# 1c. Commit & push
git add argocd/apps/
git commit -m "lab1: prometheus + argo-rollouts"
git push
kubectl annotate app -n argocd root argocd.argoproj.io/refresh=true --overwrite

# 1d. Wait for pods
echo "Waiting for monitoring pods..."
kubectl -n monitoring wait --for=condition=ready pod -l app.kubernetes.io/instance=kube-prometheus-stack --timeout=180s 2>/dev/null
echo "Waiting for argo-rollouts pods..."
kubectl -n argo-rollouts wait --for=condition=ready pod --all --timeout=120s 2>/dev/null

# 1e. If CRDs fail, install manually:
kubectl apply --server-side -f /tmp/kube-prometheus-stack/charts/crds/crds/crd-prometheuses.yaml 2>/dev/null
kubectl apply --server-side -f /tmp/kube-prometheus-stack/charts/crds/crds/crd-alertmanagers.yaml 2>/dev/null
```

---

## Lab 2 — Flask API

```bash
# 2a. Create app files
mkdir -p app

cat <<'EOF' > app/app.py
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
EOF

cat <<'EOF' > app/Dockerfile
FROM python:3.12-slim
RUN pip install flask prometheus-flask-exporter
COPY app.py /app/app.py
WORKDIR /app
ENV FLASK_APP=app.py
EXPOSE 8080
CMD ["flask", "run", "--host=0.0.0.0", "--port=8080"]
EOF

# 2b. Build & load image
docker build -t w9-api:1 app/
minikube image load w9-api:1 -p w9
minikube image ls -p w9 | grep w9-api
```

---

## Lab 3 — k8s manifests + API Application

```bash
# 3a. Create k8s-api directory
mkdir -p k8s-api

cat <<'EOF' > k8s-api/api.yaml
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
EOF

cat <<'EOF' > k8s-api/servicemonitor.yaml
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
EOF

cat <<'EOF' > argocd/apps/api.yaml
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
EOF

# 3b. Commit & push
git add k8s-api/ argocd/apps/api.yaml
git commit -m "lab3: w9-api rollout + servicemonitor"
git push
kubectl annotate app -n argocd root argocd.argoproj.io/refresh=true --overwrite

# 3c. Wait for sync
kubectl -n argocd wait app/api --for=jsonpath='{.status.health.status}'=Healthy --timeout=60s 2>/dev/null

# 3d. Generate load & verify metrics
kubectl -n demo run load --image=busybox --restart=Never -- sh -c "while true; do wget -qO- api:8080/; done" 2>/dev/null
echo "Checking Prometheus targets..."
curl -s http://$(minikube service -n monitoring kube-prometheus-stack-prometheus -p w9 --url 2>/dev/null | head -1)/api/v1/targets 2>/dev/null | python3 -c "import json,sys; data=json.load(sys.stdin); [print(t['labels']['job'], '→', t['health']) for t in data['data']['activeTargets'] if 'api' in t['labels'].get('job','')]" 2>/dev/null || echo "Prometheus target check — run manually after sync"
```

---

## Lab 4 — Canary (promote/abort)

```bash
# 4a. Install argo rollouts plugin
curl -sSL --connect-timeout 30 --max-time 120 \
  https://github.com/argoproj/argo-rollouts/releases/download/v1.7.2/kubectl-argo-rollouts-linux-amd64 \
  -o /tmp/kubectl-argo-rollouts
chmod +x /tmp/kubectl-argo-rollouts
mv /tmp/kubectl-argo-rollouts ~/.local/bin/
kubectl argo rollouts version

# 4b. Trigger canary (edit VERSION v1 -> v2)
sed -i 's/value: "v1"/value: "v2"/' k8s-api/api.yaml
git commit -am "lab4: bump api to v2"
git push
kubectl annotate app -n argocd root argocd.argoproj.io/refresh=true --overwrite

# 4c. Watch rollout
kubectl argo rollouts get rollout api -n demo --watch

# 4d. Promote (skip current pause)
kubectl argo rollouts promote api -n demo

# 4e. Abort (if needed)
kubectl argo rollouts abort api -n demo

# 4f. Revert in git
git revert HEAD --no-edit
git push
```

---

## Lab 5 — Expense-Tracker (2-repo GitOps)

Run these in the Expense-Tracker repo:

```bash
cd /home/quang/Documents/GitHub/Expense-Tracker

# 5a. Backend Dockerfile
cat <<'EOF' > backend/Dockerfile
FROM golang:1.26 AS build
WORKDIR /app
COPY go.mod ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /app/server .
FROM alpine:3.21
COPY --from=build /app/server /server
EXPOSE 8081
CMD ["/server"]
EOF

# 5b. Frontend Dockerfile
cat <<'EOF' > frontend/Dockerfile
FROM node:22 AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build
FROM nginx:1.27-alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
EOF

# 5c. Frontend nginx config
cat <<'EOF' > frontend/nginx.conf
server {
    listen 80;
    location / {
        root /usr/share/nginx/html;
        try_files $uri $uri/ /index.html;
    }
    location /api/ {
        proxy_pass http://expense-backend:8081;
    }
}
EOF

# 5d. k8s manifests
mkdir -p k8s/backend k8s/frontend

cat <<'EOF' > k8s/backend.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: expense-backend
  namespace: demo
  labels:
    app: expense-backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: expense-backend
  template:
    metadata:
      labels:
        app: expense-backend
    spec:
      containers:
        - name: backend
          image: expense-backend:1
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8081
---
apiVersion: v1
kind: Service
metadata:
  name: expense-backend
  namespace: demo
  labels:
    app: expense-backend
spec:
  selector:
    app: expense-backend
  ports:
    - port: 8081
      targetPort: 8081
EOF

cat <<'EOF' > k8s/frontend.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: expense-frontend
  namespace: demo
  labels:
    app: expense-frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: expense-frontend
  template:
    metadata:
      labels:
        app: expense-frontend
    spec:
      containers:
        - name: frontend
          image: expense-frontend:1
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: expense-frontend
  namespace: demo
  labels:
    app: expense-frontend
spec:
  selector:
    app: expense-frontend
  ports:
    - port: 80
      targetPort: 80
  type: NodePort
EOF

# 5e. Commit & push to Expense-Tracker
git add backend/Dockerfile frontend/Dockerfile frontend/nginx.conf k8s/
git commit -m "add Dockerfiles + k8s manifests"
git push

# 5f. Build images & load into minikube
docker build -t expense-backend:1 backend/
docker build -t expense-frontend:1 frontend/
minikube image load expense-backend:1 -p w9
minikube image load expense-frontend:1 -p w9
```

Now switch back to gitops-lab-morning:

```bash
cd /home/quang/Documents/GitHub/GitOps-lab-morning

# 5g. ArgoCD Applications
cat <<'EOF' > argocd/apps/expense-backend.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: expense-backend
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/zacknguyn/Expense-Tracker.git
    targetRevision: HEAD
    path: k8s/backend
  destination:
    server: https://kubernetes.default.svc
    namespace: demo
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF

cat <<'EOF' > argocd/apps/expense-frontend.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: expense-frontend
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/zacknguyn/Expense-Tracker.git
    targetRevision: HEAD
    path: k8s/frontend
  destination:
    server: https://kubernetes.default.svc
    namespace: demo
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF

# 5h. Commit & push
git add argocd/apps/expense-backend.yaml argocd/apps/expense-frontend.yaml
git commit -m "lab5: expense-tracker apps"
git push
kubectl annotate app -n argocd root argocd.argoproj.io/refresh=true --overwrite

# 5i. Verify
kubectl -n argocd get apps -w
```

---

## Challenge — AnalysisTemplate + SLO alert

```bash
# Write AnalysisTemplate
cat <<'EOF' > k8s-api/analysis-template.yaml
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
EOF

# Edit api.yaml: replace pause: {} with pause: {duration: 2m} + add analysis block
# (Open the file in your editor and make the change)

# Deploy broken version with ERROR_RATE=0.1
# Edit k8s-api/api.yaml: change ERROR_RATE "0" to "0.1"
# Build v2 image, commit, push, watch auto-abort
```
