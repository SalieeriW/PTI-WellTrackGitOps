# WellTrack GitOps Repository

This repository manages the Kubernetes manifests and Helm charts for the WellTrack application deployment using the GitOps approach.

## Architecture Overview

This GitOps repository is part of a CI/CD pipeline that includes:

- **GitHub Actions**: Builds Docker images and updates this GitOps repository
- **Harbor**: Private container registry for storing application images
- **ArgoCD**: GitOps controller that syncs manifests from this repository to the Kubernetes cluster

![Architecture Diagram](docs/architecture.png)

## Repository Structure

```
PTI-WellTrackGitOps/
├── charts/
│   ├── welltrack-frontend/
│   │   ├── Chart.yaml
│   │   ├── values.yaml
│   │   └── templates/
│   │       ├── deployment.yaml
│   │       ├── service.yaml
│   │       └── ingress.yaml
│   ├── welltrack-backend/
│   │   ├── Chart.yaml
│   │   ├── values.yaml
│   │   └── templates/
│   │       ├── deployment.yaml
│   │       ├── service.yaml
│   │       └── externalsecret.yaml
│   └── welltrack-ml/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           └── ...
└── argo-apps/
    ├── welltrack-frontend-app.yaml
    ├── welltrack-backend-app.yaml
    └── welltrack-ml-app.yaml
```

## Setup Instructions

### Prerequisites

- Kubernetes cluster (e.g., Kind)
- ArgoCD installed on the cluster
- Harbor registry configured
- Vault for secrets management 
- ExternalSecrets Operator installed 

### Deploy ArgoCD Applications

To deploy the applications to your Kubernetes cluster:

```bash
kubectl apply -f argo-apps/
```

This will create ArgoCD applications that automatically sync the Helm charts to your cluster.

## CI/CD Workflow

1. Developers push code to the application repositories (frontend/backend)
2. GitHub Actions in those repositories:
   - Build Docker images
   - Push images to Harbor registry
   - Update image tags in this GitOps repository
3. ArgoCD detects changes in this repository
4. ArgoCD automatically deploys the updated applications to Kubernetes

## Configuration

### Frontend Configuration

The frontend configuration is stored in `charts/welltrack-frontend/values.yaml`:

```yaml
replicaCount: 1
image:
  repository: harbor.welltrack.local/welltrack/welltrack-frontend
  tag: latest
  pullPolicy: Always
# ...
```

### Backend Configuration

The backend configuration is stored in `charts/welltrack-backend/values.yaml`:

```yaml
replicaCount: 1
image:
  repository: harbor.welltrack.local/welltrack/welltrack-backend
  tag: latest
  pullPolicy: Always
# ...
```

## Sensitive Information

Sensitive information is managed using HashiCorp Vault and the External Secrets Operator. The ExternalSecret resources in each chart reference secrets stored in Vault.

## Contributing

1. **Never** commit sensitive information to this repository
2. Always update image tags using the automated CI/CD process
3. Manual changes should be reserved for configuration updates only

## Related Repositories

- [WellTrack Frontend](https://github.com/hongda-zhu/PTI-WellTrackFront)
- [WellTrack Backend](https://github.com/hongda-zhu/PTI-WellTrackBack)
