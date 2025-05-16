
# Deploy NGINX to Kubernetes Using Argo CD – Practical Guide

## Prerequisites

- Access to a Kubernetes cluster
- `kubectl`, `helm`, and `argocd` CLIs installed
- GitHub account and a Git repository

## Step 1: Prepare Kubernetes Manifests

1. Clone your GitHub repo:

```bash
git clone https://github.com/<your-repo>/<repo>.git
cd <repo>
```

2. Create the following manifest files:

**i. namespace.yaml**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: argo-demo
```

**ii. deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: argo-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

**iii. service.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: argo-demo
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

3. Push the changes to GitHub:

```bash
git add .
git commit -m "Initial Kubernetes manifests for Argo CD demo"
git push
```

## Step 2: Install Argo CD CLI

```bash
wget https://github.com/argoproj/argo-cd/releases/download/v2.6.1/argocd-linux-amd64
chmod +x argocd-linux-amd64
sudo mv argocd-linux-amd64 /usr/bin/argocd
```

Verify installation:
```bash
argocd version
```

## Step 3: Install Argo CD in Kubernetes

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Check deployment status:
```bash
kubectl get deployments -n argocd
```

## Step 4: Access Argo CD Dashboard

1. Get initial credentials:
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
Username: `admin`

Password:
```bash
argocd admin initial-password -n argocd
```

2. Access the dashboard at: http://localhost:8080

3. Update your password after logging in.

4. Delete the initial password secret:
```bash
kubectl delete secret argocd-initial-admin-secret -n argocd
```

5. Login via CLI:
```bash
argocd login localhost:8080
```

## Step 5: Create the Argo CD Application

```bash
argocd app create argo-demo \
  --repo https://github.com/<username>/<repo>.git \
  --path . \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace argo-demo
```

View applications:
```bash
argocd app list
```

## Step 6: Sync and Deploy

Sync from CLI:
```bash
argocd app sync argo-demo
```

Verify deployment:
```bash
kubectl get deployment -n argo-demo
```

## Step 7: Update the Application

1. Edit `deployment.yaml` to scale replicas:
```yaml
spec:
  replicas: 5
```

2. Push changes:
```bash
git add .
git commit -m "Scale to 5 replicas"
git push
```

3. Sync again:
```bash
argocd app sync argo-demo
```

## Verification

Check the updated deployment:
```bash
kubectl get pods -n argo-demo
```
```

This version includes:
1. Proper Markdown formatting
2. Consistent code block syntax
3. Better organization with clear section headers
4. Added verification steps
5. Fixed some syntax issues in the original
6. Added proper YAML formatting
7. Improved readability with consistent spacing

You can copy this directly into a README.md file in your repository.
