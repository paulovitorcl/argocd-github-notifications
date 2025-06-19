# 🔁 Argo CD + GitHub Webhook Sync Flow (Local Dev on macOS)

This guide shows how to run **Argo CD locally** on a Mac using Kubernetes (`kind`), and integrate Argo CD with **GitHub Actions** via dynamic webhooks that respond to application sync status.

---

## 🛠️ Prerequisites

* macOS with [Homebrew](https://brew.sh/)
* GitHub repo with a workflow listening to `repository_dispatch`
* GitHub Personal Access Token (PAT) with `repo` scope

---

## 📆 Setup Steps

### ✅ Step 1: Install Required Tools

```bash
brew install kubectl kind jq argocd
```

---

### 🚀 Step 2: Create Kubernetes Cluster with Kind

```bash
kind create cluster --name argocd-local
```

---

### 🛥️ Step 3: Install Argo CD in the Cluster

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

---

### 🌐 Step 4: Access Argo CD UI

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Then open: [https://localhost:8080](https://localhost:8080)

---

### 🔐 Step 5: Get Admin Password and Login

```bash
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

Use this to login via browser or CLI:

```bash
argocd login localhost:8080 --username admin --password <paste-password> --insecure
```

---

### 📆 Step 6: Create Sample Argo CD App

Replace `your-org/your-repo` with your GitHub path:

```bash
kubectl apply -n argocd -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sample-app
  annotations:
    notifications.argoproj.io/github-repo: "your-org/your-repo"
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/your-repo
    targetRevision: HEAD
    path: .
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated: {}
EOF
```

---

### 🔒 Step 7: Store GitHub Token as Secret

```bash
kubectl -n argocd create secret generic argocd-notifications-secret \
  --from-literal=github-token=<your_github_pat_token>
```

---

### 💡 Step 8: Configure Notifications with Dynamic Webhook

```bash
kubectl -n argocd apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
data:
  template.github-dispatch: |
    url: https://api.github.com/repos/{{.app.metadata.annotations.notifications\.argoproj\.io/github-repo}}/dispatches
    method: POST
    headers:
      Authorization: Bearer $github-token
    body: |
      {
        "event_type": "argocd-sync-complete",
        "client_payload": {
          "app": "{{.app.metadata.name}}",
          "sync_status": "{{.app.status.sync.status}}",
          "health_status": "{{.app.status.health.status}}"
        }
      }

  trigger.on-sync-finished: |
    - when: app.status.operationState.phase in ['Succeeded', 'Failed']
      send: [github-dispatch]
EOF
```

---

### 🛠️ Step 9: Deploy Argo CD Notifications Controller

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/notifications/install.yaml
```

---

### 🔄 Step 10: GitHub Workflow Example

Add to `.github/workflows/argocd-response.yml` in your repo:

```yaml
name: Handle ArgoCD Notification

on:
  repository_dispatch:
    types: [argocd-sync-complete]

jobs:
  check-sync-result:
    runs-on: ubuntu-latest
    steps:
      - name: Print Result
        run: |
          echo "App: ${{ github.event.client_payload.app }}"
          echo "Sync: ${{ github.event.client_payload.sync_status }}"
          echo "Health: ${{ github.event.client_payload.health_status }}"
```

---

### 🔁 Step 11: Trigger Sync

```bash
argocd app sync sample-app
```

Then check GitHub Actions for a new run from `argocd-sync-complete` event.

---

### ❌ Optional Cleanup

```bash
kind delete cluster --name argocd-local
```

---

## 🔐 Notes

* Use annotations per app to dynamically select which GitHub repo gets notified.
* Token used must have access to all target repos.
* GitHub workflow must have `repository_dispatch` trigger configured.

---

## 🔗 References

* [https://argo-cd.readthedocs.io/](https://argo-cd.readthedocs.io/)
* [https://argo-cd.readthedocs.io/en/stable/operator-manual/notifications/](https://argo-cd.readthedocs.io/en/stable/operator-manual/notifications/)
* [https://docs.github.com/en/rest/repos/repos#create-a-repository-dispatch-event](https://docs.github.com/en/rest/repos/repos#create-a-repository-dispatch-event)
