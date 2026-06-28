# Kubernetes Learning - Day 2 🚀

---

## What We Covered Today 🎯

```
✅ Namespaces - dev + prod isolation
✅ ConfigMaps - non-sensitive config
✅ Secrets - passwords and keys
✅ Deployment using ConfigMap + Secret
✅ Verified env variables inside pod
✅ Dev vs Prod different configs
```

---

## Namespaces 📁

Virtual partition inside cluster. Organizes and isolates resources.

```
CLUSTER
├── namespace: dev    ← isolated
├── namespace: prod   ← isolated
└── namespace: monitoring ← isolated
```

### namespace.yaml

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
---
apiVersion: v1
kind: Namespace
metadata:
  name: prod
```

```bash
# Apply
kubectl apply -f namespace.yaml

# Verify
kubectl get namespaces
```

Output:
```
NAME      STATUS
default   Active
dev       Active  ✅
prod      Active  ✅
```

### Namespace Isolation 🔒

```
dev/my-app  ❌ cannot reach  prod/my-app
prod/my-app ❌ cannot reach  dev/my-app

dev/frontend ✅ can reach dev/backend (same namespace)
```

### Set Default Namespace

```bash
# Stop typing -n dev every time
kubectl config set-context --current --namespace=dev

# Switch to prod
kubectl config set-context --current --namespace=prod
```

---

## ConfigMap 📋

ConfigMap = .env file for non-sensitive config.

```
DB_HOST=dev-database
DB_PORT=5432
LOG_LEVEL=debug
```

### configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: dev
data:
  APP_NAME: "my-app"
  APP_PORT: "8080"
  DB_HOST: "dev-database"
  DB_PORT: "5432"
  LOG_LEVEL: "debug"
```

```bash
# Apply
kubectl apply -f configmap.yaml

# Verify
kubectl get configmap -n dev
kubectl describe configmap app-config -n dev

# Live edit
kubectl edit configmap app-config -n dev

# Delete
kubectl delete configmap app-config -n dev
```

---

## Secret 🔒

Secret = .env file for passwords (base64 encoded).

### Encode Values First

```bash
echo -n "yourpasswordhere" | basse64

echo -n "yourkeyhere" | base64
```

### secret.yaml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: dev
type: Opaque
data:
  DB_PASSWORD: here
  API_KEY: here
```

```bash
# Apply
kubectl apply -f secret.yaml

# Verify
kubectl get secrets -n dev
kubectl describe secret app-secret -n dev

# See encoded values
kubectl get secret app-secret -n dev -o yaml

# Decode value
kubectl get secret app-secret -n dev \
  -o jsonpath='{.data.DB_PASSWORD}' | base64 --decode
# Output: yourpassword ✅
```

---

## ConfigMap vs Secret 🤔

| | ConfigMap | Secret |
|---|---|---|
| Use for | Non-sensitive config | Passwords, API keys |
| Examples | DB host, port, app name | DB password, tokens |
| Stored as | Plain text | Base64 encoded |
| Think of it as | .env file | .env file but locked 🔒 |

---

## Deployment Using ConfigMap + Secret 🚀

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: nginx:latest
          ports:
            - containerPort: 80
          envFrom:
            - configMapRef:
                name: app-config    # all configmap keys as env vars
            - secretRef:
                name: app-secret    # all secret keys as env vars
```

```bash
kubectl apply -f deployment.yaml
kubectl get pods -n dev
```

---

## Verify Inside Pod 🔍

```bash
# Get pod name
kubectl get pods -n dev

# SSH into pod
kubectl exec -it <pod-name> -n dev -- /bin/bash

# Check env variables
echo $APP_NAME       # my-app ✅
echo $DB_HOST        # dev-database ✅
echo $DB_PASSWORD    # yourpassword (decoded!) ✅
echo $API_KEY        # yourapikey ✅
echo $LOG_LEVEL      # debug ✅

# Exit
exit
```

K8s automatically decodes base64 when injecting into pod! 🎯

---

## Dev vs Prod Config 🌍

```yaml
# configmap-prod.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: prod
data:
  APP_NAME: "my-app"
  APP_PORT: "8080"
  DB_HOST: "prod-database"   # different from dev!
  DB_PORT: "5432"
  LOG_LEVEL: "error"         # less logging in prod
```

```bash
kubectl apply -f configmap-prod.yaml

# Compare dev vs prod
kubectl describe configmap app-config -n dev
kubectl describe configmap app-config -n prod
```

Output:
```
dev  -> DB_HOST: dev-database  ✅
prod -> DB_HOST: prod-database ✅
```

Same app name - completely different config per namespace! 🎯

---

## All Commands Quick Reference ⚡

```bash
# Namespace commands
kubectl get namespaces
kubectl create namespace dev
kubectl delete namespace dev
kubectl config set-context --current --namespace=dev

# ConfigMap commands
kubectl get configmap -n dev
kubectl describe configmap app-config -n dev
kubectl edit configmap app-config -n dev
kubectl delete configmap app-config -n dev

# Secret commands
kubectl get secrets -n dev
kubectl describe secret app-secret -n dev
kubectl get secret app-secret -n dev -o yaml
kubectl get secret app-secret -n dev \
  -o jsonpath='{.data.DB_PASSWORD}' | base64 --decode

# Deployment commands
kubectl get pods -n dev
kubectl get pods -A
kubectl exec -it <pod-name> -n dev -- /bin/bash
```

---

## Folder Structure 📂

```
k8s-learning/
└── Day-9/
      ├── namespace.yaml        ✅
      ├── configmap.yaml        ✅
      ├── configmap-prod.yaml   ✅
      ├── secret.yaml           ✅
      └── deployment.yaml       ✅
```

---

## Push to GitHub 🐙

```bash
cd ~/k8s-learning
git add .
git commit -m "day2: configmaps, secrets, namespaces hands-on"
git push
```

---

## Key Concepts Summary 🧠

```
Namespace  = folder inside cluster (isolates dev/prod)
ConfigMap  = .env file for normal values
Secret     = .env file for passwords (base64)
envFrom    = inject ALL keys as env variables at once
```

### Golden Rules 🏆

```
✅ Never hardcode config in container image
✅ Never hardcode passwords anywhere
✅ Use ConfigMap for non-sensitive config
✅ Use Secret for passwords and API keys
✅ Different ConfigMap per namespace for dev/prod
✅ Never push Secrets to GitHub!
```

---

## Day 2 Summary ✅

```
✅ dev + prod namespaces created and verified
✅ ConfigMap created with app config
✅ Secret created with base64 encoded password
✅ Deployment uses both ConfigMap + Secret
✅ Verified env variables inside running pod
✅ Dev vs Prod different configs confirmed
✅ Pushed to GitHub
```

---

*K8s Day 2 complete - Namespaces, ConfigMaps, Secrets, Deployments* 🚀
