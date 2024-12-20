# blue-green-deployment

app/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
├── overlays/
│   ├── blue/
│   │   ├── deployment-patch.yaml
│   │   ├── kustomization.yaml
│   ├── green/
│       ├── deployment-patch.yaml
│       ├── kustomization.yaml

Base directory: Contains common manifests (e.g., Deployment and Service).
Overlays directory: Contains specific configurations for blue and green.