# E-Commerce Application CI/CD Pipeline


# Plan summary (what you'll produce)

1. Repo layout: `api/` (Node/Express), `webapp/` (React).
2. Local dev + unit tests for both projects.
3. GitHub Actions workflows that:

   * run tests & build for both projects,
   * build Docker images,
   * push images to cloud registry (example uses AWS ECR),
   * deploy to cloud (example uses AWS ECS Fargate),
   * cache dependencies & Docker layers to speed builds,
   * deploy automatically from `main`.
4. `README.md` documenting everything.

---

# Task-by-task step-by-step

## Task 1 — Project setup (repo + structure)

1. Create repo and basic layout:

```bash
mkdir ecommerce-platform
cd ecommerce-platform
git init
mkdir api webapp
mkdir -p .github/workflows
```

2. Create `README.md`, `.gitignore` (node, build artifacts, `.env`), and initial commit:

```bash
cat > .gitignore <<'EOF'
node_modules
dist
/build
.env
.DS_Store
EOF

git add .
git commit -m "initial project structure"
```

---

## Task 2 — Initialize GitHub Actions folder

(You already created `.github/workflows`.) We'll add workflows later (CI + CD). For now keep folder ready.

---

## Task 3 — Backend API setup (Node.js + Express + tests)

Create a minimal Node/Express app inside `api/`:

1. Initialize:

```bash
cd api
npm init -y
npm install express
npm install --save-dev jest supertest nodemon
```

2. Minimal app: `api/src/index.js`

```js
const express = require('express');
const app = express();
app.use(express.json());

let products = [{ id: 1, name: "T-shirt", price: 20 }];

app.get('/health', (req, res) => res.json({ ok: true }));
app.get('/products', (req, res) => res.json(products));
app.post('/orders', (req, res) => {
  const { items, user } = req.body;
  // very basic order emulation
  return res.status(201).json({ id: Date.now(), items, user });
});

const port = process.env.PORT || 3000;
if (require.main === module) {
  app.listen(port, () => console.log(`API running on ${port}`));
}
module.exports = app;
```

3. Add scripts in `api/package.json`:

```json
"scripts": {
  "start": "node src/index.js",
  "dev": "nodemon src/index.js",
  "test": "jest --runInBand --detectOpenHandles",
  "build": "echo 'no build for node app' && exit 0"
}
```

4. Unit test example `api/test/api.test.js`:

```js
const request = require('supertest');
const app = require('../src/index');

describe('API', () => {
  test('GET /health', async () => {
    const res = await request(app).get('/health');
    expect(res.statusCode).toBe(200);
    expect(res.body.ok).toBe(true);
  });
  test('GET /products', async () => {
    const res = await request(app).get('/products');
    expect(res.statusCode).toBe(200);
    expect(Array.isArray(res.body)).toBe(true);
  });
});
```

5. Commit changes:

```bash
git add api
git commit -m "add simple express api + tests"
```

---

## Task 4 — Frontend webapp setup (React + tests)

Create a React app in `webapp/`. Use Vite or CRA (I'll show Vite quick setup):

1. Initialize Vite React app:

```bash
cd ../
npm create vite@latest webapp -- --template react
cd webapp
npm install
npm install --save-dev jest @testing-library/react @testing-library/jest-dom
```

2. Add API integration example (fetch products) — `webapp/src/App.jsx`:

```jsx
import React, { useEffect, useState } from 'react';

export default function App() {
  const [products, setProducts] = useState([]);
  useEffect(() => {
    fetch(import.meta.env.VITE_API_URL + '/products')
      .then(r => r.json())
      .then(setProducts)
      .catch(console.error);
  }, []);
  return (
    <div>
      <h1>Products</h1>
      <ul>{products.map(p => <li key={p.id}>{p.name} — ${p.price}</li>)}</ul>
    </div>
  );
}
```

3. `webapp/package.json` scripts (example):

```json
"scripts": {
  "dev": "vite",
  "build": "vite build",
  "preview": "vite preview --port 5173",
  "test": "jest"
}
```

4. Add basic tests (React Testing Library) — `webapp/src/App.test.jsx`.

5. Commit:

```bash
git add webapp
git commit -m "add webapp skeleton"
```

---

## Task 5 — Continuous Integration workflow (tests & builds)

We'll create workflows that:

* run tests,
* build,
* run lint (optional).

You can create one workflow that runs jobs in parallel (api & webapp) or two separate workflows. Example combined CI workflow: `.github/workflows/ci.yml`

```yaml
name: CI

on:
  push:
    branches: [ '**' ]
  pull_request:
    branches: [ '**' ]

jobs:
  api:
    name: Backend CI
    runs-on: ubuntu-latest
    defaults: { run: { working-directory: api } }
    steps:
      - uses: actions/checkout@v4
      - name: Use Node.js 18
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
      - name: Install dependencies
        run: npm ci
      - name: Run tests
        run: npm test

  webapp:
    name: Frontend CI
    runs-on: ubuntu-latest
    defaults: { run: { working-directory: webapp } }
    steps:
      - uses: actions/checkout@v4
      - name: Use Node.js 18
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
      - name: Install dependencies
        run: npm ci
      - name: Run tests
        run: npm test
      - name: Build
        run: npm run build
```

Notes:

* `actions/setup-node` with `cache: 'npm'` caches `~/.npm`.
* Use `matrix` if you want multiple Node versions.

Commit workflow:

```bash
git add .github/workflows/ci.yml
git commit -m "add CI workflow"
```

---

## Task 6 — Docker integration (Dockerfiles for both services)

### API Dockerfile (`api/Dockerfile`)

```dockerfile
# Use small Node image
FROM node:18-alpine AS base
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm ci --production
COPY . .
EXPOSE 3000
CMD ["node", "src/index.js"]
```

If you want smaller builds with multi-stage and dev deps for build/test, adjust accordingly.

### Webapp Dockerfile (`webapp/Dockerfile`)

```dockerfile
# build stage
FROM node:18-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# serve stage
FROM nginx:stable-alpine AS prod
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

Commit Dockerfiles:

```bash
git add api/Dockerfile webapp/Dockerfile
git commit -m "add dockerfiles for api and webapp"
```

---

## Task 7 — Deploy to Cloud (example: AWS ECR + ECS Fargate)

I pick AWS in this guide. If you prefer Azure (ACR + Azure App Service / AKS) or GCP (GCR/Artifact Registry + Cloud Run/GKE) I can provide those workflows too.

### Required AWS setup (one-time)

1. Create ECR repositories (one for `api`, one for `webapp`) — you can create via AWS Console or CLI:

```bash
aws ecr create-repository --repository-name ecommerce-api
aws ecr create-repository --repository-name ecommerce-webapp
```

2. Create an ECS cluster and Fargate services (or use a task definition that you update from Actions).
3. Create an IAM user with permissions to push to ECR and update ECS (or use OIDC GitHub Actions role — more secure).

### GitHub Secrets (in repo Settings → Secrets → Actions)

Set the following secrets:

* `AWS_ACCESS_KEY_ID`
* `AWS_SECRET_ACCESS_KEY`
* `AWS_REGION` (e.g., `us-east-1`)
* `ECR_REPOSITORY_API` (e.g., `123456789012.dkr.ecr.us-east-1.amazonaws.com/ecommerce-api`)
* `ECR_REPOSITORY_WEBAPP`
* `ECS_CLUSTER` — your ECS cluster name
* `ECS_SERVICE_API` — ECS service name for API
* `ECS_SERVICE_WEBAPP` — ECS service name for front-end (if using ECS)
* (Optional) `DOCKER_BUILDKIT` or other options

### Workflow to build Docker images + push to ECR + deploy (example `.github/workflows/deploy.yml`)

This will run on pushes to `main` and deploy.

```yaml
name: CD (build & deploy)

on:
  push:
    branches: [ main ]

env:
  AWS_REGION: ${{ secrets.AWS_REGION }}

jobs:
  login-and-build:
    name: Build, push images and deploy
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v2

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Login to Amazon ECR
        id: ecr-login
        uses: aws-actions/amazon-ecr-login@v1

      # Build and push API image
      - name: Build and push API image
        uses: docker/build-push-action@v4
        with:
          context: ./api
          file: ./api/Dockerfile
          push: true
          tags: |
            ${{ secrets.ECR_REPOSITORY_API }}:latest
            ${{ secrets.ECR_REPOSITORY_API }}:${{ github.sha }}
          cache-from: type=registry,ref=${{ secrets.ECR_REPOSITORY_API }}:cache
          cache-to: type=registry,ref=${{ secrets.ECR_REPOSITORY_API }}:cache,mode=max

      # Build and push Webapp image
      - name: Build and push Webapp image
        uses: docker/build-push-action@v4
        with:
          context: ./webapp
          file: ./webapp/Dockerfile
          push: true
          tags: |
            ${{ secrets.ECR_REPOSITORY_WEBAPP }}:latest
            ${{ secrets.ECR_REPOSITORY_WEBAPP }}:${{ github.sha }}
          cache-from: type=registry,ref=${{ secrets.ECR_REPOSITORY_WEBAPP }}:cache
          cache-to: type=registry,ref=${{ secrets.ECR_REPOSITORY_WEBAPP }}:cache,mode=max

      # Deploy to ECS (update task definition)
      - name: Deploy API to ECS
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ./deploy/api-taskdef.json
          service: ${{ secrets.ECS_SERVICE_API }}
          cluster: ${{ secrets.ECS_CLUSTER }}
          wait-for-service-stability: true

      - name: Deploy Webapp to ECS
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ./deploy/webapp-taskdef.json
          service: ${{ secrets.ECS_SERVICE_WEBAPP }}
          cluster: ${{ secrets.ECS_CLUSTER }}
          wait-for-service-stability: true
```

Notes:

* `deploy/*.json` are ECS task definition templates. One approach: have JSON with placeholders, and update the container image URI using `jq` or `sed` in the workflow before calling the deploy action.
* `docker/build-push-action` uses Buildx and supports caching to registry (speeds builds).

If you prefer AWS Elastic Beanstalk, Cloud Run, Azure WebApp, or Kubernetes, replace the deploy steps accordingly.

---

## Task 8 — Continuous Deployment (auto deploy from `main`)

The `deploy.yml` above triggers on push to `main`. To have deployments on merges to `main`, ensure:

* Protect `main` with branch protection and require PRs for changes.
* Optionally restrict deployments to tag creation or to merges that pass CI.

You can also:

* Add `if: github.event_name == 'push' && github.ref == 'refs/heads/main'` to be explicit.
* Add approvals step (manual step) using `workflow_run` or environment protection rules.

---

## Task 9 — Performance and security

### Build caching

1. Node modules: use `actions/setup-node` with `cache: 'npm'` (already in CI).
2. Docker layer caching: Use `docker/build-push-action` with `cache-from` and `cache-to` registry cache (example above).
3. Speed up tests: run unit tests in parallel if your test runner supports it.

### Secrets & sensitive data

* Use GitHub Secrets to store cloud credentials (we named them above).
* Never commit `.env` to repo. For local dev use `.env.example`.
* If you need DB credentials, store in Secrets and populate environment variables in ECS task definition or use AWS Secrets Manager + inject into ECS.

### Security best-practices

* Use least-privilege IAM role for GitHub Actions (prefer OIDC federation instead of long-lived AWS keys).
* Scan your images for vulnerabilities (e.g., use `anchore/scan-action` or ECR image scan).
* Use Dependabot or similar for dependency updates.
* Enforce branch protection, code review, and signed commits if desired.
* Rotate credentials.

---

## Task 10 — Project documentation (`README.md`)

Create a clear README template in root:

```md
# ecommerce-platform

## Overview
Monorepo: `api/` (Node/Express) and `webapp/` (React). CI/CD via GitHub Actions. Docker images pushed to AWS ECR and deployed to ECS Fargate.

## Local development

### Backend
```

cd api
npm install
npm run dev

```

### Frontend
```

cd webapp
npm install
VITE_API_URL=[http://localhost:3000](http://localhost:3000) npm run dev

```

## Tests
```

cd api && npm test
cd webapp && npm test

```

## Deployment
- CI runs on PRs. Push to `main` triggers deployment workflow that builds & deploys images to AWS ECR/ECS.
- Required GitHub Secrets:
  - `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`
  - `ECR_REPOSITORY_API`, `ECR_REPOSITORY_WEBAPP`
  - `ECS_CLUSTER`, `ECS_SERVICE_API`, `ECS_SERVICE_WEBAPP`
```

Also add `CONTRIBUTING.md` and `deploy/` templates.

---

# Extra suggestions & improvements (next-level)

* Use **infrastructure-as-code**: Terraform or CloudFormation (store TF in `/infra`) to create ECR, ECS, ALB, task definitions — makes deployment reproducible.
* Use **GitHub OIDC** for AWS: avoid storing long-lived AWS keys; configure trust between GitHub Actions and AWS IAM role.
* Add **staging** environment: deploy `main` to staging and `production` only on `release` tag or manual approval.
* Add **health checks** and **smoke tests** after deployment.
* Add **observability**: CloudWatch / Datadog metrics, error reporting.
* Add **canary** deployments or blue/green using ECS or Kubernetes for safer releases.
* Add **Secrets Manager** or Parameter Store for DB credentials and rotate them.
* Add **PR previews** for frontend (deploy per-PR preview to ephemeral environment) using dynamic subdomains.

---

# Example file references included above

* `api/Dockerfile`, `webapp/Dockerfile`
* `.github/workflows/ci.yml`
* `.github/workflows/deploy.yml`
* `README.md` template
* `deploy/api-taskdef.json` & `deploy/webapp-taskdef.json` (task definition templates — you’ll need to fill container definitions and set container image URIs at deploy time)

---
