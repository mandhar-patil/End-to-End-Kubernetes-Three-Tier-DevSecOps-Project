# Frontend — Kubernetes Deployment & Backend Connectivity Guide

## Overview

This is a React-based frontend that connects to a Node.js backend API. It runs as a containerized workload on EKS and communicates with the backend via an AWS Application Load Balancer (ALB).

---

## Architecture

```
Browser
   │
   │  http://<ALB-URL>/          → Frontend (React) :3000
   │  http://<ALB-URL>/api/tasks → Backend (Node.js) :3500
   ▼
AWS ALB (Ingress)
   ├──  /        →  frontend-service:3000
   ├──  /api     →  api-service:3500
   └──  /healthz →  api-service:3500

Frontend Pod  ──HTTP──▶  Backend Pod  ──TCP──▶  MongoDB Pod
(React/Nginx)            (Node.js)               (mongo:4.4.6)
                         :3500                    :27017
```

> **Important:** React runs in the browser — it cannot talk to MongoDB directly.
> All database operations go through the backend API.

---

## Prerequisites

Before deploying the frontend, make sure these are ready:

| # | Requirement | Check Command |
|---|---|---|
| 1 | EKS cluster running | `kubectl get nodes` |
| 2 | `three-tier` namespace exists | `kubectl get ns three-tier` |
| 3 | Backend (API) service running | `kubectl get svc api -n three-tier` |
| 4 | MongoDB service running | `kubectl get svc mongodb-svc -n three-tier` |
| 5 | AWS Load Balancer Controller installed | `kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller` |
| 6 | ECR repository exists | `aws ecr describe-repositories --region us-east-2` |
| 7 | ALB Ingress deployed and ADDRESS populated | `kubectl get ingress -n three-tier` |

---

## Step 1 — Get Your ALB URL

```bash
kubectl get ingress -n three-tier

# Output:
# NAME     CLASS   HOSTS   ADDRESS                                                    PORTS
# mainlb   alb     *       k8s-threetie-mainlb-xxxx.us-east-2.elb.amazonaws.com      80
```

Copy the ADDRESS — you will need it in Step 2.

---

## Step 2 — Set the Backend URL in Deployment

The React app reads `REACT_APP_BACKEND_URL` at **build time**.
You must set this env var to your ALB URL before building the Docker image.

Update `Kubernetes-Manifests-file/Frontend/deployment.yaml`:

```yaml
env:
  - name: REACT_APP_BACKEND_URL
    value: "http://<YOUR-ALB-ADDRESS>/api/tasks"
    # Example:
    # value: "http://k8s-threetie-mainlb-xxxx.us-east-2.elb.amazonaws.com/api/tasks"
```

> **Why?** React bakes environment variables into the JavaScript bundle at build time.
> If this URL is wrong, the ADD TASK button will silently fail.

---

## Step 3 — Dockerfile

The frontend Dockerfile accepts `REACT_APP_BACKEND_URL` as a **build argument** so it can be passed at build time without hardcoding it into the image.

```dockerfile
FROM node:14
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm install

# Accept build arg and set as env var
# This gets baked into the React bundle at npm run build
ARG REACT_APP_BACKEND_URL
ENV REACT_APP_BACKEND_URL=$REACT_APP_BACKEND_URL

COPY . .
RUN npm run build
EXPOSE 3000
CMD ["npm", "start"]
```

> **How ARG vs ENV works:**
> - `ARG` — available only during Docker build
> - `ENV` — persists into the running container
> - React reads `ENV` at `npm run build` and bakes it into the JS bundle

---

## Step 4 — Build and Push Docker Image

```bash
# Navigate to frontend source
cd Application-Code/frontend

# Build image — pass ALB URL as build argument
docker build \
  --build-arg REACT_APP_BACKEND_URL=http://<YOUR-ALB-ADDRESS>/api/tasks \
  -t three-tier-frontend:latest .

# Login to ECR
aws ecr get-login-password --region us-east-2 | \
  docker login --username AWS --password-stdin \
  986485678858.dkr.ecr.us-east-2.amazonaws.com

# Tag and push
docker tag three-tier-frontend:latest \
  986485678858.dkr.ecr.us-east-2.amazonaws.com/three-tier-frontend:latest

docker push 986485678858.dkr.ecr.us-east-2.amazonaws.com/three-tier-frontend:latest
```

---

## Step 5 — Deploy to Kubernetes

```bash
# Apply frontend deployment and service
kubectl apply -f Kubernetes-Manifests-file/Frontend/deployment.yaml

# Verify pod is running
kubectl get pods -n three-tier -l role=frontend

# Check logs if pod fails
kubectl logs -n three-tier -l role=frontend
```

---

## Step 6 — Apply Ingress

```bash
kubectl apply -f Kubernetes-Manifests-file/ingress.yaml

# Wait for ALB ADDRESS to appear (2-3 minutes)
kubectl get ingress -n three-tier -w
```

---

## Step 7 — Verify End-to-End Connectivity

```bash
# 1. Frontend loads
curl -I http://<ALB-ADDRESS>/

# 2. Backend API responds
curl http://<ALB-ADDRESS>/api/tasks

# 3. Health checks pass
curl http://<ALB-ADDRESS>/healthz
curl http://<ALB-ADDRESS>/ready
curl http://<ALB-ADDRESS>/started
```

Expected responses:
```
/ → HTTP 200 (React app HTML)
/api/tasks → [] or list of tasks (JSON)
/healthz → 200 OK
/ready → 200 OK
/started → 200 OK
```

---

## Frontend Service Definition

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: three-tier
spec:
  ports:
  - port: 3000
    protocol: TCP
  type: ClusterIP
  selector:
    role: frontend
```

> Service type is `ClusterIP` — the frontend is NOT exposed directly.
> All external traffic comes through the ALB Ingress.

---

## Frontend Deployment Definition

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: three-tier
spec:
  replicas: 1
  selector:
    matchLabels:
      role: frontend
  template:
    metadata:
      labels:
        role: frontend
    spec:
      containers:
      - name: frontend
        image: 986485678858.dkr.ecr.us-east-2.amazonaws.com/three-tier-frontend:<BUILD_NUMBER>
        imagePullPolicy: Always
        env:
          - name: REACT_APP_BACKEND_URL
            value: "http://<ALB-ADDRESS>/api/tasks"
        ports:
        - containerPort: 3000
```

---

## Ingress Definition

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mainlb
  namespace: three-tier
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]'
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 3500
          - path: /healthz
            pathType: Exact
            backend:
              service:
                name: api
                port:
                  number: 3500
          - path: /ready
            pathType: Exact
            backend:
              service:
                name: api
                port:
                  number: 3500
          - path: /started
            pathType: Exact
            backend:
              service:
                name: api
                port:
                  number: 3500
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 3000
```

---

## CI/CD Pipeline (Jenkins)

The Jenkins pipeline automatically:
1. Builds Docker image from `Application-Code/frontend`
2. Passes `REACT_APP_BACKEND_URL` as `--build-arg`
3. Pushes to ECR with `BUILD_NUMBER` as tag
4. Updates `deployment.yaml` image tag in GitHub
5. ArgoCD detects the change and redeploys automatically

```
Code Push → Jenkins Build → ECR Push → GitHub Update → ArgoCD Sync → Pod Redeploy
```

### Jenkins Pipeline — Docker Build Stage

```groovy
stage('Docker Image Build') {
    steps {
        dir('Application-Code/frontend') {
            sh 'docker system prune -f'
            sh 'docker container prune -f'
            sh '''
                docker build \
                  --build-arg REACT_APP_BACKEND_URL=${ALB_URL}/api/tasks \
                  -t ${AWS_ECR_REPO_NAME}:latest .
            '''
        }
    }
}
```

Store `ALB_URL` as a Jenkins credential:

```
Jenkins → Manage Jenkins → Credentials → Add
Kind    : Secret text
ID      : ALB_URL
Value   : http://k8s-threetie-mainlb-xxxx.us-east-2.elb.amazonaws.com
```

Then reference in pipeline environment block:

```groovy
environment {
    ALB_URL = credentials('ALB_URL')
}
```

---

## Troubleshooting

### ADD TASK button does nothing
```bash
# Open browser DevTools → Network tab
# Look for failed requests to /api/tasks
# Check REACT_APP_BACKEND_URL is set to correct ALB URL
kubectl describe deployment frontend -n three-tier | grep REACT_APP
```

### Frontend pod not starting
```bash
kubectl describe pod -n three-tier -l role=frontend
kubectl logs -n three-tier -l role=frontend
# Common cause: ECR image pull failure — check IAM role on nodes
```

### ALB ADDRESS is empty
```bash
# Check controller logs
kubectl logs -n kube-system \
  -l app.kubernetes.io/name=aws-load-balancer-controller --tail=30

# Check subnets are tagged
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=<VPC_ID>" \
  --query "Subnets[*].{ID:SubnetId,Tags:Tags}"
```

### Tasks not saving (500 error from API)
```bash
# Backend cannot reach MongoDB
kubectl logs -n three-tier -l role=api | grep -i "mongo\|error"

# Verify MongoDB service name matches connection string
kubectl get svc -n three-tier
# Must show: mongodb-svc (not mongodb)
```

---

## Environment Variables Reference

| Variable | Description | Example |
|---|---|---|
| `REACT_APP_BACKEND_URL` | Full URL to backend API tasks endpoint | `http://<ALB-URL>/api/tasks` |

> **Note:** All `REACT_APP_*` variables are baked into the build at `npm run build`.
> Changing them requires passing a new `--build-arg` and rebuilding the Docker image.
> The Dockerfile uses `ARG` + `ENV` to accept this value at build time.
