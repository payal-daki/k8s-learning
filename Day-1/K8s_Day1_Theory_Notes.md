# Kubernetes Learning - Day 1

---

## Why Kubernetes Exists

Without K8s - manual server management:
- App crashes -> down until you restart manually
- Traffic spike -> server struggles
- Update needed -> downtime
- Scale needed -> manual work

With K8s - automated container management:
- App crashes -> auto restarts
- Traffic spike -> auto scales
- Update -> zero downtime rolling update
- Scale -> one command

---

## What is an Orchestrator?

Orchestrator = conductor of containers.

Like a music conductor manages 50+ musicians,
K8s manages 50+ containers automatically.

Handles:
- Scheduling (which machine runs which container)
- Health checking (detects crashes, restarts)
- Scaling (adds/removes containers based on load)
- Networking (containers talk to each other)
- Updates (replaces old containers safely)

---

## Cluster vs Node

```
Cluster = entire K8s environment (the whole kingdom)
Node    = single machine/EC2 inside cluster (one city)
```

Node is a machine. Cluster is a group of machines working together.

---

## Control Plane - The Brain (Master Node)

Your app never runs here. Only management components.

| Component | Job |
|---|---|
| API Server | Entry point for all commands |
| etcd | Database - stores entire cluster state |
| Scheduler | Assigns pods to nodes |
| Controller Manager | Keeps desired state = actual state |

```
YOU -> kubectl -> API Server -> rest of K8s
```

---

## Data Plane - The Muscle (Worker Nodes)

Your actual app runs here.

| Component | Job |
|---|---|
| Kubelet | Agent - talks to master, manages pods |
| Kube-proxy | Routes network traffic |
| Container Runtime | Actually runs containers (containerd) |

---

## Pod

Smallest unit in Kubernetes.
Pod = wrapper around your container.

- 1 Pod = usually 1 container
- Every container runs inside a Pod
- Pod crashes -> K8s creates new one automatically

---

## Deployment

Never use raw Pods in real life.
Use Deployments instead.

```
Deployment -> ReplicaSet -> Pods
```

| Feature | Raw Pod | Deployment |
|---|---|---|
| Self healing | No | Yes |
| Scaling | No | Yes |
| Rolling updates | No | Yes |
| Rollback | No | Yes |

---

## Service

Stable address in front of pods.
Pod IPs change every restart. Service IP never changes.

| Type | Access | Use |
|---|---|---|
| ClusterIP | Internal only | App to app |
| NodePort | NodeIP:Port | Dev/Testing |
| LoadBalancer | Public IP | Production |

Finds pods using Labels and Selectors.

---

## Namespace

Virtual partition inside cluster.
Organizes and isolates resources.

```
CLUSTER
├── namespace: dev
├── namespace: prod
└── namespace: monitoring
```

---

## kubectl - CLI Tool

Every command hits API Server.

```bash
kubectl get pods                     # view pods
kubectl get pods -A                  # all namespaces
kubectl get deployments              # view deployments
kubectl get services                 # view services
kubectl get all                      # everything

kubectl apply -f file.yaml           # create/update
kubectl delete -f file.yaml          # delete
kubectl delete pod <name>            # delete pod

kubectl describe pod <name>          # full details
kubectl logs <pod-name>              # pod logs
kubectl logs -f <pod-name>           # live logs
kubectl exec -it <pod-name> -- bash  # SSH into pod

kubectl scale deployment <name> --replicas=3
kubectl get pods -w                  # watch live
```

---

## Complete Flow - What Happens on kubectl apply

```
1. You run kubectl apply -f deployment.yaml
2. Hits API Server
3. Stored in etcd
4. Scheduler picks a Worker Node
5. Kubelet on that node gets the order
6. Kubelet tells Container Runtime
7. Container Runtime starts the Pod
8. Kubelet reports back "Pod is running"
```

---

## Full Architecture

```
YOU
 └── kubectl
       └── API Server (Master)
             ├── etcd
             ├── Scheduler
             └── Controller Manager
                   │
                   └── Worker Nodes (Data Plane)
                         ├── Kubelet
                         ├── Kube-proxy
                         └── Pods (inside Namespaces)
                               └── managed by Deployments
                                     └── exposed by Services
```

---

## k3s - Lightweight K8s for Learning

Full K8s is heavy. k3s is lightweight version.
Same Kubernetes - just stripped of heavy components.
Used in actual production by many companies.

```bash
# Install k3s
curl -sfL https://get.k3s.io | sh -

# Check status
sudo systemctl status k3s

# Check node
sudo kubectl get nodes
```

---

## Setup kubectl Without sudo

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config

# Now works without sudo
kubectl get nodes
```

---

## Hands-on - First Pod

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-first-pod
  labels:
    app: my-app
spec:
  containers:
    - name: my-app
      image: nginx:latest
      ports:
        - containerPort: 80
```

```bash
kubectl apply -f pod.yaml
kubectl get pods
kubectl describe pod my-first-pod
kubectl logs my-first-pod
```

---

## Hands-on - First Deployment

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
  namespace: default
spec:
  replicas: 3
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
```

```bash
kubectl apply -f deployment.yaml
kubectl get deployments
kubectl get pods
kubectl get pods -w
```

---

## Hands-on - Self Healing Test

```bash
# Delete one pod
kubectl delete pod my-deployment-7d6f8-abc12

# Watch K8s recreate automatically
kubectl get pods -w
```

K8s immediately creates new pod to maintain 3 replicas.

---

## Hands-on - First Service

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: default
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

```bash
kubectl apply -f service.yaml
kubectl get services
```

Access app:
```
http://<ec2-public-ip>:30080
```

Open port 30080 in EC2 Security Group first!

---

## Hands-on - Scaling

```bash
# Scale up
kubectl scale deployment my-deployment --replicas=5

# Scale down
kubectl scale deployment my-deployment --replicas=1

# Check
kubectl get pods
```

---

## Hands-on - Namespaces

```bash
# Create namespaces
kubectl create namespace dev
kubectl create namespace prod

# Deploy in specific namespace
kubectl apply -f deployment.yaml -n dev
kubectl apply -f deployment.yaml -n prod

# View per namespace
kubectl get pods -n dev
kubectl get pods -n prod
kubectl get pods -A
```

---

## YAML File Structure Explained

```yaml
apiVersion: apps/v1      # which K8s API version
kind: Deployment         # what type of resource
metadata:                # info about the resource
  name: my-deployment
  namespace: default
spec:                    # desired state
  replicas: 3
  selector:
    matchLabels:
      app: my-app        # find pods with this label
  template:              # pod template
    metadata:
      labels:
        app: my-app      # label on pods
    spec:
      containers:
        - name: my-app
          image: nginx
          ports:
            - containerPort: 80
```

---

## Day 1 Summary

```
k3s installed on EC2           -> single node K8s cluster
kubectl configured             -> no sudo needed
First pod created              -> nginx running
First deployment created       -> 3 replicas
Self healing tested            -> pod deleted, recreated auto
First service created          -> app accessible on port 30080
Scaling tested                 -> up and down
Namespaces created             -> dev + prod isolated
Code pushed to GitHub
```

---

## Key Concepts

```
Cluster          -> entire K8s environment
Node             -> single machine
Pod              -> smallest unit, wraps container
Deployment       -> manages pods, self healing
Service          -> stable network access
Namespace        -> isolation (dev/prod)
kubectl          -> CLI to control cluster
k3s              -> lightweight K8s for learning
```

---

*K8s Day 1 complete - k3s setup, Pod, Deployment, Service, Namespaces*
