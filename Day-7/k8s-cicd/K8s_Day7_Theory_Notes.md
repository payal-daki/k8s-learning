# Kubernetes with CI/CD 🚀

---

## What We Built 🎯

```
Developer pushes code to GitHub
        |
        v
GitHub Actions triggers
        |
        v
Build Docker image
        |
        v
Push image to ECR (AWS registry)
        |
        v
Deploy to EKS automatically
        |
        v
App updated with zero downtime ✅
```

---

## Why CI/CD for K8s? 🤔

```
Without CI/CD:
-> Build image manually
-> Push to registry manually
-> kubectl apply manually
-> Error prone, slow ❌

With CI/CD:
-> Push code to GitHub
-> Everything happens automatically ✅
```

---

## Tools Used 🔧

```
GitHub Actions -> pipeline
ECR            -> Docker image registry (AWS)
EKS            -> K8s cluster on AWS
Docker         -> build image
kubectl        -> deploy to K8s
```

---

## Folder Structure 📂

```
k8s-learning/
└── Day-7/
      └── k8s-cicd/
            ├── app/
            │     ├── Dockerfile
            │     └── index.html
            ├── k8s/
            │     ├── deployment.yaml
            │     └── service.yaml
            └── .github/
                  └── workflows/
                        └── deploy.yml
```

---

## app/index.html 🌐

```html
<!DOCTYPE html>
<html>
<head><title>K8s CI/CD Demo</title></head>
<body>
  <h1>Hello from K8s CI/CD! 🚀</h1>
  <p>Version: 1.0.0</p>
</body>
</html>
```

---

## app/Dockerfile 🐳

```dockerfile
FROM nginx:latest
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
```

---

## k8s/deployment.yaml 🚀

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cicd-demo
  namespace: dev
spec:
  replicas: 2
  selector:
    matchLabels:
      app: cicd-demo
  template:
    metadata:
      labels:
        app: cicd-demo
    spec:
      containers:
        - name: cicd-demo
          image: AWS_ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com/cicd-demo:latest
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

---

## k8s/service.yaml 🌐

```yaml
apiVersion: v1
kind: Service
metadata:
  name: cicd-demo-service
  namespace: dev
spec:
  type: LoadBalancer
  selector:
    app: cicd-demo
  ports:
    - port: 80
      targetPort: 80
```

---

## Create ECR Repository 📦

```bash
aws ecr create-repository \
  --repository-name cicd-demo \
  --region ap-south-1
```

Note the URI from output:
```
xxx.dkr.ecr.ap-south-1.amazonaws.com/cicd-demo
```

---

## GitHub Secrets Required 🔒

```
GitHub -> repo -> Settings -> Secrets -> Actions

Add these:
AWS_ACCESS_KEY_ID      -> your key
AWS_SECRET_ACCESS_KEY  -> your secret
AWS_ACCOUNT_ID         -> your AWS account ID
```

---

## .github/workflows/deploy.yml ⚙️

```yaml
name: K8s CI/CD Pipeline

on:
  push:
    branches:
      - main

env:
  AWS_REGION: ap-south-1
  ECR_REPOSITORY: cicd-demo
  EKS_CLUSTER: dev-cluster
  NAMESPACE: dev

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      # Checkout repository
      - name: Checkout code
        uses: actions/checkout@v4

      # Configure AWS Credentials
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      # Login to Amazon ECR
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      # Build Docker Image
      - name: Build Docker Image
        run: |
          docker build -t $ECR_REPOSITORY ./Day-7/k8s-cicd/app

      # Tag Image
      - name: Tag Docker Image
        run: |
          ECR_URI=${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{ env.ECR_REPOSITORY }}
          docker tag $ECR_REPOSITORY:latest $ECR_URI:latest
          docker tag $ECR_REPOSITORY:latest $ECR_URI:${{ github.sha }}
          echo "ECR_URI=$ECR_URI" >> $GITHUB_ENV

      # Push Image to ECR
      - name: Push Docker Image
        run: |
          docker push $ECR_URI:latest
          docker push $ECR_URI:${{ github.sha }}

      # Update kubeconfig for EKS
      - name: Configure kubectl
        run: |
          aws eks update-kubeconfig \
            --name ${{ env.EKS_CLUSTER }} \
            --region ${{ env.AWS_REGION }}

      # Create Namespace if not exists
      - name: Create Namespace
        run: |
          kubectl create namespace ${{ env.NAMESPACE }} \
            --dry-run=client -o yaml | kubectl apply -f -

      # Deploy Kubernetes Manifests
      - name: Deploy Application
        run: |
          kubectl apply -f ./Day-7/k8s-cicd/k8s/deployment.yaml -n ${{ env.NAMESPACE }}
          kubectl apply -f ./Day-7/k8s-cicd/k8s/service.yaml -n ${{ env.NAMESPACE }}

      # Update Deployment Image to latest SHA
      - name: Update Deployment Image
        run: |
          kubectl set image deployment/cicd-demo \
            cicd-demo=$ECR_URI:${{ github.sha }} \
            -n ${{ env.NAMESPACE }}

      # Wait for Rollout to complete
      - name: Wait for Rollout
        run: |
          kubectl rollout status deployment/cicd-demo \
            -n ${{ env.NAMESPACE }} \
            --timeout=180s

      # Show Service URL
      - name: Get Service
        run: |
          kubectl get svc -n ${{ env.NAMESPACE }}
```

---

## Pipeline Steps Explained 📋

| Step | What happens |
|---|---|
| Checkout | Gets code from GitHub |
| AWS Auth | Configures AWS credentials |
| ECR Login | Authenticates to ECR |
| Build | Creates Docker image |
| Tag | Tags with latest + commit SHA |
| Push | Uploads image to ECR |
| kubeconfig | Connects kubectl to EKS |
| Namespace | Creates dev namespace if missing |
| Deploy | Applies K8s manifests |
| Update image | Sets new image SHA on deployment |
| Rollout | Waits 180s for zero downtime update |
| Get Service | Shows ELB URL |

---

## Rolling Update - Zero Downtime 🔄

```
Before push:
Pod 1 -> image:abc (old)
Pod 2 -> image:abc (old)

After push:
Pod 1 -> image:xyz (new) ✅  <- updated first
Pod 2 -> image:abc (old)     <- still running

Pod 1 -> image:xyz (new) ✅
Pod 2 -> image:xyz (new) ✅  <- updated second

Zero downtime! 🎯
```

---

## Image Tagging Strategy 🏷️

```
latest    -> always latest image
commit SHA -> specific version (abc123def)

Why both?
latest -> easy reference
SHA    -> rollback to exact version ✅
```

---

## Rollback if Something Breaks ↩️

```bash
# Rollback to previous version
kubectl rollout undo deployment/cicd-demo -n dev

# See rollout history
kubectl rollout history deployment/cicd-demo -n dev

# Rollback to specific revision
kubectl rollout undo deployment/cicd-demo \
  --to-revision=2 -n dev
```

---

## Full Flow on Every git push 🔁

```
git push to main
    |
    v
GitHub Actions triggers
    |
    v
Docker image built from ./app
    |
    v
Image tagged (latest + SHA)
    |
    v
Image pushed to ECR
    |
    v
kubectl configured for EKS
    |
    v
K8s manifests applied
    |
    v
Deployment image updated to new SHA
    |
    v
Rolling update (180s timeout)
    |
    v
New pods running ✅
Zero downtime 🎯
```

---

## EKS Cluster Setup 🔧

```bash
# Create cluster
eksctl create cluster \
  --name dev-cluster \
  --region ap-south-1 \
  --nodegroup-name dev-nodes \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --managed \
  --timeout 40m

# Verify
kubectl get nodes
eksctl get cluster
```

---

## Cleanup After Practice 💰

```bash
# Delete K8s resources
kubectl delete -f ./Day-7/k8s-cicd/k8s/ -n dev

# Delete EKS cluster
eksctl delete cluster \
  --name dev-cluster \
  --region ap-south-1

# Delete ECR repo
aws ecr delete-repository \
  --repository-name cicd-demo \
  --force \
  --region ap-south-1
```

---

## Golden Rules 🏆

```
✅ Always use commit SHA for image tagging
✅ Never use only latest tag in production
✅ Always wait for rollout status
✅ Keep secrets in GitHub Secrets never in code
✅ Always cleanup EKS after practice (expensive!)
✅ Use --timeout 40m for eksctl (avoid timeout errors)
✅ Always have rollback plan ready
✅ Use --dry-run=client for namespace creation
```

---

## Summary 🧠

```
ECR      = Docker image storage on AWS
EKS      = K8s cluster on AWS
Pipeline = build -> push -> deploy automatically
SHA tag  = every commit has unique image version
Rolling  = zero downtime deployments
Rollback = one command to go back
```

---

## Complete K8s + CI/CD Learning ✅

```
Day 1 - k3s setup, Pod, Deployment, Service
Day 2 - ConfigMaps, Secrets, Namespaces
Day 3 - Ingress, Helm
Day 4 - Persistent Volumes, HPA
Day 5 - Full K8s Project
Day 6 - EKS on AWS
Day 7 - K8s + CI/CD Pipeline (GitHub Actions + ECR + EKS)
```

---

*K8s CI/CD complete - GitHub Actions + ECR + EKS = full automated pipeline* 🚀
