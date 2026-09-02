# Todo App — AWS DevOps Pipeline

A simple full-stack Todo app (React + Node/Express + MongoDB) deployed on AWS
using Docker, Amazon ECR, Amazon EKS (Kubernetes), and CI/CD via GitHub Actions.

## Flow
```
Code (frontend + backend + .env)
        ↓
Docker build (per service)
        ↓
Push images to Amazon ECR
        ↓
Deploy to Amazon EKS (Kubernetes)
        ↓
GitHub Actions CI/CD (auto build + deploy on push to main)
        ↓
Live app via LoadBalancer
```

## Prerequisites
- AWS CLI configured (`aws configure`)
- `eksctl`, `kubectl`, `docker` installed
- An AWS account with permissions for ECR, EKS, IAM

## 1. Local development
```bash
# Backend
cd backend
cp .env.example .env
npm install
npm start          # runs on :5000
```

The frontend is a **single static file** (`frontend/index.html`) — no npm, no
build step. Just double-click it to open in your browser, or run:
```bash
cd frontend
python3 -m http.server 3000   # then open http://localhost:3000
```
At the top of the page there's an "API URL" box (defaults to
`http://localhost:5000`) — that's where it points its requests. Change it if
your backend runs somewhere else, then click **Connect**.

## 2. Create ECR repositories
```bash
aws ecr create-repository --repository-name todo-backend
aws ecr create-repository --repository-name todo-frontend
```

## 3. Build & push images manually (first time)
```bash
ACCOUNT_ID=<your-aws-account-id>
REGION=us-east-1

aws ecr get-login-password --region $REGION | \
  docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com

docker build -t todo-backend ./backend
docker tag todo-backend:latest $ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/todo-backend:latest
docker push $ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/todo-backend:latest

docker build -t todo-frontend ./frontend
docker tag todo-frontend:latest $ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/todo-frontend:latest
docker push $ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/todo-frontend:latest
```

## 4. Create the EKS cluster
```bash
eksctl create cluster --name todo-cluster --region us-east-1 --nodes 2 --node-type t3.medium
```

## 5. Create the backend secret (from .env — never commit .env itself)
```bash
kubectl create secret generic backend-secrets --from-env-file=backend/.env
```

## 6. Update image placeholders
In `k8s/backend.yaml` and `k8s/frontend.yaml`, replace:
```
<ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/...
```
with your real ECR URLs.

## 7. Deploy
```bash
kubectl apply -f k8s/mongo.yaml
kubectl apply -f k8s/backend.yaml
kubectl apply -f k8s/frontend.yaml
```

## 8. Get the public URL
```bash
kubectl get svc frontend-service
```
Use the `EXTERNAL-IP` (AWS LoadBalancer DNS) shown — that's your live app.

## 9. Enable CI/CD (automatic deploys)
Add these secrets in your GitHub repo settings (Settings → Secrets → Actions):
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`

Every push to `main` will now: build both Docker images → push to ECR → deploy to EKS automatically (see `.github/workflows/deploy.yml`).

## Cleanup (avoid AWS charges)
```bash
eksctl delete cluster --name todo-cluster --region us-east-1
aws ecr delete-repository --repository-name todo-backend --force
aws ecr delete-repository --repository-name todo-frontend --force
```
