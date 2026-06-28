# Kubernetes Learning - Day 3 🚀

---

## What We Covered Today 🎯

```
✅ Ingress - smart traffic routing
✅ Nginx Ingress Controller
✅ Path based routing
✅ Helm - K8s package manager
✅ Created first Helm chart
✅ Deployed with Helm to dev + prod
```

---

## Part 1 - Ingress 🌐

---

## Why Ingress? 🤔

Without Ingress - NodePort for every app:
```
http://ec2-ip:30080  -> app1
http://ec2-ip:30090  -> app2
http://ec2-ip:30091  -> app3
```

Problems:
- Ugly URLs with ports ❌
- Different port for every app ❌
- No HTTPS ❌
- Hard to manage ❌

With Ingress:
```
http://myapp.com/app1  -> app1 ✅
http://myapp.com/app2  -> app2 ✅
http://myapp.com/api   -> backend ✅
```

Clean URLs. One entry point. 🎯

---

## What is Ingress? 🎯

Ingress = smart receptionist at the front door.

Every request comes to Ingress first.
Ingress reads the URL and routes to the right service.

```
Internet
    |
    v
Ingress Controller  <- one entry point
    |
    ├── /app1  -> app1 service
    ├── /app2  -> app2 service
    └── /api   -> backend service
```

---

## Two Parts of Ingress 🔧

```
Ingress Resource   = the rules (what routes where)
Ingress Controller = the engine (does the routing)
```

Rules without engine = nothing happens.
You need BOTH. ✅

Most popular controller = Nginx Ingress Controller

---

## Ingress vs LoadBalancer ⚡

| | LoadBalancer | Ingress |
|---|---|---|
| One app | Fine | Fine |
| Multiple apps | One LB per app (expensive!) | One Ingress for all |
| URL routing | No | Yes |
| HTTPS/SSL | Manual | Built in |
| Cost on AWS | High (one ELB each) | Low (one ELB total) |

---

## Install Nginx Ingress Controller 🔧

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/cloud/deploy.yaml

# Wait for it to be ready
kubectl get pods -n ingress-nginx -w
```

Wait until:
```
ingress-nginx-controller-xxx   Running ✅
```

---

## Deploy Two Apps 🚀

### app1-deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1
  namespace: dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app1
  template:
    metadata:
      labels:
        app: app1
    spec:
      containers:
        - name: app1
          image: nginx:latest
          ports:
            - containerPort: 80
```

### app1-service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app1-service
  namespace: dev
spec:
  type: ClusterIP       # internal only - Ingress handles external
  selector:
    app: app1
  ports:
    - port: 80
      targetPort: 80
```

### app2-deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app2
  namespace: dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app2
  template:
    metadata:
      labels:
        app: app2
    spec:
      containers:
        - name: app2
          image: httpd:latest    # apache - different from app1
          ports:
            - containerPort: 80
```

### app2-service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app2-service
  namespace: dev
spec:
  type: ClusterIP
  selector:
    app: app2
  ports:
    - port: 80
      targetPort: 80
```

```bash
kubectl apply -f app1-deployment.yaml
kubectl apply -f app1-service.yaml
kubectl apply -f app2-deployment.yaml
kubectl apply -f app2-service.yaml

# Verify
kubectl get pods -n dev
kubectl get services -n dev
```

---

## Ingress Rules - Path Based Routing 🛣️

### ingress.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  namespace: dev
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /app1
            pathType: Prefix
            backend:
              service:
                name: app1-service
                port:
                  number: 80
          - path: /app2
            pathType: Prefix
            backend:
              service:
                name: app2-service
                port:
                  number: 80
```

```bash
kubectl apply -f ingress.yaml

# Check ingress
kubectl get ingress -n dev
kubectl describe ingress my-ingress -n dev
```

---

## Test Ingress 🧪

```bash
# Get ingress IP
kubectl get ingress -n dev

# Test paths
curl http://<ingress-ip>/app1   # nginx response ✅
curl http://<ingress-ip>/app2   # apache response ✅
```

---

## Ingress Commands ⚡

```bash
kubectl get ingress -n dev
kubectl describe ingress my-ingress -n dev
kubectl delete ingress my-ingress -n dev
```

---

## Part 2 - Helm 📦

---

## Why Helm? 🤔

Without Helm - apply files one by one:
```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f configmap.yaml
kubectl apply -f ingress.yaml
kubectl apply -f hpa.yaml
```

With Helm - one command:
```bash
helm install my-app ./my-chart
```

Same result. One command. ✅

---

## What is Helm? 🎯

Helm = npm for Kubernetes.

```
npm install express   -> installs express
helm install my-app   -> deploys entire app on K8s
```

Two key concepts:
```
Chart  = package (all YAMLs bundled together)
Values = variables (customize the package)
```

---

## Install Helm 🔧

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify
helm version
```

---

## Helm Chart Structure 📂

```
helm-chart/
├── Chart.yaml          <- chart info (name, version)
├── values.yaml         <- default variables
└── templates/          <- YAML templates
      ├── deployment.yaml
      └── service.yaml
```

---

## Chart.yaml

```yaml
apiVersion: v2
name: my-app
description: My first Helm chart
type: application
version: 0.1.0
appVersion: "1.0.0"
```

---

## values.yaml - The Magic File ✨

```yaml
replicaCount: 1

image:
  repository: nginx
  tag: latest

service:
  type: ClusterIP
  port: 80

environment: dev
appName: my-helm-app
```

---

## templates/deployment.yaml - Uses Variables 🔄

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.appName }}           # from values.yaml
  namespace: {{ .Values.environment }}
spec:
  replicas: {{ .Values.replicaCount }}  # from values.yaml
  selector:
    matchLabels:
      app: {{ .Values.appName }}
  template:
    metadata:
      labels:
        app: {{ .Values.appName }}
    spec:
      containers:
        - name: {{ .Values.appName }}
          image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
          ports:
            - containerPort: 80
```

---

## templates/service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Values.appName }}-service
  namespace: {{ .Values.environment }}
spec:
  type: {{ .Values.service.type }}
  selector:
    app: {{ .Values.appName }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: 80
```

---

## Deploy with Helm 🚀

```bash
# Install in dev
helm upgrade --install my-app ./helm-chart \
  --set environment=dev \
  --set replicaCount=1 \
  -n dev

# Install in prod with different values
helm upgrade --install my-app ./helm-chart \
  --set environment=prod \
  --set replicaCount=2 \
  --set image.tag=stable \
  -n prod
```

Same chart. Different values. Different environments! 🎯

---

## Helm Commands Daily Use ⚡

```bash
# Deploy
helm install my-app ./helm-chart -n dev
helm upgrade my-app ./helm-chart -n dev
helm upgrade --install my-app ./helm-chart -n dev  # safest

# Check
helm list -n dev
helm list -n prod
helm history my-app -n dev

# Rollback
helm rollback my-app 1 -n dev

# Preview without deploying
helm upgrade --install my-app ./helm-chart -n dev --dry-run

# Remove
helm uninstall my-app -n dev
```

---

## Helm vs Raw kubectl 🥊

| | Raw kubectl | Helm |
|---|---|---|
| Multiple YAMLs | Apply one by one | One command |
| Dev vs Prod | Duplicate files | One chart, different values |
| Updates | Manual | helm upgrade |
| Rollback | Manual | helm rollback |
| Versioning | No | Yes - automatic |

---

## Folder Structure 📂

```
k8s-learning/
└── Day-3/
      ├── app1-deployment.yaml   ✅
      ├── app1-service.yaml      ✅
      ├── app2-deployment.yaml   ✅
      ├── app2-service.yaml      ✅
      ├── ingress.yaml           ✅
      └── helm-chart/
            ├── Chart.yaml       ✅
            ├── values.yaml      ✅
            └── templates/
                  ├── deployment.yaml  ✅
                  └── service.yaml     ✅
```

---

## Push to GitHub 🐙

```bash
cd ~/k8s-learning
git add .
git commit -m "day3: ingress nginx and helm chart hands-on"
git push
```

---

## Key Concepts Summary 🧠

```
Ingress Controller  = engine that handles routing
Ingress Resource    = rules (which URL goes where)
Path routing        = /app1 -> app1 service
ClusterIP           = internal service (Ingress handles external)

Helm Chart          = package of all K8s YAMLs
values.yaml         = variables for the chart
helm install        = deploy everything in one shot
helm upgrade        = update existing deployment
helm rollback       = go back to previous version
```

---

## Golden Rules 🏆

```
✅ Use Ingress instead of NodePort in production
✅ Use ClusterIP for services behind Ingress
✅ Use Helm for complex multi-file deployments
✅ Use values.yaml for env specific config
✅ Always use --dry-run before applying in prod
✅ Never skip helm history - always know your versions
```

---

## Day 3 Summary ✅

```
✅ Nginx Ingress Controller installed
✅ Two apps deployed (nginx + apache)
✅ Ingress rules created
✅ Path based routing tested (/app1, /app2)
✅ Helm installed
✅ First Helm chart created from scratch
✅ Deployed to dev with Helm
✅ Deployed to prod with different values
✅ Helm commands practiced
✅ Pushed to GitHub
```

---

*K8s Day 3 complete - Ingress, Nginx Controller, Helm Chart* 🚀
