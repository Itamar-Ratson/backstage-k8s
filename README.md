# Deploying Backstage on Kubernetes with Minikube

This guide walks you through building a Backstage Docker image using a multi-stage Dockerfile and deploying it to Kubernetes (minikube). It incorporates lessons learned from common issues and their solutions directly into each step.

**Official Documentation:**

- [Backstage Docker Deployment](https://backstage.io/docs/deployment/docker/)
- [Backstage Kubernetes Deployment](https://backstage.io/docs/deployment/k8s)
- [Backstage Authentication](https://backstage.io/docs/auth/)

## Prerequisites

### Node.js Version

Backstage requires Node.js 20 or 22. Using an older or incompatible version will cause native module compilation failures, particularly with `isolated-vm` which depends on V8 APIs that change between Node versions.

```bash
# Install nvm if not already installed
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
source ~/.bashrc

# Install and use Node 22 (recommended - matches the Dockerfile runtime)
nvm install 22
nvm use 22

# Verify the installation
node -v  # Should show v22.x.x
```

### Build Dependencies

Install build tools required for compiling native Node.js modules:

```bash
sudo apt-get update
sudo apt-get install -y python3 g++ build-essential
```

### gitignore

Create a .gitignore to exclude build artifacts, dependencies, and local configuration files from version control:

```bash
cat > .gitignore << 'EOF'
# Dependencies
node_modules/

# Build outputs
dist/
dist-types/
packages/*/dist/
plugins/*/dist/

# Yarn
.yarn/*
!.yarn/patches
!.yarn/plugins
!.yarn/releases
!.yarn/sdks
!.yarn/versions
.pnp.*

# Local configuration (contains secrets)
*.local.yaml

# IDE
.idea/
.vscode/
*.swp
*.swo

# OS
.DS_Store
Thumbs.db

# Logs
*.log
npm-debug.log*
yarn-debug.log*
yarn-error.log*

# Test coverage
coverage/

# Temporary files
*.tmp
*.temp
EOF
```

## Step 2: Configure the Multi-Stage Dockerfile

The multi-stage Dockerfile performs the entire build inside Docker, eliminating host environment dependencies. Replace the default `Dockerfile` in your Backstage app directory with this version:

```dockerfile
# Stage 1 - Create yarn install skeleton layer
FROM node:22-bookworm-slim AS packages
WORKDIR /app
COPY backstage.json package.json yarn.lock ./
COPY .yarn ./.yarn
COPY .yarnrc.yml ./
COPY packages packages
RUN find packages \! -name "package.json" -mindepth 2 -maxdepth 2 -exec rm -rf {} \+

# Stage 2 - Install dependencies and build
FROM node:22-bookworm-slim AS build
ENV PYTHON=/usr/bin/python3
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && \
    apt-get install -y --no-install-recommends python3 g++ build-essential libsqlite3-dev
USER node
WORKDIR /app
COPY --from=packages --chown=node:node /app .
RUN --mount=type=cache,target=/home/node/.cache/yarn,sharing=locked,uid=1000,gid=1000 \
    yarn install --immutable
COPY --chown=node:node . .
RUN yarn tsc
RUN yarn --cwd packages/backend build
RUN mkdir packages/backend/dist/skeleton packages/backend/dist/bundle \
    && tar xzf packages/backend/dist/skeleton.tar.gz -C packages/backend/dist/skeleton \
    && tar xzf packages/backend/dist/bundle.tar.gz -C packages/backend/dist/bundle

# Stage 3 - Final image
FROM node:22-bookworm-slim
ENV PYTHON=/usr/bin/python3
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && \
    apt-get install -y --no-install-recommends python3 g++ build-essential libsqlite3-dev
USER node
WORKDIR /app
COPY --from=build --chown=node:node /app/.yarn ./.yarn
COPY --from=build --chown=node:node /app/.yarnrc.yml  ./
COPY --from=build --chown=node:node /app/backstage.json ./
COPY --from=build --chown=node:node /app/yarn.lock /app/package.json /app/packages/backend/dist/skeleton/ ./
RUN --mount=type=cache,target=/home/node/.cache/yarn,sharing=locked,uid=1000,gid=1000 \
    yarn workspaces focus --all --production && rm -rf "$(yarn cache clean)"
COPY --from=build --chown=node:node /app/packages/backend/dist/bundle/ ./
COPY --chown=node:node app-config*.yaml ./
COPY --chown=node:node examples ./examples
ENV NODE_ENV=production
ENV NODE_OPTIONS="--no-node-snapshot"
CMD ["node", "packages/backend", "--config", "app-config.yaml", "--config", "app-config.production.yaml"]
```

## Step 3: Configure .dockerignore for Multi-Stage Builds

The default `.dockerignore` from `create-app` is designed for host builds and excludes source files with `packages/*/src`. This causes the multi-stage build to fail at the `yarn tsc` step with "No inputs were found in config file" because TypeScript has no source files to compile.

Replace the `.dockerignore` with the correct version for multi-stage builds:

```bash
cat > .dockerignore << 'EOF'
dist-types
node_modules
packages/*/dist
packages/*/node_modules
plugins/*/dist
plugins/*/node_modules
*.local.yaml
EOF
```

## Step 4: Configure Production Settings

The default `app-config.production.yaml` has two issues that must be fixed before deployment. First, it references database connection environment variables. Second, the guest authentication provider is disabled by default in containerized environments, which will cause 401 Unauthorized errors.

Replace `app-config.production.yaml` with this configuration:

```yaml
app:
  baseUrl: http://localhost:7007

backend:
  baseUrl: http://localhost:7007
  listen: ':7007'
  database:
    client: pg
    connection:
      host: ${POSTGRES_HOST}
      port: ${POSTGRES_PORT}
      user: ${POSTGRES_USER}
      password: ${POSTGRES_PASSWORD}

auth:
  providers:
    guest:
      dangerouslyAllowOutsideDevelopment: true  # Required for containerized environments

catalog:
  locations:
    - type: file
      target: ./examples/entities.yaml
    - type: file
      target: ./examples/template/template.yaml
      rules:
        - allow: [Template]
    - type: file
      target: ./examples/org.yaml
      rules:
        - allow: [User, Group]
```

The `dangerouslyAllowOutsideDevelopment: true` flag explicitly enables guest authentication outside of local development. For production deployments, you should configure a proper auth provider (GitHub, Google, Okta, etc.) instead.

## Step 5: Build the Docker Image

Build the image with a version tag. Using semantic versioning makes it easier to track changes and avoid caching issues:

```bash
docker build -t backstage:1.0.0 .
```

## Step 6: Set Up Kubernetes Resources

### Create Namespace

```bash
kubectl create namespace backstage
```

### Create Kubernetes Secrets

The secret must include all four environment variables that `app-config.production.yaml` references. Missing `POSTGRES_HOST` or `POSTGRES_PORT` will cause Backstage to fail with "Port should be >= 0 and < 65536. Received type number (NaN)" because the missing values parse as NaN.

```bash
kubectl create secret generic postgres-secrets -n backstage \
  --from-literal=POSTGRES_HOST=postgres \
  --from-literal=POSTGRES_PORT=5432 \
  --from-literal=POSTGRES_USER=backstage \
  --from-literal=POSTGRES_PASSWORD=hunter2
```

### Deploy PostgreSQL

Create `postgres-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: backstage
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:15
          env:
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: postgres-secrets
                  key: POSTGRES_USER
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secrets
                  key: POSTGRES_PASSWORD
          ports:
            - containerPort: 5432
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: backstage
spec:
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
```

Apply the PostgreSQL deployment:

```bash
kubectl apply -f postgres-deployment.yaml
```

Wait for PostgreSQL to be ready:

```bash
kubectl get pods -n backstage -w
# Wait until postgres pod shows 1/1 Running
```

### Deploy Backstage

Create `backstage-deployment.yaml`. Note the `imagePullPolicy: Never` setting, which tells Kubernetes to use the locally loaded image rather than trying to pull from a registry:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backstage
  namespace: backstage
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backstage
  template:
    metadata:
      labels:
        app: backstage
    spec:
      containers:
        - name: backstage
          image: backstage:1.0.0
          imagePullPolicy: Never  # Use locally loaded image
          ports:
            - containerPort: 7007
              name: http
          envFrom:
            - secretRef:
                name: postgres-secrets
---
apiVersion: v1
kind: Service
metadata:
  name: backstage
  namespace: backstage
spec:
  selector:
    app: backstage
  ports:
    - name: http
      port: 80
      targetPort: http
```

## Step 7: Load Image into Minikube and Deploy

Minikube runs in an isolated environment with its own Docker daemon. Your locally built image exists only in your host's Docker daemon, so you must explicitly load it into minikube:

```bash
minikube image load backstage:1.0.0
```

Verify the image was loaded:

```bash
minikube image ls | grep backstage
```

Now deploy Backstage:

```bash
kubectl apply -f backstage-deployment.yaml
```

Watch the pod status until it shows `1/1 Running`:

```bash
kubectl get pods -n backstage -w
```

Check the logs to ensure Backstage started successfully:

```bash
kubectl logs -n backstage -l app=backstage
```

You should see messages about plugins initializing without errors.

## Step 8: Access Backstage

Forward the service port to your local machine. Use port 8080 to avoid needing sudo:

```bash
kubectl port-forward --namespace=backstage svc/backstage 8080:80
```

Open <http://localhost:8080> in your browser. You should see the Backstage UI and be able to log in as a guest.

## Updating Backstage

When you make changes and need to redeploy, always use a new image tag. Kubernetes caches images by tag, so using the same tag causes it to reuse the cached version even after you've loaded a new image.

```bash
# Make your changes to the source code or configuration

# Build with a new tag
docker build -t backstage:1.0.1 .

# Load the new image into minikube
minikube image load backstage:1.0.1

# Update the deployment to use the new image
kubectl set image deployment/backstage -n backstage backstage=backstage:1.0.1
```

## Useful Commands

```bash
# View pod logs
kubectl logs -n backstage -l app=backstage

# View pod logs with timestamps
kubectl logs -n backstage deploy/backstage --timestamps

# Execute commands in the container (useful for debugging config)
kubectl exec -n backstage deploy/backstage -- cat /app/app-config.production.yaml

# Restart deployment
kubectl rollout restart deployment backstage -n backstage

# Delete and recreate pod
kubectl delete pod -n backstage -l app=backstage

# Check minikube images
minikube image ls | grep backstage

# Remove image from minikube
minikube ssh "docker rmi -f backstage:1.0.0"

# View all resources in the namespace
kubectl get all -n backstage
```

## Next Steps

Once Backstage is running, you may want to:

1. **Configure a production auth provider** - Replace guest auth with GitHub, Google, Okta, or another provider. See the [Authentication documentation](https://backstage.io/docs/auth/).

2. **Add catalog entities** - Populate the software catalog with your services, APIs, and documentation.

3. **Configure the Kubernetes plugin** - Enable viewing Kubernetes resources from within Backstage.

4. **Set up TechDocs** - Enable documentation generation and viewing.

5. **Deploy to a production cluster** - Move beyond minikube to EKS, GKE, or another managed Kubernetes service.
