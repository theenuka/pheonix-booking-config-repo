# Phoenix Booking - Configuration Repository

This repository serves as the **GitOps Source of Truth** for the Phoenix Booking system. It contains all Kubernetes manifests, Helm chart values, and ArgoCD application definitions.

## Structure

*   **`app-of-apps/`**: The entry point for ArgoCD. Contains the "Root App" which points to other applications.
*   **`apps/`**: Application-specific manifests (e.g., `hotel-reservation`).
*   **`platform/`**: Infrastructure services (Istio, Vault, Monitoring, Logging).
*   **`environments/`**: Environment-specific overlays (staging/production).

## GitOps Workflow

1.  **CI Pipeline**: Builds a new Docker image and pushes it to the registry.
2.  **Image Update**: Argo CD Image Updater monitors the image registry for new tags. When a new image is pushed, Image Updater automatically updates the corresponding image tag in the Kubernetes manifests within this repository.
3.  **Sync**: ArgoCD detects the change in this repository (the updated image tag) and synchronizes the Kubernetes cluster to match the new desired state.

## Security

*   **mTLS**: Enforced globally via `platform/istio/peer-authentication.yaml`.
*   **Secrets**: Managed via External Secrets Operator and Vault.
