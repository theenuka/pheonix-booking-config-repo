# Kustomize Configuration (Optional)

This directory contains Kustomize overlays as an **alternative configuration management approach** for platform services.

## Current Architecture

The Phoenix Booking project primarily uses **Helm** for configuration management, which is the recommended approach for this application because:

- ✅ Better templating capabilities for complex configurations
- ✅ Dependency management between charts
- ✅ Values-based configuration is more maintainable
- ✅ Excellent integration with ArgoCD
- ✅ Argo Rollouts already implemented for advanced deployments

## When to Use Kustomize

Kustomize is provided here as a **complementary tool** for specific use cases:

1. **Quick patches** to third-party manifests without Helm charts
2. **Simple configuration overlays** for non-Helm resources
3. **Development environments** where Helm overhead isn't needed

## Structure

```
kustomize/
├── base/                  # Base Kubernetes manifests
│   └── ...
├── overlays/
│   ├── development/       # Development environment overrides
│   ├── staging/           # Staging environment overrides
│   └── production/        # Production environment overrides
└── README.md
```

## Usage

```bash
# Preview changes for staging
kubectl kustomize overlays/staging

# Apply to cluster
kubectl apply -k overlays/staging

# Or with ArgoCD
argocd app create my-app \
  --repo https://github.com/theenuka/pheonix-booking-config-repo.git \
  --path kustomize/overlays/production \
  --dest-namespace phoenix-app \
  --dest-server https://kubernetes.default.svc
```

## Recommendation

**Continue using Helm for the main application.** This Kustomize structure is optional and provided for educational purposes and specific edge cases.

The current Helm-based approach with ArgoCD GitOps and Argo Rollouts is **industry best practice** for microservices deployments.
