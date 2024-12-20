# Blue-Green Deployment with ArgoCD

This repository demonstrates a blue-green deployment strategy for a application using ArgoCD. The deployment involves two versions of the application: **Blue** and **Green**, with traffic switching managed via Kubernetes Service selectors.

## Repository Structure

```
repo-root/
├── argocd-config/
│   ├── app-blue.yaml       # ArgoCD Application manifest for Blue
│   ├── app-green.yaml      # ArgoCD Application manifest for Green
├── app/
│   ├── base/                  # Base Kubernetes configuration
│   │   ├── deployment.yaml    # Deployment template
│   │   ├── service.yaml       # Service template
│   │   └── kustomization.yaml # Kustomize base file
│   ├── overlays/
│   │   ├── blue/              # Overlay for Blue deployment
│   │   │   ├── deployment-patch.yaml
│   │   │   └── kustomization.yaml
│   │   ├── green/             # Overlay for Green deployment
│   │       ├── deployment-patch.yaml
│   │       └── kustomization.yaml
```

## Prerequisites

- Kubernetes cluster
- ArgoCD installed and running in the cluster
- Docker installed to build application images
- A Git repository to store manifests and application files

## Steps

### 1. Build Docker Images

Navigate to the `blue` and `green` folders of your Node.js application and build the images:

```bash
docker build -t <your-dockerhub-username>/node-app:blue ./nodejs/blue
docker build -t <your-dockerhub-username>/node-app:green ./nodejs/green
```

Push the images to a Docker registry:

```bash
docker push <your-dockerhub-username>/node-app:blue
docker push <your-dockerhub-username>/node-app:green
```

### 2. Configure Kubernetes Resources

#### Base Deployment

**`app/base/deployment.yaml`**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
    spec:
      containers:
      - name: app
        image: node-app:latest # Placeholder, overridden by overlays
        ports:
        - containerPort: 3000
```

**`app/base/service.yaml`**:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-service
spec:
  selector:
    app: app
    version: blue # Default traffic routing to Blue version
  ports:
  - protocol: TCP
    port: 80
    targetPort: 3000
  type: LoadBalancer
```

#### Overlay for Blue

**`app/overlays/blue/deployment-patch.yaml`**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  template:
    metadata:
      labels:
        version: blue
    spec:
      containers:
      - name: app
        image: <your-dockerhub-username>/node-app:blue
```

#### Overlay for Green

**`app/overlays/green/deployment-patch.yaml`**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  template:
    metadata:
      labels:
        version: green
    spec:
      containers:
      - name: app
        image: <your-dockerhub-username>/node-app:green
```

### 3. Configure ArgoCD Applications

Create ArgoCD Application manifests for both Blue and Green deployments.

#### Blue Application

**`argocd-apps/app-blue.yaml`**:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-blue
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/<your-repo>.git'
    targetRevision: HEAD
    path: app/overlays/blue
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

#### Green Application

**`argocd-apps/app-green.yaml`**:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-green
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/<your-repo>.git'
    targetRevision: HEAD
    path: app/overlays/green
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### 4. Deploy with ArgoCD

Apply the ArgoCD manifests to your cluster:

```bash
kubectl apply -f argocd-apps/app-blue.yaml
kubectl apply -f argocd-apps/app-green.yaml
```

Sync the applications from the ArgoCD UI or CLI:

```bash
argocd app sync app-blue
argocd app sync app-green
```

### 5. Switch Traffic

To switch traffic to the Green version:

1. Update the `spec.selector.version` in `app/base/service.yaml` to:
   ```yaml
   selector:
     app: app
     version: green
   ```

2. Commit and push the changes to the repository.
3. ArgoCD will automatically synchronize the change.

### 6. Test Deployment

- Access the application via the LoadBalancer URL to verify the active version.
- Use ArgoCD UI to monitor deployments and rollbacks.

## Notes
- Ensure Docker images are available in the registry.
- Use proper versioning and tagging strategies for images.
- Always test changes in a staging environment before production rollout.

---

For further questions or improvements, feel free to open an issue!

