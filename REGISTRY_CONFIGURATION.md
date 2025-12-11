# Container Registry Configuration

This document explains how to configure the Harbor container registry URL for the Phoenix Booking project.

## Overview

The project uses a **global registry configuration** approach to avoid hardcoding registry URLs in multiple places.

## Configuration Location

**File:** `apps/hotel-reservation/values.yaml`

```yaml
global:
  registry:
    url: "harbor.phoenix-booking.com"  # ← Change this
    project: "phoenix-project"
    imagePullSecrets:
      - name: harbor-creds
```

## Setup Steps

### 1. Update Global Registry URL

Edit `apps/hotel-reservation/values.yaml`:

```yaml
global:
  registry:
    url: "your-harbor-domain.com"  # or use ELB URL: k8s-harbor-xxx.elb.amazonaws.com
    project: "phoenix-project"     # Harbor project name
```

### 2. Create Harbor Credentials Secret

Create the `harbor-creds` secret in both staging and production namespaces:

```bash
# For Staging
kubectl create secret docker-registry harbor-creds \
  --docker-server=your-harbor-domain.com \
  --docker-username=admin \
  --docker-password=Harbor12345 \
  --namespace=staging

# For Production
kubectl create secret docker-registry harbor-creds \
  --docker-server=your-harbor-domain.com \
  --docker-username=admin \
  --docker-password=Harbor12345 \
  --namespace=production
```

**Recommended:** Use External Secrets Operator to manage this secret from Vault:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: harbor-creds
  namespace: staging
spec:
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: harbor-creds
    creationPolicy: Owner
    template:
      type: kubernetes.io/dockerconfigjson
      data:
        .dockerconfigjson: |
          {
            "auths": {
              "{{ .harborUrl }}": {
                "username": "{{ .harborUsername }}",
                "password": "{{ .harborPassword }}",
                "auth": "{{ printf "%s:%s" .harborUsername .harborPassword | b64enc }}"
              }
            }
          }
  data:
    - secretKey: harborUrl
      remoteRef:
        key: harbor
        property: url
    - secretKey: harborUsername
      remoteRef:
        key: harbor
        property: username
    - secretKey: harborPassword
      remoteRef:
        key: harbor
        property: password
```

### 3. Update GitHub Actions Secrets

Update the following GitHub Actions secrets in your repository:

- `HARBOR_URL`: Your Harbor registry URL (e.g., `harbor.phoenix-booking.com`)
- `HARBOR_USERNAME`: Harbor username (e.g., `admin`)
- `HARBOR_PASSWORD`: Harbor password

```bash
# Using GitHub CLI
gh secret set HARBOR_URL --body "harbor.phoenix-booking.com" --repo theenuka/Hotel-Reservation-System
gh secret set HARBOR_USERNAME --body "admin" --repo theenuka/Hotel-Reservation-System
gh secret set HARBOR_PASSWORD --body "YourSecurePassword" --repo theenuka/Hotel-Reservation-System
```

### 4. Update ArgoCD Image Updater Annotations

The ArgoCD Image Updater uses the following annotation format:

```yaml
annotations:
  argocd-image-updater.argoproj.io/image-list: |
    frontend={{ .Values.global.registry.url }}/{{ .Values.global.registry.project }}/phoenix-frontend-service
```

This is automatically configured in the ArgoCD Application manifests.

## Environment-Specific Overrides

You can override the registry URL per environment if needed:

**`environments/staging/staging-frontend.yaml`:**
```yaml
global:
  registry:
    url: "staging-harbor.phoenix-booking.com"  # Different registry for staging
```

**`environments/production/production-frontend.yaml`:**
```yaml
global:
  registry:
    url: "harbor.phoenix-booking.com"  # Production registry
```

## How It Works

### Image Name Construction

The image repository is constructed using this template:

```
{{ .Values.global.registry.url }}/{{ .Values.global.registry.project }}/{{ .Values.image.name }}:{{ .Values.image.tag }}
```

**Example:**
- `global.registry.url` = `harbor.phoenix-booking.com`
- `global.registry.project` = `phoenix-project`
- `image.name` = `phoenix-frontend-service`
- `image.tag` = `5c0c469477dd...` (Git SHA)

**Result:**
```
harbor.phoenix-booking.com/phoenix-project/phoenix-frontend-service:5c0c469477dd...
```

### Benefits of This Approach

1. **Single Source of Truth**: Change registry URL in one place
2. **Environment Flexibility**: Override per environment if needed
3. **No Hardcoding**: Registry URL is never hardcoded in charts
4. **Easy Migration**: Switch registries without changing multiple files
5. **GitOps Friendly**: All configuration is declarative and versioned

## Verification

After configuration, verify the correct image is being pulled:

```bash
# Check deployment
kubectl get deployment frontend -n staging -o jsonpath='{.spec.template.spec.containers[0].image}'

# Expected output:
# harbor.phoenix-booking.com/phoenix-project/phoenix-frontend-service:latest

# Check image pull secrets
kubectl get deployment frontend -n staging -o jsonpath='{.spec.template.spec.imagePullSecrets}'

# Expected output:
# [{"name":"harbor-creds"}]
```

## Troubleshooting

### ImagePullBackOff Error

```bash
kubectl describe pod <pod-name> -n staging
```

**Common causes:**
1. Incorrect registry URL
2. Invalid credentials in `harbor-creds` secret
3. Image doesn't exist in Harbor
4. Network connectivity issues

**Fix:**
```bash
# Test registry connectivity
docker login harbor.phoenix-booking.com
docker pull harbor.phoenix-booking.com/phoenix-project/phoenix-frontend-service:latest

# Verify secret
kubectl get secret harbor-creds -n staging -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq .
```

### ArgoCD Image Updater Not Working

Check the image updater logs:

```bash
kubectl logs -n argocd deployment/argocd-image-updater -f
```

Verify annotations in the Application:

```bash
kubectl get application hotel-reservation-staging -n argocd -o yaml | grep image-list
```

## Migration from Hardcoded URLs

If you have hardcoded URLs in your values files, use this script to update them:

```bash
# Run from config repo root
find apps/hotel-reservation/charts/*/values.yaml -type f -exec sed -i '' \
  's|repository: k8s-harbor-.*\.elb\.amazonaws\.com/phoenix-project/|repository: ""|g' {} \;
```

Then add the `name` field to each values.yaml:

```yaml
image:
  repository: ""
  name: "phoenix-<service>-service"
  tag: "latest"
```

## Reference

- **Global Values**: `apps/hotel-reservation/values.yaml`
- **Frontend Chart**: `apps/hotel-reservation/charts/frontend/`
- **Environment Overrides**: `environments/staging/` and `environments/production/`
- **ArgoCD Apps**: `app-of-apps/apps/`
