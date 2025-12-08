# Phoenix Booking - Configuration Repository

This repository serves as the **GitOps Source of Truth** for the Phoenix Booking system. It contains all Kubernetes manifests, Helm chart values, and ArgoCD application definitions.

## Structure

*   **`app-of-apps/`**: The entry point for ArgoCD. Contains the "Root App" which points to other applications.
*   **`apps/`**: Application-specific manifests (e.g., `hotel-reservation`).
*   **`platform/`**: Infrastructure services (Istio, Vault, Monitoring, Logging).
*   **`environments/`**: Environment-specific overlays (staging/production).

## GitOps Workflow

1.  **CI Pipeline**: Builds a new Docker image and pushes it to the registry.
2.  **Update**: The CI pipeline commits a change to this repository (e.g., updating the `image` tag in `apps/hotel-reservation/values.yaml`).
3.  **Sync**: ArgoCD detects the change in this repository and synchronizes the Kubernetes cluster to match the new state.

## Security

*   **mTLS**: Enforced globally via `platform/istio/peer-authentication.yaml`.
*   **Secrets**: Managed via External Secrets Operator and Vault.
