# Kubernetes Learning - Day 4 🚀

---

## What We Covered Today 🎯

```
✅ Persistent Volumes (PV + PVC)
✅ Data survival test after pod restart
✅ Metrics Server installation
✅ HPA - Horizontal Pod Autoscaler
✅ Load testing HPA
✅ Auto scaling up and down
```

---

## Part 1 - Persistent Volumes 💾

---

## Why Persistent Volumes? 🤔

```
Pod dies → data inside pod = GONE ❌
Pod dies → data on PV     = SAFE ✅
```

Think of it like a hotel room:
```
Pod  = hotel room (temporary)
PV   = external hard drive (permanent)
PVC  = your request for that drive
```

Three things you need:
```
PV  = actual storage (the hard drive)
PVC = your request for storage
Pod = mounts PVC to use storage
```

---

## PV vs PVC vs Pod 🔍

| Object | What it is | Analogy |
|---|---|---|
| PV | Actual storage | Parking lot |
| PVC | Your claim to storage | Parking ticket |
| StorageClass | Type of storage | Covered vs open parking |

---

## Access Modes 🔐

| Mode | Meaning | Use |
|---|---|---|
| ReadWriteOnce | One pod reads + writes | Most apps |
| ReadOnlyMany | Many pods read only | Shared config |
| ReadWriteMany | Many pods read + write | Shared storage |

---

## pv.yaml 💽

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /tmp/k8s-data    # stored on node disk
```

```bash
kubectl apply -f pv.yaml
kubectl get pv
```

Output:
```
NAME    CAPACITY  STATUS      CLAIM
my-pv   1Gi       Available         ✅
```

---

## pvc.yaml 📋

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
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
kubectl get pvc -n dev
```

Output:
```
NAME     STATUS  VOLUME  CAPACITY
my-pvc   Bound   my-pv   1Gi      ✅
```

Bound = PVC found matching PV! 🎯

---

## Deployment With PVC 🚀

```yaml
# deployment-with-pvc.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-storage
  namespace: dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app-with-storage
  template:
    metadata:
      labels:
        app: app-with-storage
    spec:
      containers:
        - name: app
          image: nginx:latest
          ports:
            - containerPort: 80
          volumeMounts:
            - name: my-storage
              mountPath: /usr/share/nginx/html   # data saved here
      volumes:
        - name: my-storage
          persistentVolumeClaim:
            claimName: my-pvc                    # use our PVC
```

```bash
kubectl apply -f deployment-with-pvc.yaml
kubectl get pods -n dev
```

---

## Test Data Survives Pod Restart 🧪

```bash
# Write data to persistent storage
kubectl exec -it <pod-name> -n dev -- \
  bash -c "echo 'Hello from PV!' > /usr/share/nginx/html/test.txt"

# Verify data written
kubectl exec -it <pod-name> -n dev -- \
  cat /usr/share/nginx/html/test.txt
# Output: Hello from PV! ✅

# Delete pod - K8s will recreate it
kubectl delete pod <pod-name> -n dev

# Check data still there in new pod!
kubectl exec -it <new-pod-name> -n dev -- \
  cat /usr/share/nginx/html/test.txt
# Output: Hello from PV! ✅ Data survived!
```

---

## PV Commands ⚡

```bash
kubectl get pv
kubectl get pvc -n dev
kubectl describe pvc my-pvc -n dev
kubectl delete pvc my-pvc -n dev
```

---

## Important Note 💡

```
Deleting PVC -> PV still exists (Retain policy)
Deleting Pod -> PVC still exists
Deleting Deployment -> PVC still exists

Data is safe even when pods come and go! ✅
```

---

## Part 2 - HPA Auto Scaling 📈

---

## Why HPA? 🤔

```
Without HPA:
9 AM  -> traffic spikes -> manually add pods 😴
3 AM  -> no traffic    -> wasting money 💸

With HPA:
9 AM  -> traffic spikes -> auto adds pods ✅
3 AM  -> no traffic    -> auto removes pods 💰
```

Like a restaurant:
```
Lunch rush -> call more waiters
Night time -> send waiters home
All automatic! 🎯
```

---

## How HPA Works 🔄

```
HPA watches pods every 15 seconds
    |
    ├── CPU too high? -> add more pods
    ├── CPU too low?  -> remove pods
    └── stays within min and max replicas you set
```

---

## Install Metrics Server First 🔧

HPA needs metrics server to measure CPU:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Check running
kubectl get pods -n kube-system | grep metrics

# Test
kubectl top nodes
kubectl top pods -n dev
```

---

## Deployment With Resource Limits 📊

HPA needs resource requests to calculate percentages:

```yaml
# hpa-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hpa-demo
  namespace: dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hpa-demo
  template:
    metadata:
      labels:
        app: hpa-demo
    spec:
      containers:
        - name: hpa-demo
          image: nginx:latest
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "100m"       # 0.1 CPU - HPA uses this!
              memory: "128Mi"
            limits:
              cpu: "200m"       # max 0.2 CPU
              memory: "256Mi"
```

Without resources.requests -> HPA cannot work! ❌

---

## CPU Units Explained 💡

```
1000m = 1 CPU core
500m  = 0.5 CPU core
200m  = 0.2 CPU core
100m  = 0.1 CPU core (good starting point)
```

---

## hpa.yaml 📈

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-demo
  namespace: dev
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hpa-demo          # watch this deployment
  minReplicas: 1            # never go below 1
  maxReplicas: 5            # never go above 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50   # scale if CPU > 50%
```

```bash
kubectl apply -f hpa.yaml
kubectl get hpa -n dev
```

Output:
```
NAME       TARGETS   MINPODS   MAXPODS   REPLICAS
hpa-demo   5%/50%    1         5         1
```

---

## Test HPA With Load Test 🔥

Terminal 1 - Watch HPA:
```bash
kubectl get hpa -n dev -w
```

Terminal 2 - Generate load:
```bash
kubectl run load-test \
  --image=busybox \
  --restart=Never \
  -n dev \
  -- /bin/sh -c "while true; do wget -q -O- http://hpa-demo-service; done"
```

Watch scaling happen:
```
TARGETS    REPLICAS
5%/50%     1        <- before load
85%/50%    1        <- load started
85%/50%    3        <- HPA scaling up! 🚀
60%/50%    4        <- still scaling
45%/50%    4        <- stabilizing ✅
```

Stop load and watch scale down:
```bash
kubectl delete pod load-test -n dev
```

```
45%/50%    4
20%/50%    2
5%/50%     1        <- back to 1 pod 💰
```

---

## HPA With Memory Scaling 🧠

```yaml
metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70
```

---

## HPA Commands ⚡

```bash
kubectl get hpa -n dev
kubectl get hpa -n dev -w          # watch live
kubectl describe hpa hpa-demo -n dev
kubectl delete hpa hpa-demo -n dev
kubectl top pods -n dev            # check resource usage
kubectl top nodes                  # check node usage
```

---

## PV vs HPA - When to Use What 🤔

| | Persistent Volume | HPA |
|---|---|---|
| Purpose | Save data permanently | Scale pods automatically |
| Use for | Databases, file storage | Web apps, APIs |
| Solves | Data loss on pod restart | Manual scaling problem |

---

## Folder Structure 📂

```
k8s-learning/
└── Day-4/
      ├── pv.yaml                  ✅
      ├── pvc.yaml                 ✅
      ├── deployment-with-pvc.yaml ✅
      ├── hpa-deployment.yaml      ✅
      └── hpa.yaml                 ✅
```

---

## Push to GitHub 🐙

```bash
cd ~/k8s-learning
git add .
git commit -m "day4: persistent volumes and HPA hands-on"
git push
```

---

## Golden Rules 🏆

```
✅ Always use PV for data that must survive pod restarts
✅ Always set resource requests before creating HPA
✅ Set minReplicas to at least 1 (never 0 in production)
✅ Set maxReplicas based on your budget
✅ Delete load-test pod after testing
✅ Use kubectl top to verify metrics server working
```

---

## Day 4 Summary ✅

```
✅ PV created with hostPath storage
✅ PVC created and bound to PV
✅ Deployment with PVC mounted
✅ Data survived pod restart test
✅ Metrics server installed
✅ HPA deployment with resource limits
✅ HPA created (1-5 replicas, 50% CPU)
✅ Load test done - watched auto scaling
✅ Scale down verified after load stopped
✅ Pushed to GitHub
```

---

*K8s Day 4 complete - Persistent Volumes, PVC, HPA Auto Scaling* 🚀
