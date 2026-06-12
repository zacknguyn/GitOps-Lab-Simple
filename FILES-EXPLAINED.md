# GitOps Files Explained

## 1. argocd/root.yaml — The App-of-Apps Entry Point

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/zacknguyn/GitOps-Lab-Simple.git
    targetRevision: HEAD
    path: argocd/apps
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**What it does:** This is a "meta-application" — an ArgoCD Application that manages other Applications. It watches the `argocd/apps/` folder in the Git repo. Any YAML file you add to that folder becomes a new Application automatically.

**Line-by-line:**
- `apiVersion: argoproj.io/v1alpha1` — The CRD version. `argoproj.io` is the API group registered by ArgoCD. `v1alpha1` means this is still evolving.
- `kind: Application` — The custom resource. Tells ArgoCD "sync a source to a destination."
- `spec.project: default` — ArgoCD groups Applications into Projects for RBAC. `default` exists out-of-the-box.
- `spec.source.repoURL` — Which Git repo to clone.
- `spec.source.targetRevision: HEAD` — Git ref. `HEAD` follows the latest commit. Prod would pin to a tag like `v1.2.3`.
- `spec.source.path: argocd/apps` — Subdirectory. ArgoCD scans this folder and treats every valid K8s YAML as a child Application.
- `spec.destination.server: https://kubernetes.default.svc` — The cluster API. `kubernetes.default.svc` is the in-cluster address. This means "deploy to the same cluster ArgoCD runs on." For multi-cluster, you'd put a different cluster URL here.
- `spec.destination.namespace: argocd` — Where to create child Applications. They're ArgoCD CRDs, so they go in `argocd`.
- `syncPolicy.automated.prune: true` — Delete resources from cluster if their files are deleted from Git. Without this, ArgoCD only creates/updates, never deletes.
- `syncPolicy.automated.selfHeal: true` — If someone edits or deletes a resource in the cluster, ArgoCD reverts it within 3 minutes. Makes the cluster a "read-only" reflection of Git.

**Theoretical concept — App-of-Apps:**
ArgoCD Applications are themselves K8s custom resources. So an Application can create other Applications. The root Application scans a Git folder and creates child Applications for each YAML file it finds. Each child Application then syncs its own source (Helm chart or another Git folder). This lets you manage 100s of Applications with one root entry point.

---

## 2. argocd/apps/kube-prometheus-stack.yaml — Observability Stack

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
          service:
            type: NodePort
          prometheusSpec:
            serviceMonitorSelectorNilUsesHelmValues: false
        alertmanager:
          service:
            type: NodePort
          config:
            global:
              resolve_timeout: 5m
              smtp_smarthost: smtp.gmail.com:587
              smtp_from: quangphongnguyen147@gmail.com
              smtp_auth_username: quangphongnguyen147@gmail.com
              smtp_auth_password: egkvbxkfztxtwfit
            route:
              group_by: ["namespace"]
              group_wait: 30s
              group_interval: 5m
              receiver: "null"
              repeat_interval: 12h
              routes:
                - matchers: ["alertname = Watchdog"]
                  receiver: "null"
                - matchers: ["alertname = APISuccessRateLow"]
                  receiver: email-alerts
            receivers:
              - name: "null"
              - name: email-alerts
                email_configs:
                  - to: quangphongnguyen147@gmail.com
                    send_resolved: true
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

**What it installs:** Prometheus (metrics DB), AlertManager (alert routing), Grafana (dashboards), Prometheus Operator (manages Prometheus/AlertManager CRDs), kube-state-metrics (cluster object metrics), node-exporter (node metrics).

**Why Helm instead of plain YAML?** The kube-prometheus-stack generates ~50+ resources. Writing them all individually would be impractical. Helm is the right tool for packaging complex software with default values and upgrade logic.

**Key fields explained:**
- `spec.source.chart: kube-prometheus-stack` — Instead of a Git path, this installs from a Helm chart repository. ArgoCD downloads the chart, merges our values with defaults, and applies the rendered templates.
- `targetRevision: 65.1.1` — Chart version, not app version. v65.1.1 maps to specific internal versions of Prometheus/Grafana/AlertManager.
- `prometheusSpec.serviceMonitorSelectorNilUsesHelmValues: false` — **This was the critical fix.** By default, Prometheus only discovers ServiceMonitors with label `release: kube-prometheus-stack`. Setting this to `false` makes it discover ALL ServiceMonitors (including ours for the Flask API).
- `alertmanager.config.global.smtp_smarthost: smtp.gmail.com:587` — Gmail's SMTP server on port 587 (STARTTLS encryption). AlertManager connects here to send emails.
- `smtp_auth_password` — A Google App Password (16 characters). Generated at myaccount.google.com/apppasswords. Requires 2FA on the Google account.
- `route.receiver: "null"` — Default receiver is the built-in null/blackhole. Only specific routes (like APISuccessRateLow) get forwarded.
- `syncOptions: [ServerSideApply: true]` — Uses kubectl's server-side apply instead of client-side. Necessary for CRDs because they can be large and server-side apply handles field ownership better.

---

## 3. argocd/apps/argo-rollouts.yaml — Canary Controller

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
    targetRevision: 2.41.0
    helm:
      values: |
        controller:
          metrics:
            enabled: true
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

**What it does:** Installs the Argo Rollouts controller, which watches for `Rollout` resources (Kubernete's built-in `Deployment` replacement). It handles canary deployments: traffic splitting, analysis runs, automatic rollbacks.

**Why not use Deployments?** K8s Deployments only support rolling updates (gradually replace pods). They can't pause at a specific traffic percentage, run measurements to validate the new version, or auto-abort based on metrics. Argo Rollouts adds these capabilities by introducing the `Rollout` CRD.

**The chart version issue:** v2.37.7 crashes on K8s 1.35.1 with "unmarshal object into string." This is a Go JSON serialization bug — the chart's templates produced YAML that the K8s 1.35 API server couldn't parse. v2.41.0 fixed this.

---

## 4. argocd/apps/api.yaml — Flask API Application

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
    syncOptions:
      - CreateNamespace=true
    automated:
      prune: true
      selfHeal: true
```

**What happens when you push a change:**
1. ArgoCD clones the repo
2. Looks at `k8s-api/` directory
3. Finds all YAML files: `api.yaml`, `servicemonitor.yaml`, `analysis-template.yaml`, `slo-alert.yaml`
4. Applies them to the cluster in `demo` namespace
5. If a file is deleted from Git → ArgoCD deletes it from cluster (prune)

**Note on namespaces:** The Rollout has `namespace: demo` in its metadata. The Application also has `destination.namespace: demo`. If these conflict, ArgoCD's destination wins. For resources without a namespace in their YAML, ArgoCD auto-injects the destination namespace.

---

## 5. argocd/apps/expense-tracker.yaml — 2-Repo Pattern

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: expense-tracker
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/zacknguyn/Expense-Tracker.git
    targetRevision: HEAD
    path: k8s/
  destination:
    server: https://kubernetes.default.svc
    namespace: expense-tracker
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

**The 2-repo pattern demonstrated:** This Application points to a **different Git repo** than root. root watches `GitOps-Lab-Simple/argocd/apps/`. One of the files there is `expense-tracker.yaml`, which creates an Application that points to `Expense-Tracker.git/k8s/`.

**Why 2 repos?**
- App developers work in Expense-Tracker repo, change k8s manifests independently
- GitOps repo stays lean — only manages Application YAMLs and tooling config
- Different access controls: app team commits to Expense-Tracker, platform team reviews the Application creation

---

## 6. k8s-api/api.yaml — Rollout + Service (The Core)

### Rollout section

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
              value: "8"
          readinessProbe:
            httpGet:
              path: /healthz
              port: 8080
  strategy:
    canary:
      steps:
        - setWeight: 25
        - pause:
            duration: 2m
        - setWeight: 50
        - pause:
            duration: 2m
        - setWeight: 100
      analysis:
        templates:
          - templateName: error-rate
        startingStep: 0
```

**Rollout vs Deployment:** The pod template section (replicas, selector, containers, probes) is identical to a Deployment. The difference is in `spec.strategy`. Deployment uses `strategy.type: RollingUpdate`. Rollout uses `strategy.canary` with fine-grained steps and analysis.

**Container config:**
- `image: w9-api:1` — Locally built image. In production this would be `registry.example.com/api:v1.2.3` with `imagePullPolicy: Always`.
- `imagePullPolicy: IfNotPresent` — Only pull if not cached. Since we loaded the image into minikube, it exists locally. `Always` would try Docker Hub (where it doesn't exist) and fail.
- `ERROR_RATE: "0"` — Controls the Flask app's random error injection. 0 = no errors. 0.3 = 30% of requests return 500.
- `VERSION: "8"` — Just a label in the JSON response. Changing it triggers a new Rollout revision.
- `readinessProbe` — K8s checks this every ~5s. Pod isn't Ready until it returns 200. The Service won't send traffic to non-Ready pods. The Rollout also waits for Ready before proceeding.

**Canary steps:**
```
Step 0: setWeight 25   → Scale canary to 1, stable to 3. ~25% traffic.
Step 1: pause 2m       → Wait here. AnalysisRun takes measurements.
Step 2: setWeight 50   → Scale canary to 2, stable to 2. ~50% traffic.
Step 3: pause 2m       → Wait. AnalysisRun continues measuring.
Step 4: setWeight 100  → Scale canary to 4, stable to 0. 100% traffic. Canary becomes new stable.
```

**Analysis integration:** `startingStep: 0` means the AnalysisRun starts immediately when the canary begins (at step 0). If the analysis fails at any point, the rollout aborts.

### How traffic splitting actually works

Argo Rollouts doesn't modify the Service to do weighted routing. Instead, it scales ReplicaSets:

```
Desired: 4 replicas

Step 0 (25%):
  Stable RS: 3 pods  (3/4 = 75%)
  Canary RS: 1 pod   (1/4 = 25%)

Step 2 (50%):
  Stable RS: 2 pods  (2/4 = 50%)
  Canary RS: 2 pods  (2/4 = 50%)

Step 4 (100%):
  Stable RS: 0 pods  (promoted)
  Canary RS: 4 pods  (now the new stable)
```

The Service selector `app: api` matches both stable and canary pods (they all have that label). K8s distributes requests randomly across all matching pods. With 3 stable + 1 canary, each request has a 1/4 = 25% chance of hitting canary.

### Service section

```yaml
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

**Service concepts:**
- `selector: { app: api }` — Match pods with label `app: api`. Both stable and canary pods have this label, so the Service distributes traffic to all of them.
- `port: 8080` — The port other services/consumers use. `api:8080` in DNS resolves to this port.
- `targetPort: 8080` — The port on the pods. Must match `containerPort`.
- `type: NodePort` — Opens a high port on every node. `nodeIP:nodePort` → forwarded to Service → load-balanced to pod.

**Kubernetes service discovery flow:**
```
Pod A wants to reach "api:8080":
  1. Resolves "api" via CoreDNS → gets ClusterIP 10.105.30.122
  2. Sends request to 10.105.30.122:8080
  3. iptables DNAT → one of the pod IPs: 10.244.0.X:8080
  4. Request arrives at the container
```

---

## 7. k8s-api/servicemonitor.yaml — Prometheus Discovers Metrics

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

**What it does:** This CRD tells Prometheus Operator: "find Services with label `app: api`, then scrape their `/metrics` endpoint every 15 seconds."

**How Prometheus discovers targets:**
1. Prometheus Operator watches for ServiceMonitor CRs
2. For each ServiceMonitor, finds matching Services across all namespaces
3. For each matching Service, resolves pod IPs and ports
4. Configures Prometheus (updates Prometheus config file) to scrape those endpoints

**The critical config:** `prometheusSpec.serviceMonitorSelectorNilUsesHelmValues: false` in the Helm values makes Prometheus discover ALL ServiceMonitors. Without it, only ServiceMonitors with label `release: kube-prometheus-stack` are discovered.

**Where `/metrics` comes from:** The `prometheus_flask_exporter` library in `app.py` automatically exposes Flask metrics at `/metrics`. It doesn't need explicit route registration.

---

## 8. k8s-api/analysis-template.yaml — The Auto-Abort Brain

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: error-rate
  namespace: demo
spec:
  metrics:
    - name: success-rate
      initialDelay: 120s
      interval: 30s
      successCondition: "len(result) > 0 ? result[0] >= 0.95 : false"
      failureLimit: 3
      provider:
        prometheus:
          address: http://kube-prometheus-stack-prometheus.monitoring:9090
          query: |
            1 - (sum(rate(flask_http_request_total{status=~"5..",namespace="demo"}[2m]))
                 / sum(rate(flask_http_request_total{namespace="demo"}[2m])))
```

**What it does:** Defines what metric to measure and what counts as success. Argo Rollouts controller reads this and runs the measurement periodically during a canary.

**The PromQL explained:**
```
1 - (5xx_rate / total_rate)
```
- `rate(METRIC[2m])` = how fast the counter is increasing per second, averaged over 2 minutes
- `status=~"5.."` = regex match for 500-599
- `sum()` = aggregate across all pods (since each pod has its own counter)
- Result: success rate between 0 and 1 (e.g., 0.97 = 97%)

**Why `result[0]` and the guard are necessary:**
```
Prometheus HTTP API always returns data as an array:
    result = [{"metric": {...}, "value": [timestamp, "0.97"]}]

result >= 0.95 compares: []float64 >= float64 → CRASH
result[0] >= 0.95 gets: "0.97" >= 0.95 → but "0.97" is a STRING, not a number

Wait — how does result[0] work then?
The Argo Rollouts Prometheus provider parses the API response and stores
the numeric value. So result[0] is actually a float64. The provider handles
the conversion internally.

Empty result (no data in time window):
    result = []
    result[0] → index out of range → ERROR
    len(result) > 0 ? result[0] >= 0.95 : false → false → not an error
```

**initialDelay: 120s:** The `[2m]` lookback in the query needs 2 minutes of data. Without the delay, the first measurement happens immediately, the query returns nothing, and every measurement errors. After 5 errors (default consecutiveErrorLimit), the analysis aborts the rollout — even for a good version.

**failureLimit: 3:** After 3 measurements fail (result < 0.95), the analysis fails and the rollout aborts. With `interval: 30s` and a 2-minute pause step, there are 4 measurements per step. 3 failures = 75% bad measurements = auto-abort.

---

## 9. k8s-api/slo-alert.yaml — Alerting Rule

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: api-slo
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  groups:
    - name: api-slo
      rules:
        - record: slo:success_rate:5m
          expr: |
            1 - (sum(rate(flask_http_request_total{status=~"5..",namespace="demo"}[5m]))
                 / sum(rate(flask_http_request_total{namespace="demo"}[5m])))
        - alert: APISuccessRateLow
          expr: slo:success_rate:5m < 0.95
          for: 2m
          labels:
            severity: critical
          annotations:
            summary: API success rate is below 95%
            description: |
              Success rate for the demo API is {{ $value | humanizePercentage }}.
              Expected at least 95%.
```

**Recording rule vs alerting rule:**
- `record: slo:success_rate:5m` — Prometheus evaluates this query periodically and stores the result as a new time series named `slo:success_rate:5m`. This is like a materialized view in SQL: pre-compute once, reference many times.
- `alert: APISuccessRateLow` — Evaluates the expression. If true for `for: 2m`, fires the alert.

**Why the recording rule:** The full query involves two `rate()` calls with `[5m]` windows, each scanning all `flask_http_request_total` data points. Running this every alert evaluation would be expensive. The recording rule runs once per evaluation interval (~30s) and stores the result. The alert just checks `slo:success_rate:5m < 0.95` — instant.

**`for: 2m`:** The condition must hold for 2 minutes before the alert fires. This prevents flapping from momentary error spikes.

**`release: kube-prometheus-stack` label:** This is critical. Prometheus has `ruleSelector.matchLabels: { release: kube-prometheus-stack }`. Without this label, Prometheus ignores this rule entirely and the alert never fires.

**Alert flow:**
```
Prometheus evaluates slo:success_rate:5m < 0.95
  → True for 2 minutes
  → Alert state: PENDING → FIRING
  → Sends Alert to AlertManager
  → AlertManager matches route "alertname = APISuccessRateLow"
  → Forwards to email-alerts receiver
  → Sends email via Gmail SMTP
  → Subject: [FIRING:1] APISuccessRateLow
  → Body: "Success rate for the demo API is 85%. Expected at least 95%."
```

---

## 10. Network Routing in Detail

### External user → Expense-Tracker frontend

```
Browser: http://192.168.49.2:31279/
           │
           │ 192.168.49.2 = minikube host IP
           │ 31279 = NodePort assigned to expense-tracker-frontend Service
           ▼
minikube host (192.168.49.2)
           │
           │ iptables rule: port 31279 → ClusterIP 10.104.3.200:80
           │ This is managed by kube-proxy
           ▼
kube-proxy DNAT (Destination NAT)
           │
           │ Randomly picks a pod matching the Service selector:
           │ DNAT to: 10.244.0.5:80 (frontend pod 1)
           │       or: 10.244.0.6:80 (frontend pod 2)
           ▼
Nginx container in frontend pod
           │
           │ ┌─ path starts with /api/?
           │ │    proxy_pass http://expense-tracker-backend:8081
           │ │    DNS resolves → ClusterIP 10.97.10.226
           │ │    kube-proxy DNAT → backend pod (10.244.0.7:8081)
           │ │    Go API processes → returns JSON
           │ │    Response flows back the same path
           │ │
           │ └─ otherwise?
           │      Serve static file from /usr/share/nginx/html
           │      (React app: index.html, .js, .css)
           │
           ▼
Response → iptables reverse NAT → Browser
```

### Canary AnalysisRun → Prometheus

```
Argo Rollouts controller (argo-rollouts namespace)
           │
           │ HTTP GET http://kube-prometheus-stack-prometheus.monitoring:9090/api/v1/query
           │                                   │           │
           │                              service name    namespace
           ▼
CoreDNS resolves: kube-prometheus-stack-prometheus.monitoring
           │
           │ Full DNS: kube-prometheus-stack-prometheus.monitoring.svc.cluster.local
           │ Returns ClusterIP: 10.108.170.204
           ▼
kube-proxy DNAT → prometheus-0 pod (10.244.0.8:9090)
           │
           │ Prometheus processes the PromQL query
           │ Looks up stored time series data
           │ Computes the result
           ▼
Response: {"status":"success","data":{"resultType":"vector","result":[{"metric":{},"value":[timestamp,"0.97"]}]}}
```

### Flask API → Prometheus scraping

```
Every 15 seconds:
Prometheus pod → HTTP GET http://10.244.0.X:8080/metrics
                                          │
                                    api pod IP (resolved via ServiceMonitor)
                                          │
                                          ▼
Flask app's prometheus_flask_exporter handler
  │
  │ Reads prometheus client counters:
  │   flask_http_request_total{status="200", ...} = 42
  │   flask_http_request_total{status="500", ...} = 3
  │
  ▼
Response body:
  # HELP flask_http_request_total Total number of requests
  # TYPE flask_http_request_total counter
  flask_http_request_total{method="GET",status="200",...} 42
  flask_http_request_total{method="GET",status="500",...} 3
```

### Kubernetes DNS resolution

Each Service gets a DNS record: `<name>.<namespace>.svc.cluster.local`

| From | To | DNS query |
|------|----|-----------|
| argo-rollouts controller | Prometheus | `kube-prometheus-stack-prometheus.monitoring` |
| frontend nginx | backend | `expense-tracker-backend` (same namespace → just service name) |
| analysis-run | Prometheus | `kube-prometheus-stack-prometheus.monitoring:9090` |

CoreDNS runs as a pod in `kube-system`. Every pod's `/etc/resolv.conf` points to CoreDNS. DNS queries like `servicename.namespace` are resolved by appending `svc.cluster.local`.

---

## Key Concepts Summary

**ArgoCD Application** — A CRD that tells ArgoCD "sync source X to destination Y." Source can be a Git path or a Helm chart. Destination is typically the same cluster but can be remote.

**Rollout** — ArgoCD's replacement for Kubernetes Deployment. Same pod template, but with canary/bluegreen strategy instead of rolling update.

**AnalysisTemplate** — Defines measurements that run during a canary. If measurements fail, the rollout aborts. This is the "decision engine" for auto-rollbacks.

**ServiceMonitor** — Tells Prometheus Operator which Services to scrape. Creates a bridge between app deployments and Prometheus config.

**PrometheusRule** — Defines recording rules (pre-computed metrics) and alerting rules (conditions that fire alerts). Evaluated by Prometheus periodically.

**Service (ClusterIP/NodePort)** — Stable network endpoint backed by pods. ClusterIP is internal-only. NodePort adds external access via a high port on every node.
