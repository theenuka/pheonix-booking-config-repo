# Phoenix Booking - Configuration Repository

This repository serves as the **GitOps Source of Truth** for the Phoenix Booking system. It contains all Kubernetes manifests, Helm chart values, and ArgoCD application definitions, managed by ArgoCD.

## Structure

*   **`root-app.yaml`**: The single entry point for ArgoCD, which deploys the App of Apps.
*   **`app-of-apps/`**: Contains the ArgoCD Application definitions for all platform services and the hotel reservation application for different environments.
*   **`apps/`**: Contains the Helm chart for the `hotel-reservation` application.
    *   `values.yaml`: Default values for the Helm chart.
    *   `values-staging.yaml`: Overrides for the staging environment.
    *   `values-production.yaml`: Overrides for the production environment.
*   **`platform/`**: Contains manifests and configurations for infrastructure services like Istio, Vault, Prometheus, etc.
*   **`environments/`**: This directory is now deprecated in favor of environment-specific `values` files in the Helm chart.

## GitOps Workflow

The deployment process follows a pure GitOps model managed by ArgoCD:

1.  **Bootstrapping:** The cluster is bootstrapped by manually applying `root-app.yaml` to the ArgoCD namespace. This single application deploys all other applications defined in `app-of-apps/`.
2.  **CI Pipeline**: A CI pipeline (in the `Hotel-Reservation` repository) builds a new Docker image and pushes it to Harbor.
3.  **Image Update**: Argo CD Image Updater monitors the image registry for new tags. When a new image is pushed, it automatically updates the corresponding image tag in the Kubernetes manifests within this repository.
4.  **Sync**: ArgoCD detects the change in this repository (the updated image tag) and synchronizes the Kubernetes cluster to match the new desired state.

### Branching Strategy

*   `main` branch: Represents the desired state of the **production** environment.
*   `staging` branch: Represents the desired state of the **staging** environment.

## Key Improvements

*   **Consolidated Bootstrapping:** The ArgoCD bootstrapping process is now managed by a single `root-app.yaml`, and the previous Ansible-based bootstrapping has been removed.
*   **Environment-Specific Configurations:** The `hotel-reservation` Helm chart now correctly uses `values-staging.yaml` and `values-production.yaml` for environment-specific configurations.
*   **Externalized Database:** The in-cluster MongoDB has been removed from the Helm chart. The application is now configured to connect to an external, managed MongoDB database, with connection details provided through the `values.yaml` files.

## Security

*   **mTLS**: Enforced globally via `platform/istio/peer-authentication.yaml`.
*   **Secrets**: Managed via External Secrets Operator and Vault.
*   **Secure CI/CD:** The CI/CD pipeline has been secured by removing insecure communication with the Harbor registry and enforcing a blocking linting step.
