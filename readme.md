# WellTrack GitOps Repository

This repository stores the application-specific Kubernetes configurations (Helm charts) for the WellTrack project, serving as the **target state** for deployments managed by Argo CD.

## Architecture Overview

This GitOps repository is part of a deployment strategy orchestrated by a separate **DevOps repository** ([PTI-WellTrackDevops](https://github.com/hongda-zhu/PTI-WellTrackDevops)) which implements Argo CD's "App of Apps" pattern.

The overall flow includes:

- **Application Repositories** (e.g., [PTI-WellTrackBack](https://github.com/hongda-zhu/PTI-WellTrackBack), [PTI-WellTrackFront](https://github.com/hongda-zhu/PTI-WellTrackFront)): Contain application source code.
- **GitHub Actions**: Run in application repositories to build/push Docker images (e.g., to GHCR or Harbor) and update the corresponding Helm chart `values.yaml` in *this* GitOps repository.
- **This Repository (`PTI-WellTrackGitOps`)**: Contains the Helm charts defining the desired state for each application (Deployment, Service, ExternalSecret, etc.). It is updated by CI pipelines.
- **DevOps Repository (`PTI-WellTrackDevops`)**: Defines *which* Argo CD applications should exist and points them to the charts within *this* repository. Manages the deployment of infrastructure components (Harbor, Vault, ExternalSecrets Operator, Argo CD itself, etc.) and the application definitions.
- **ArgoCD**: Runs in the cluster, monitors the charts in *this* repository (as instructed by `PTI-WellTrackDevops`), and synchronizes the cluster state to match the desired state defined here.
- **Supporting Infrastructure**: Includes a container registry (e.g., Harbor/GHCR), Vault for secrets, and the ExternalSecrets Operator.

*(Consider updating the architecture diagram if it exists)*

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
└── readme.md 
# (The argo-apps/ directory is no longer used as Applications are defined in PTI-WellTrackDevops)
```

## Setup & Deployment Process

Deployment is managed centrally via the [PTI-WellTrackDevops](https://github.com/hongda-zhu/PTI-WellTrackDevops) repository.

1.  **Bootstrap Argo CD & Infrastructure:** Follow the setup instructions in the `PTI-WellTrackDevops` repository. This typically involves applying a root Argo CD application (`bootstrap.yaml`) which then deploys Argo CD itself and all other infrastructure components (Harbor, Vault, External Secrets Operator, etc.) along with the Argo CD `Application` definitions for the frontend, backend, and ML services.
2.  **Argo CD Takes Over:** Once bootstrapped, Argo CD (configured by `PTI-WellTrackDevops`) will automatically:
    *   Monitor the Helm charts within the `charts/` directory of *this* repository.
    *   Sync the defined resources (Deployments, Services, etc.) to the Kubernetes cluster.

**You do not apply manifests directly from this repository.**

## CI/CD Workflow

1. Developers push code to application repositories (e.g., `PTI-WellTrackBack`).
2. GitHub Actions in those repositories trigger:
   - Build Docker images.
   - Push images to the container registry (e.g., GHCR/Harbor).
   - **Checkout *this* GitOps repository (`PTI-WellTrackGitOps`).**
   - **Update the `image.tag` in the corresponding chart's `values.yaml` (e.g., `charts/welltrack-backend/values.yaml`).**
   - **Commit and push the change back to *this* repository.**
3. Argo CD detects the commit pushed to *this* repository.
4. Argo CD automatically synchronizes the cluster state by applying the updated Helm chart, rolling out the new application version.

## Configuration

The configuration for each application is managed within its Helm chart's `values.yaml` file located in the `charts/` directory.

### Frontend Configuration (`charts/welltrack-frontend/values.yaml`):
```yaml
replicaCount: 1
image:
  repository: ghcr.io/hongda-zhu/welltrack-frontend # Or harbor.welltrack.local/...
  tag: latest # This value is updated by CI
  pullPolicy: Always
# ...
```

### Backend Configuration (`charts/welltrack-backend/values.yaml`):
```yaml
replicaCount: 1
image:
  repository: ghcr.io/hongda-zhu/welltrack-backend # Or harbor.welltrack.local/...
  tag: latest # This value is updated by CI
  pullPolicy: Always
# ...
```

*(Update repository URLs and other default values as needed)*

## Sensitive Information

Sensitive information (like database passwords, API keys) is managed using HashiCorp Vault and the Kubernetes External Secrets Operator. 
- `ExternalSecret` resources defined within the Helm templates (e.g., `charts/welltrack-backend/templates/externalsecret.yaml`) specify which secrets to fetch from Vault.
- The External Secrets Operator reads these resources and creates standard Kubernetes `Secret` objects in the appropriate namespace, which are then mounted by the application pods.
- **Never commit Kubernetes `Secret` manifests or sensitive values directly to this repository.**

## Contributing

1. **Primary Changes via CI:** Updates to image tags within `values.yaml` files should **only** happen through the automated CI/CD pipelines running in the application repositories.
2. **Manual Changes:** Direct commits to this repository should generally be limited to:
    - Modifying Helm chart templates (`templates/`).
    - Adjusting non-sensitive default configuration in `values.yaml`.
    - Updating `Chart.yaml` files.
3. **Structure:** Maintain the established chart structure within the `charts/` directory.

## Related Repositories

- **Orchestration:** [PTI-WellTrackDevops](https://github.com/hongda-zhu/PTI-WellTrackDevops)
- **Applications:**
    - [WellTrack Frontend](https://github.com/hongda-zhu/PTI-WellTrackFront)
    - [WellTrack Backend](https://github.com/hongda-zhu/PTI-WellTrackBack)
    - *(Add ML repo if applicable)*
