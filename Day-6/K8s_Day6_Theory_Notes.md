# Kubernetes Learning - Day 6 - EKS ☁️

---

## What is EKS? 🤔

EKS = Elastic Kubernetes Service = AWS managed Kubernetes.

```
Self managed K8s (k3s):
├── You install master
├── You manage control plane
├── You handle updates
└── You fix if master crashes

EKS:
├── AWS manages control plane ✅
├── AWS handles master updates ✅
├── AWS fixes master issues ✅
└── You only manage worker nodes
```

EKS = Kubernetes where AWS babysits the master for you! 🎯

---

## EKS Architecture ☁️

```
AWS Managed (you don't touch):
├── API Server
├── etcd
├── Scheduler
└── Controller Manager

You Manage:
└── Worker Nodes (EC2 instances)
      └── Node Group (t3.medium x2)
```

---

## EKS vs k3s ⚡

| | k3s | EKS |
|---|---|---|
| Setup | 1 command | 15-20 mins |
| Master | You manage | AWS manages |
| Cost | Just EC2 | EC2 + $0.10/hr |
| Production | Small projects | Enterprise |
| AWS integration | Manual | Built-in |
| Load Balancer | Manual | Auto creates ELB |

---

## Cost Warning! 💰

```
EKS cluster  -> $0.10/hour = ~$72/month
Worker nodes -> t3.medium x2 = ~$60/month
NAT Gateway  -> ~$32/month
Total        -> ~$164/month

Always delete after practice!
```

---

## Prerequisites Check ✅

```bash
terraform --version
kubectl version --client
aws --version
aws sts get-caller-identity
```

---

## Step 1 - Install eksctl 🔧

```bash
curl --silent --location \
  "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" \
  | tar xz -C /tmp

sudo mv /tmp/eksctl /usr/local/bin

# Verify
eksctl version
```

---

## Step 2 - IAM Permissions 🔒

IAM user needs:
```
AmazonEKSClusterPolicy
AmazonEKSWorkerNodePolicy
AmazonEC2FullAccess
IAMFullAccess
AWSCloudFormationFullAccess
AmazonVPCFullAccess
```

For learning only:
```
AWS Console -> IAM -> Users -> your-user
-> Add permissions -> AdministratorAccess
```

---

## Step 3 - Create EKS Cluster 🚀

```bash
eksctl create cluster \
  --name dev-cluster \
  --region ap-south-1 \
  --nodegroup-name dev-nodes \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --managed
```

Takes 15-20 minutes! ⏳

What eksctl creates automatically:
```
CloudFormation Stack
├── VPC
├── Public + Private Subnets
├── Internet Gateway
├── NAT Gateway
├── Security Groups
├── EKS Cluster
└── Node Group (EC2 instances)
```

---

## Step 4 - Verify Cluster ✅

```bash
# Check cluster
eksctl get cluster

# Check nodes
kubectl get nodes

# Check all pods
kubectl get pods -A
```

Output:
```
NAME                                STATUS
ip-192-168-xxx.ap-south-1.ec2...   Ready  ✅
ip-192-168-yyy.ap-south-1.ec2...   Ready  ✅
```

---

## Step 5 - Deploy App on EKS 🚀

### deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: eks-demo
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: eks-demo
  template:
    metadata:
      labels:
        app: eks-demo
    spec:
      containers:
        - name: eks-demo
          image: nginx:latest
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "200m"
              memory: "256Mi"
```

```bash
kubectl apply -f deployment.yaml
kubectl get pods
```

---

## Step 6 - LoadBalancer Service 🌐

On EKS - LoadBalancer automatically creates AWS ELB!

### service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: eks-demo-service
spec:
  type: LoadBalancer        # AWS creates ELB automatically!
  selector:
    app: eks-demo
  ports:
    - port: 80
      targetPort: 80
```

```bash
kubectl apply -f service.yaml

# Watch ELB being created (2-3 mins)
kubectl get service eks-demo-service -w
```

Output:
```
NAME               TYPE           EXTERNAL-IP
eks-demo-service   LoadBalancer   xxx.ap-south-1.elb.amazonaws.com ✅
```

```bash
# Access your app!
curl http://xxx.ap-south-1.elb.amazonaws.com
# nginx welcome page ✅
```

---

## Step 7 - Verify on AWS Console 🖥️

```
EC2 -> Load Balancers -> ELB created automatically ✅
EKS -> Clusters -> dev-cluster -> see details ✅
```

---

## Step 8 - Namespaces on EKS 📁

Works exactly same as k3s!

```bash
kubectl create namespace dev
kubectl create namespace prod

kubectl apply -f deployment.yaml -n dev
kubectl get pods -n dev
```

---

## Step 9 - Scaling on EKS 📈

```bash
# Scale deployment
kubectl scale deployment eks-demo --replicas=4
kubectl get pods -w

# Scale node group (add more EC2s)
eksctl scale nodegroup \
  --cluster dev-cluster \
  --name dev-nodes \
  --nodes 3 \
  --region ap-south-1
```

---

## kubectl Config for EKS 🔧

```bash
# Update kubeconfig to talk to EKS
aws eks update-kubeconfig \
  --name dev-cluster \
  --region ap-south-1

# Verify connected
kubectl config current-context

# See all contexts
kubectl config get-contexts

# Switch between clusters
kubectl config use-context <context-name>
```

---

## eksctl Commands ⚡

```bash
# List clusters
eksctl get cluster

# List node groups
eksctl get nodegroup --cluster dev-cluster

# Scale node group
eksctl scale nodegroup \
  --cluster dev-cluster \
  --name dev-nodes \
  --nodes 3

# Upgrade cluster
eksctl upgrade cluster \
  --name dev-cluster \
  --approve

# Delete cluster (always do this after practice!)
eksctl delete cluster \
  --name dev-cluster \
  --region ap-south-1
```

---

## Important - Always Delete After Practice! 🗑️

```bash
eksctl delete cluster \
  --name dev-cluster \
  --region ap-south-1
```

This deletes:
```
✅ EKS cluster
✅ Node groups (EC2s)
✅ VPC + subnets
✅ NAT Gateway
✅ Load Balancers
✅ Everything eksctl created
```

---

## Push to GitHub 🐙

```bash
cd ~/k8s-learning
git add .
git commit -m "day6: EKS cluster setup and deployment"
git push
```

---

## EKS vs k8s Key Difference 💡

```
k3s/k8s:
kubectl apply -> pods created on YOUR nodes
Service: LoadBalancer -> needs manual setup

EKS:
kubectl apply -> pods created on AWS EC2 nodes
Service: LoadBalancer -> AWS auto creates ELB ✅
```

---

## Golden Rules 🏆

```
✅ Always delete EKS cluster after practice (expensive!)
✅ Use t3.medium minimum for worker nodes
✅ Always run aws sts get-caller-identity before creating cluster
✅ eksctl creates everything - VPC, subnets, nodes
✅ kubectl works same way on EKS as k3s
✅ LoadBalancer type service = auto ELB on AWS
```

---

## Day 6 Summary ✅

```
✅ Understood EKS vs self-managed K8s
✅ eksctl installed
✅ EKS cluster created (15-20 mins)
✅ Worker nodes verified (2 nodes)
✅ App deployed on EKS
✅ LoadBalancer service created
✅ AWS ELB created automatically
✅ Accessed app via ELB URL
✅ Namespaces + scaling practiced
✅ Cluster deleted after practice
✅ Pushed to GitHub
```

---

*K8s Day 6 complete - EKS Cluster Setup, Deployment, LoadBalancer* 🚀
