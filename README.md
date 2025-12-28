# Deploying Backstage on Kubernetes with Minikube

This guide walks you through building a Backstage Docker image using a multi-stage Dockerfile and deploying it to Kubernetes (minikube). It incorporates lessons learned from common issues and their solutions directly into each step.

**Official Documentation:**

- [Backstage Docker Deployment](https://backstage.io/docs/deployment/docker/)
- [Backstage Kubernetes Deployment](https://backstage.io/docs/deployment/k8s)
- [Backstage Authentication](https://backstage.io/docs/auth/)

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

## Step 1: Create the Backstage App

```bash
npx @backstage/create-app@latest
# Enter a name when prompted, e.g., "backstage"

cd backstage
```

This scaffolds a complete Backstage application with all necessary files.

## Step 2: Configure Repository Files

Several files need to be created or modified from the defaults. The complete files are provided in this repository:

| File | Purpose |
|------|---------|
| `Dockerfile` | Multi-stage build that compiles everything inside Docker |
| `.dockerignore` | Must NOT exclude source files (unlike the host build version) |
| `.gitignore` | Standard Node.js/Yarn exclusions |
| `app-config.production.yaml` | Production overrides with database connection and guest auth enabled |

Key configuration notes:

The `.dockerignore` must use the multi-stage version that does NOT exclude `packages/*/src`. The default from `create-app` excludes source files, which causes `yarn tsc` to fail with "No inputs were found".

The `app-config.production.yaml` must include `dangerouslyAllowOutsideDevelopment: true` under the guest auth provider. Without this, you will get 401 Unauthorized errors because guest auth is disabled by default in containerized environments.

## Step 3: Build the Docker Image

Build the image with a version tag. Using semantic versioning makes it easier to track changes and avoid caching issues:

```bash
docker build -t backstage:1.0.0 -f Dockerfile backstage/
```

## Step 4: Set Up Kubernetes Resources

### Create Namespace

```bash
kubectl create namespace backstage
```

### Create Kubernetes Secrets

The secret must include all four environment variables that `app-config.production.yaml` references. Missing `POSTGRES_HOST` or `POSTGRES_PORT` will cause Backstage to fail with "Port should be >= 0 and < 65536. Received type number (NaN)" because the missing values parse as NaN.

```bash
kubectl apply -f kubernetes/postgres-secrets.yaml
```

### Deploy PostgreSQL

Apply the PostgreSQL deployment from the kubernetes directory:

```bash
kubectl apply -f kubernetes/postgres-deployment.yaml
```

Wait for PostgreSQL to be ready:

```bash
kubectl get pods -n backstage -w
# Wait until postgres pod shows 1/1 Running
```

### Deploy Backstage

The `backstage-deployment.yaml` includes `imagePullPolicy: Never`, which tells Kubernetes to use the locally loaded image rather than trying to pull from a registry.

## Step 5: Load Image into Minikube and Deploy

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
kubectl apply -f kubernetes/backstage-deployment.yaml
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

## Step 6: Access Backstage

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
docker build -t backstage:1.0.1 -f Dockerfile backstage/

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
