# Progressive-Delivery-on-Kubernetes-with-Argo-Rollouts
![image](https://github.com/user-attachments/assets/7891a488-1ad1-48d5-b299-46a0df746133)



![image](https://github.com/user-attachments/assets/ba0a7a78-f6b9-4ec6-9b23-987f58f19336)


---

```markdown
# Progressive Delivery on Kubernetes with Argo Rollouts (EKS)

This guide covers deploying an application to an EKS cluster using progressive delivery strategies (e.g., Blue/Green or Canary) with Argo Rollouts and ArgoCD.

---

## Step 1: Clone the Project Repository

```bash
git clone https://github.com/lily4499/blue-green-eks.git
cd blue-green-eks
```

Ensure that:
- All manifests (deployments, rollouts, services) are in place
- Source code and Dockerfile are available
- Argo Rollouts manifest is present

---

## Step 2: Provision the EKS Cluster

Provision your cluster using Terraform or AWS CLI. Example using eksctl:

```bash
eksctl create cluster --name rollout-cluster --region us-east-1 --nodes 2
aws eks --region us-east-1 update-kubeconfig --name rollout-cluster
```

---

## Step 3: Test the Application Locally

### Backend (Node.js Example)

```bash
npm install
npm start
```

Open `http://localhost:3000` in your browser to confirm it's working.

---

## Step 4: Create CI Pipeline (Docker Build & Push)

### Docker Build and Push

```bash
# Authenticate to DockerHub
docker login

# Build the image
docker build -t <your-dockerhub-username>/sample-app:1 .

# Push to DockerHub
docker push <your-dockerhub-username>/sample-app:1
```

Ensure your `Deployment` or `Rollout` YAML references this image tag.

---

## Step 5: Create Jenkins Pipeline Jobs (CI/CD)

- Create a Jenkins pipeline for building and pushing images.
- Create another pipeline to update the rollout manifests with the new image tag (optional).
- Set up webhook triggers from GitHub to Jenkins.

---

## Step 6: Deploy ArgoCD

### Install ArgoCD in the Cluster

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Install ArgoCD CLI

```bash
# On Mac/Linux with brew
brew install argocd
```

### Access ArgoCD UI

Choose one:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

or patch service to LoadBalancer:

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

### Get Login Credentials

```bash
# Username: admin
argocd admin initial-password -n argocd
```

Visit `http://localhost:8080` or LoadBalancer IP and log in.

---

## Step 7: Deploy Argo Rollouts Controller

### Install Argo Rollouts

```bash
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
```

### Install Argo Rollouts CLI Plugin

```bash
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
chmod +x ./kubectl-argo-rollouts-linux-amd64
sudo mv ./kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts
```

Confirm installation:

```bash
kubectl argo rollouts version
```

---

## Step 8: Deploy Application with Rollout Strategy

Ensure your Kubernetes manifests use `kind: Rollout` instead of `Deployment`.

Example command to sync via ArgoCD:

```bash
argocd app create rollout-app \
  --repo https://github.com/lily4499/blue-green-eks.git \
  --path manifests \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default

argocd app sync rollout-app
```

---

## Step 9: Access Argo Rollouts Dashboard

Start dashboard:

```bash
kubectl argo rollouts dashboard
```

Open browser:

```
http://localhost:3100/rollouts
```

---

## Step 10: Control the Rollout Process

View current rollout status:

```bash
kubectl argo rollouts get rollout sample-app
```

Manually promote to the next step:

```bash
kubectl argo rollouts promote sample-app
```

Undo to previous version:

```bash
kubectl argo rollouts undo sample-app
```

---

## Step 11: Make Code Changes and Observe Progressive Delivery

1. Push a code change to GitHub.
2. Jenkins rebuilds and pushes a new Docker image.
3. Jenkins or GitHub Action updates the image tag in rollout manifest.
4. ArgoCD detects the change and triggers rollout.
5. Observe rollout progress in the dashboard.

---
##  Summary

You’ve now completed a full progressive delivery setup using:

- EKS (Kubernetes Cluster)
- Docker & Jenkins (CI)
- ArgoCD (CD)
- Argo Rollouts (Progressive Deployment)
- GitHub (Source Control)

---

---

Here's how to **add and manage Canary deployment strategies** with **Argo Rollouts** on your EKS cluster. This update extends our project to support gradual, controlled traffic shifting with analysis-based promotions or manual gates.

---

```markdown
## Step 12: Add a Canary Strategy with Argo Rollouts

To enable canary deployments, update your existing `sample-app-rollout.yaml` manifest to include the **Canary strategy**.

### Example: Canary Rollout Manifest

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: sample-app
  namespace: default
spec:
  replicas: 3
  revisionHistoryLimit: 2
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
        - name: sample-app
          image: <your-dockerhub-username>/sample-app:1
          ports:
            - containerPort: 3000
  strategy:
    canary:
      steps:
        - setWeight: 20
        - pause: { duration: 30s }
        - setWeight: 50
        - pause: { duration: 1m }
        - setWeight: 100
```

>  Replace `<your-dockerhub-username>/sample-app:1` with the actual image path and tag.

---

## Step 13: Apply and Sync the Canary Rollout

```bash
kubectl apply -f sample-app-rollout.yaml
argocd app sync rollout-app
```

Monitor rollout progress:

```bash
kubectl argo rollouts get rollout sample-app --watch
```

---

## Step 14: Manually Promote Canary Steps (Optional)

```bash
kubectl argo rollouts promote sample-app
```

To undo the current rollout:

```bash
kubectl argo rollouts undo sample-app
```

---


```



## Summary of Canary Deployment Flow

1. New image is pushed (via Jenkins/GitHub Actions)
2. ArgoCD syncs the updated rollout manifest
3. Argo Rollouts initiates canary steps:
   - 20% traffic → pause
   - 50% traffic → pause
   - 100% traffic
4. Promotion can be manual or automatic (based on analysis)

---

## Bonus: Visualize Rollout with Argo Dashboard

```bash
kubectl argo rollouts dashboard
```

Then open:

```
http://localhost:3100/rollouts
```

Watch rollout states and trigger actions directly from the UI.

---

