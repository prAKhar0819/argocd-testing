
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


This version includes:
1. Proper Markdown formatting
2. Consistent code block syntax
3. Better organization with clear section headers
4. Added verification steps
5. Fixed some syntax issues in the original
6. Added proper YAML formatting
7. Improved readability with consistent spacing

You can copy this directly into a README.md file in your repository.

---

#  🚀 Deploying Helm Charts to EKS Using Argo CD

This guide demonstrates how to create a **custom Helm chart**, push it to **Git**, and deploy it to an **Amazon EKS** cluster using **Argo CD**.

---

## 📦 Overview

We will:
1. Create a custom Helm chart with Kubernetes manifests.
2. Store it in a Git repository.
3. Deploy it on EKS using Argo CD GitOps.

---

## 📋 Prerequisites

Make sure you have the following set up:

- A running **Amazon EKS** cluster.
- Argo CD installed on the cluster.
- `kubectl`, `helm`, and `argocd` CLIs installed.
- A public or private Git repository for your Helm chart.
- (Optional) **cert-manager** installed for SSL via Let's Encrypt.

---

## 🛠️ Step 1: Create Your Helm Chart

### Create Helm Chart Structure

```bash
helm create my-app
mv my-app argocd-testing-from-helm
cd argocd-testing-from-helm
````

Replace default files with the following custom structure:

```
argocd-testing-from-helm/
├── Chart.yaml
├── charts/
├── templates/
│   ├── cluster-issuer.yaml
│   ├── deployment.yaml
│   ├── ingress.yaml
│   └── service.yaml
└── values.yaml
```

---

## 📁 File Contents

### 1. `Chart.yaml`

```yaml
apiVersion: v2
name: argocd-testing-from-helm
description: A Helm chart for deploying multiple apps with Ingress and SSL
type: application
version: 0.1.0
appVersion: "1.0"
```

---

### 2. `templates/cluster-issuer.yaml`

```yaml
{{- if .Values.clusterIssuer.enabled }}
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: {{ .Values.ingress.issuerName }}
  namespace: cert-manager
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: {{ .Values.clusterIssuer.email }}
    privateKeySecretRef:
      name: {{ .Values.clusterIssuer.secretName }}
    solvers:
      - http01:
          ingress:
            class: nginx
{{- end }}
```

---

### 3. `templates/deployment.yaml`

```yaml
{{- range .Values.apps }}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .name }}
  namespace: {{ $.Values.namespace }}
  labels:
    app: {{ .name }}
spec:
  replicas: {{ .replicas }}
  selector:
    matchLabels:
      app: {{ .name }}
  template:
    metadata:
      labels:
        app: {{ .name }}
    spec:
      containers:
        - name: {{ .name }}
          image: {{ .image }}
          imagePullPolicy: Always
          ports:
            - containerPort: 80
{{- end }}
```

---

### 4. `templates/service.yaml`

```yaml
{{- range .Values.apps }}
apiVersion: v1
kind: Service
metadata:
  name: {{ .name }}
  namespace: {{ $.Values.namespace }}
spec:
  selector:
    app: {{ .name }}
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
---
{{- end }}
```

---

### 5. `templates/ingress.yaml`

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: {{ .Values.namespace }}
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: {{ .Values.ingress.issuerName }}
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - {{ .Values.ingress.host }}
      secretName: {{ .Values.ingress.tlsSecret }}
  rules:
    - host: {{ .Values.ingress.host }}
      http:
        paths:
          {{- range .Values.apps }}
          - path: {{ printf "/%s" .name | replace "app0" "/" }}
            pathType: Prefix
            backend:
              service:
                name: {{ .name }}
                port:
                  number: 80
          {{- end }}
{{- end }}
```

---

### 6. `values.yaml` (example)

```yaml
namespace: argo-demo

clusterIssuer:
  enabled: true
  email: your-email@example.com
  secretName: letsencrypt-prod

ingress:
  enabled: true
  issuerName: letsencrypt-prod
  host: example.yourdomain.com
  tlsSecret: tls-secret

apps:
  - name: app0
    image: nginx
    replicas: 2
  - name: app1
    image: httpd
    replicas: 1
```

---

## 🧬 Step 2: Push Helm Chart to GitHub

```bash
git init
git remote add origin https://github.com/<username>/<repo>.git
git add .
git commit -m "Initial Helm chart for Argo CD"
git push -u origin main
```

---

## 🚀 Step 3: Create Argo CD Application

```bash
argocd app create argo-demo \
  --repo https://github.com/<username>/<repo>.git \
  --path . \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace argo-demo \
  --helm-chart argocd-testing-from-helm
```

---

## 🔄 Step 4: Sync the Application

```bash
argocd app sync argo-demo
```

✅ This will deploy all your defined applications, services, ingress, and TLS configuration via cert-manager.

---

## 🧪 Validation

```bash
kubectl get all -n argo-demo
kubectl get ingress -n argo-demo
```

---

## ✅ Result

You’ve successfully:

* Created a custom Helm chart
* Deployed it via GitOps using Argo CD on EKS
* Enabled SSL with cert-manager and Let's Encrypt

---



Happy Shipping 🚢

```

