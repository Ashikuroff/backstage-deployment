# backstage-deployment

Production-like local GitOps setup for Backstage on Docker Desktop Kubernetes using Argo CD and GitHub Actions.

## Repository layout

```text
backstage-deployment/
├── .github/workflows/deploy.yaml
├── argocd-app.yaml
├── backstage/
│   ├── app-config.yaml
│   ├── app-config.production.yaml
│   └── packages/backend/Dockerfile
└── k8s/
    ├── deployment.yaml
    ├── ingress.yaml
    ├── namespace.yaml
    ├── persistentvolumeclaim.yaml
    └── service.yaml
```

## 0. Prerequisites

Use the Docker Desktop Kubernetes context before you apply anything:

```bash
kubectl config get-contexts -o name
kubectl config use-context docker-desktop
kubectl get nodes
```

If `docker-desktop` is missing, enable Kubernetes in Docker Desktop first.

## 1. Install Argo CD

Recommended local install using the official manifest:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl rollout status deployment/argocd-server -n argocd --timeout=180s
kubectl get pods -n argocd
```

Helm alternative:

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm upgrade --install argocd argo/argo-cd -n argocd --create-namespace
kubectl rollout status deployment/argocd-server -n argocd --timeout=180s
```

Access Argo CD locally. Port-forward is the reliable default on Docker Desktop:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open the UI:

```text
https://localhost:8080
```

Optional service patch if you want to expose the service differently:

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec":{"type":"LoadBalancer"}}'
kubectl get svc argocd-server -n argocd
```

Retrieve the initial admin password:

```bash
argocd admin initial-password -n argocd
```

Fallback with `kubectl`:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

Login with CLI:

```bash
argocd login localhost:8080 --username admin --password "$(argocd admin initial-password -n argocd)" --insecure
argocd account update-password
kubectl delete secret argocd-initial-admin-secret -n argocd
```

## 2. Backstage app

The Backstage app lives in `backstage/`. It uses the standard Backstage backend Dockerfile at `backstage/packages/backend/Dockerfile`.

Key runtime defaults in this repo:

- Single container on port `7007`
- SQLite plugin databases persisted under `/var/backstage`
- Guest auth enabled for local use
- TechDocs set to `local` inside the container to avoid Docker-in-Docker

Local build test:

```bash
cd backstage
node .yarn/releases/yarn-4.4.1.cjs install --immutable
node .yarn/releases/yarn-4.4.1.cjs tsc
node .yarn/releases/yarn-4.4.1.cjs build:backend
docker build -t backstage:bootstrap -f packages/backend/Dockerfile .
docker run --rm -p 7007:7007 -e APP_BASE_URL=http://localhost:7007 backstage:bootstrap
```

Open:

```text
http://localhost:7007
```

## 3. Kubernetes manifests

Apply the manifests directly if you want to test without Argo CD first:

```bash
kubectl apply -k k8s
kubectl get all -n backstage
kubectl get pvc -n backstage
kubectl port-forward svc/backstage -n backstage 7007:80
```

Open:

```text
http://localhost:7007
```

The default deployment image is:

```text
backstage:bootstrap
```

For the first local deployment on Docker Desktop, build that bootstrap image locally first:

```bash
cd backstage
node .yarn/releases/yarn-4.4.1.cjs install --immutable
node .yarn/releases/yarn-4.4.1.cjs tsc
node .yarn/releases/yarn-4.4.1.cjs build:backend
docker build -t backstage:bootstrap -f packages/backend/Dockerfile .
```

GitHub Actions then replaces that image reference with a commit-specific GHCR tag after each successful build.

Optional ingress:

```bash
kubectl apply -f k8s/ingress.yaml
kubectl get ingress -n backstage
```

If you want to access Backstage through ingress at `http://backstage.localtest.me`, update `APP_BASE_URL` in `k8s/deployment.yaml` from `http://localhost:7007` to `http://backstage.localtest.me` and let Argo CD sync the change.

## 4. Argo CD GitOps application

Bootstrap the application into Argo CD:

```bash
kubectl apply -f argocd-app.yaml
argocd app get backstage
argocd app sync backstage
kubectl get pods -n backstage
```

The application watches:

```text
https://github.com/Ashikuroff/backstage-deployment.git
```

at path:

```text
k8s
```

## 5. GitHub Actions automation

The workflow file is:

```text
.github/workflows/deploy.yaml
```

What it does on every push to `main` that changes `backstage/**`:

1. Installs Node 22.
2. Runs the repo-pinned Yarn 4 binary with `install --immutable`.
3. Builds the Backstage backend bundle.
4. Builds and pushes the image to `ghcr.io/ashikuroff/backstage`.
5. Rewrites `k8s/deployment.yaml` with a new image tag like `sha-<commit>`.
6. Commits the manifest change back to `main`.

GitHub requirements:

- `Actions` must be enabled.
- `GITHUB_TOKEN` package write access must be allowed.
- The GHCR package should be public for the simplest local pull flow.

If you keep the package private, create an image pull secret and add it to the deployment before syncing with Argo CD.

## 6. End-to-end flow

1. Push a code change under `backstage/`.
2. GitHub Actions builds a new container image and pushes it to GHCR.
3. The workflow commits the new image tag into `k8s/deployment.yaml`.
4. Argo CD detects the Git commit in `backstage-deployment`.
5. Argo CD syncs the updated manifest to the Docker Desktop Kubernetes cluster.
6. Kubernetes rolls out the new Backstage pod.
7. Access the UI with:

```bash
kubectl port-forward svc/backstage -n backstage 7007:80
```

Then open:

```text
http://localhost:7007
```

## 7. Quick verification commands

Argo CD:

```bash
kubectl get pods -n argocd
argocd app list
argocd app get backstage
```

Backstage:

```bash
kubectl get deploy,svc,pods -n backstage
kubectl logs deploy/backstage -n backstage --tail=100
kubectl port-forward svc/backstage -n backstage 7007:80
curl -I http://localhost:7007
```
