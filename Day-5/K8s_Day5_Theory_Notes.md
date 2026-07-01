# Kubernetes Learning - Day 5 🚀

---

## What We Built Today 🎯

Complete production-like K8s setup with all components connected:

```
Namespace + ConfigMap + Secret + PV + PVC +
Deployment + Service + Ingress + HPA
```

---

## Architecture 🏗️

```
Internet
    |
    v
Ingress (nginx)
    |
    v
Service (ClusterIP)
    |
    v
Deployment (2 replicas)
    |
    ├── ConfigMap (DB_HOST, LOG_LEVEL)
    ├── Secret (DB_PASSWORD)
    ├── PVC (persistent storage)
    └── HPA (auto scale 1-5 pods)
```

---

## Folder Structure 📂

```
k8s-learning/
└── Day-5/
      └── full-project/
            ├── namespace.yaml
            ├── configmap.yaml
            ├── secret.yaml
            ├── pv.yaml
            ├── pvc.yaml
            ├── deployment.yaml
            ├── service.yaml
            ├── ingress.yaml
            └── hpa.yaml
```

---

## namespace.yaml 📁

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
kubectl apply -f namespace.yaml
kubectl get namespaces
```

---

## configmap.yaml 📋

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: dev
data:
  APP_NAME: "full-project"
  APP_PORT: "80"
  DB_HOST: "dev-database"
  DB_PORT: "5432"
  LOG_LEVEL: "debug"
```

```bash
kubectl apply -f configmap.yaml
kubectl get configmap -n dev
```

---

## secret.yaml 🔒

```bash
# Encode password first
echo -n "devpassword123" | base64
# ZGV2cGFzc3dvcmQxMjM=
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: dev
type: Opaque
data:
  DB_PASSWORD: ZGV2cGFzc3dvcmQxMjM=
```

```bash
kubectl apply -f secret.yaml
kubectl get secrets -n dev
```

---

## pv.yaml 💽

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: project-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /tmp/full-project-data
```

```bash
kubectl apply -f pv.yaml
kubectl get pv
```

---

## pvc.yaml 📋

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: project-pvc
  namespace: dev
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

```bash
kubectl apply -f pvc.yaml

# Verify bound to PV
kubectl get pvc -n dev
```

Output:
```
NAME          STATUS  VOLUME      CAPACITY
project-pvc   Bound   project-pv  1Gi      ✅
```

---

## deployment.yaml 🚀

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: full-project
  namespace: dev
spec:
  replicas: 2
  selector:
    matchLabels:
      app: full-project
  template:
    metadata:
      labels:
        app: full-project
    spec:
      containers:
        - name: full-project
          image: nginx:latest
          ports:
            - containerPort: 80
          envFrom:
            - configMapRef:
                name: app-config      # inject all configmap keys
            - secretRef:
                name: app-secret      # inject all secret keys
          resources:
            requests:
              cpu: "100m"             # HPA needs this!
              memory: "128Mi"
            limits:
              cpu: "200m"
              memory: "256Mi"
          volumeMounts:
            - name: project-storage
              mountPath: /usr/share/nginx/html
      volumes:
        - name: project-storage
          persistentVolumeClaim:
            claimName: project-pvc   # use our PVC
```

```bash
kubectl apply -f deployment.yaml
kubectl get pods -n dev
```

---

## service.yaml 🌐

```yaml
apiVersion: v1
kind: Service
metadata:
  name: full-project-service
  namespace: dev
spec:
  type: ClusterIP
  selector:
    app: full-project
  ports:
    - port: 80
      targetPort: 80
```

```bash
kubectl apply -f service.yaml
kubectl get services -n dev
```

---

## ingress.yaml 🛣️

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: full-project-ingress
  namespace: dev
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: full-project-service
                port:
                  number: 80
```

```bash
kubectl apply -f ingress.yaml
kubectl get ingress -n dev
```

---

## hpa.yaml 📈

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: full-project-hpa
  namespace: dev
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: full-project
  minReplicas: 1
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
```

```bash
kubectl apply -f hpa.yaml
kubectl get hpa -n dev
```

---

## Apply Everything at Once ⚡

```bash
# Apply all files in folder
kubectl apply -f .

# Check everything
kubectl get all -n dev
```

Output:
```
NAME                              READY   STATUS
pod/full-project-xxx              1/1     Running ✅
pod/full-project-yyy              1/1     Running ✅

NAME                           TYPE        PORT
service/full-project-service   ClusterIP   80   ✅

NAME                       READY   UP-TO-DATE
deployment/full-project    2/2     2            ✅

NAME                              TARGETS
horizontalpodautoscaler/...       5%/50%       ✅
```

---

## Verify Everything Working 🧪

```bash
# Get pod name automatically
POD=$(kubectl get pods -n dev -l app=full-project -o jsonpath='{.items[0].metadata.name}')

# Check env variables
kubectl exec -it $POD -n dev -- bash -c "echo $APP_NAME"
# full-project ✅

kubectl exec -it $POD -n dev -- bash -c "echo $DB_HOST"
# dev-database ✅

kubectl exec -it $POD -n dev -- bash -c "echo $DB_PASSWORD"
# devpassword123 ✅

# Check PVC mounted
kubectl exec -it $POD -n dev -- bash -c "df -h /usr/share/nginx/html"
# shows mounted volume ✅

# Check ingress
curl http://<ingress-ip>/
# nginx welcome page ✅
```

---

## Full Summary Commands 🔍

```bash
# Everything in dev namespace
kubectl get all -n dev

# Storage
kubectl get pv
kubectl get pvc -n dev

# Config
kubectl get configmap -n dev
kubectl get secrets -n dev

# Networking
kubectl get ingress -n dev
kubectl get services -n dev

# Scaling
kubectl get hpa -n dev
```

---

## Full Component Map 🗺️

```
namespace: dev
    |
    ├── ConfigMap: app-config
    |     └── DB_HOST, LOG_LEVEL, APP_NAME
    |
    ├── Secret: app-secret
    |     └── DB_PASSWORD (base64 encoded)
    |
    ├── PVC: project-pvc -> PV: project-pv
    |     └── /usr/share/nginx/html
    |
    ├── Deployment: full-project
    |     ├── uses ConfigMap ✅
    |     ├── uses Secret ✅
    |     ├── uses PVC ✅
    |     └── has resource limits ✅
    |
    ├── Service: full-project-service
    |     └── routes traffic to pods
    |
    ├── Ingress: full-project-ingress
    |     └── routes / to service
    |
    └── HPA: full-project-hpa
          └── scales 1-5 pods on CPU > 50%
```

---

## Push to GitHub 🐙

```bash
cd ~/k8s-learning
git add .
git commit -m "day5: full k8s project all components connected"
git push
```

---

## Golden Rules 🏆

```
✅ Always apply namespace first
✅ Apply ConfigMap + Secret before Deployment
✅ Apply PV before PVC
✅ Apply PVC before Deployment
✅ Apply Service before Ingress
✅ Always verify with kubectl get all -n dev
✅ Use kubectl apply -f . to apply all at once
```

---

## K8s Learning Complete! 🎉

```
Day 1 ✅ - k3s setup, Pod, Deployment, Service, Namespaces
Day 2 ✅ - ConfigMaps, Secrets, Namespaces hands-on
Day 3 ✅ - Ingress, Helm Chart
Day 4 ✅ - Persistent Volumes, HPA
Day 5 ✅ - Full Project all components connected
```
---

*K8s Day 5 complete - Full Project with all components* 🚀
