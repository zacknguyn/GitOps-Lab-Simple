# PRATICE.md — GitOps Lab (theo slide)

Theo slide buổi sáng, Lab 0 → 7.

Cần sẵn: Docker, kubectl, minikube, git, 1 repo GitHub trống.

---

## Lab 0: Dựng cụm + app đầu tiên

### Tạo cụm minikube

```bash
minikube start -p w9 --driver=docker
kubectl config use-context w9
kubectl get nodes  # STATUS Ready
```

### Tạo repo + app

```bash
cd /tmp
mkdir gitops && cd gitops
mkdir k8s argocd/apps .github/workflows -p
```

### Tạo `k8s/web.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: nginx:1.27
```

### Push lên GitHub

```bash
# Tạo repo trống trên GitHub trước (vd: gitops-lab)
git init
git add .
git commit -m "init"
git branch -M main
git remote add origin https://github.com/<USER>/gitops-lab.git
git push -u origin main
```

**Checkpoint:** cụm w9 chạy + repo có `k8s/web.yaml`. CHƯA apply vào cụm.

---

## Lab 1: Cài ArgoCD

```bash
kubectl create ns argocd
kubectl apply --server-side -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd rollout status deploy/argocd-server
kubectl -n argocd get pods  # argocd-* Running
```

### Mở UI (tùy chọn)

```bash
kubectl -n argocd port-forward svc/argocd-server 8080:443 &
# Lấy password:
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo
# https://localhost:8080 — user: admin
```

**Checkpoint:** argocd-* pods Running.

---

## Lab 2: Tạo Application → ArgoCD tự sync

### Tạo `argocd/apps/web.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/<USER>/gitops-lab.git
    targetRevision: HEAD
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: demo
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Sửa `<USER>` thành username GitHub.

### Apply Application (bằng TAY)

```bash
# Push argocd/apps/web.yaml lên Git
git add argocd/
git commit -m "add Application"
git push

# Apply Application bằng tay (chưa có root)
kubectl apply -f argocd/apps/web.yaml
```

### Kiểm tra

```bash
kubectl -n argocd get app web       # Synced/Healthy
kubectl -n demo get deploy,pod      # 2 pod web
```

**Checkpoint:** web = Synced/Healthy. ArgoCD tự tạo Deployment trong cụm.

---

## Lab 3: Sync & Self-heal

### Đổi qua Git

```bash
# Sửa k8s/web.yaml: replicas 2 → 4
git commit -am "scale 2->4"
git push
# ArgoCD tự kéo → 4 pod (~3 phút)
kubectl -n demo get pods
```

### Self-heal

```bash
# Scale tay lên 9
kubectl -n demo scale deploy/web --replicas=9
# Quan sát: tự kéo về 4
kubectl -n demo get deploy web -w
```

**Hiểu:** Cụm luôn = Git. Sửa tay không sống sót.

---

## Lab 4: Rollback

```bash
git revert HEAD --no-edit
git push
# ArgoCD sync cụm về trạng thái cũ
kubectl -n demo get pods  # 2 pods (trở lại)
```

**Rule:** `git revert` (có audit trail). Không dùng `kubectl rollout undo` (self-heal ghi đè).

---

## Lab 5: App-of-apps

### Tạo `argocd/root.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/<USER>/gitops-lab.git
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

Sửa `<USER>` thành username GitHub.

### Apply root

```bash
git add argocd/root.yaml
git commit -m "app-of-apps"
git push
kubectl apply -f argocd/root.yaml
```

### Kiểm tra

```bash
kubectl -n argocd get applications
# root + web — root quản web (đã ở argocd/apps/)
```

**Chốt:** Từ giờ thêm app mới = thả file vào `argocd/apps/` + `git push` → root tự đẻ. KHÔNG cần `kubectl apply` nữa.

---

## Lab 6: Sync Waves

Thêm Namespace (chạy đầu), ConfigMap (trước), Service (sau), gắn sync-wave để ép thứ tự.

### Tạo `k8s/namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
```

### Sửa `k8s/web.yaml` thành 3 resource (ConfigMap + Deployment + Service)

```yaml
# --- ConfigMap (wave 0) ---
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-config
  namespace: demo
  annotations:
    argocd.argoproj.io/sync-wave: "0"
data:
  MESSAGE: "hello from gitops"

---
# --- Deployment (wave 1) ---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: demo
  annotations:
    argocd.argoproj.io/sync-wave: "1"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: nginx:1.27
          envFrom:
            - configMapRef:
                name: web-config

---
# --- Service (wave 2) ---
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: demo
  annotations:
    argocd.argoproj.io/sync-wave: "2"
spec:
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
```

### Push và quan sát

```bash
git add .
git commit -m "sync waves: ns(-1) cm(0) deploy(1) svc(2)"
git push
```

Vào UI ArgoCD → tab Sync → thấy thứ tự:
```
Namespace -1 → ConfigMap 0 → Deployment 1 → Service 2
```

**Checkpoint:** wave apply đúng thứ tự.

---

## Lab 7: CI (plan-on-PR)

### Tạo `.github/workflows/validate.yml`

```yaml
name: validate

on:
  pull_request:
    paths:
      - "k8s/**"

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install kubeconform
        run: |
          curl -sSLo kc.tgz https://github.com/yannh/kubeconform/releases/download/v0.6.7/kubeconform-linux-amd64.tar.gz
          tar -xzf kc.tgz && sudo mv kubeconform /usr/local/bin/
      - name: Validate YAML
        run: kubeconform -strict -summary k8s/
```

### Push

```bash
git add .github/
git commit -m "CI validate"
git push
```

### Branch protection (GitHub UI)

1. GitHub → Repo → Settings → Branches → Add rule
2. Branch name pattern: `main`
3. Bật:
   - [x] Require a pull request before merging
   - [x] Require status checks to pass → chọn `validate`
4. Save

### Kiểm tra

Mở PR sửa `k8s/web.yaml` (vd: sai schema `replicas: abc`) → workflow chạy → ❌ đỏ → nút Merge bị khóa.

---

## Tổng kết

```
Lab 0: Cụm + repo + app
Lab 1: Cài ArgoCD
Lab 2: Application → auto sync
Lab 3: Self-heal (sửa tay bị ghi đè)
Lab 4: Rollback (git revert)
Lab 5: App-of-apps (root quản nhiều con)
Lab 6: Sync waves (ép thứ tự apply)
Lab 7: CI validate (chặn YAML lỗi)
```

Cụm đã GitOps-managed. Git là nguồn sự thật. ArgoCD tự sync. Rollback bằng `git revert`.
