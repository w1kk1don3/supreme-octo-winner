# External Secrets

External Secrets enables you to manage Kubernetes secrets stored in external secret management systems (AWS Secrets Manager, HashiCorp Vault, 1Password, etc.) and synchronize them to Kubernetes `Secret` resources.

## Overview

This deployment uses the official External Secrets Operator Helm chart from the GHCR OCI registry. It configures:

- **2 replicas** for high availability
- **CRDs** automatically created and updated
- **RBAC** resources for proper authorization
- **ServiceMonitor** for Prometheus monitoring integration

## Files Structure

```
external-secrets/
├── namespace.yaml                         # External Secrets namespace
├── kustomization.yaml                     # Parent Kustomization
└── external-secrets/
    ├── ks.yaml                            # Flux Kustomization resource
    └── app/
        ├── helmrelease.yaml               # Helm Release definition
        ├── ocirepository.yaml             # OCI chart repository
        └── kustomization.yaml             # App Kustomization
```

## Quick Start

### 1. Verify Deployment

```bash
# Check if the External Secrets operator is running
kubectl get pods -n external-secrets

# Check HelmRelease status
kubectl get helmrelease -n external-secrets

# Check OCIRepository status
kubectl get ocirepository -n external-secrets
```

### 2. Verify the CRDs are installed

```bash
# List all External Secrets CRDs
kubectl get crd | grep external-secrets
```

Expected CRDs:
- `secretstores.external-secrets.io`
- `clustersecretstores.external-secrets.io`
- `externalsecrets.external-secrets.io`

## Configuration

### Common SecretStore Examples

#### AWS Secrets Manager

```yaml
---
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets
  namespace: external-secrets
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
```

#### HashiCorp Vault

```yaml
---
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-secrets
  namespace: external-secrets
spec:
  provider:
    vault:
      server: "https://vault.example.com:8200"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "external-secrets"
          serviceAccountRef:
            name: external-secrets-sa
```

#### 1Password

```yaml
---
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: onepwd-secrets
  namespace: external-secrets
spec:
  provider:
    onepwd:
      connectHost: http://op-connect:8080
      vaults:
        "My Vault": "id123"
      auth:
        secretRef:
          connectTokenSecretRef:
            name: onepassword-token
            key: token
```

### Example External Secret

Once a `SecretStore` is configured:

```yaml
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-secret
  namespace: default
spec:
  refreshInterval: 15m
  secretStoreRef:
    name: aws-secrets
    kind: SecretStore
  target:
    name: my-secret
    creationPolicy: Owner
    template:
      engineVersion: v2
      data:
        username: "{{ .username }}"
        password: "{{ .password }}"
  data:
    - secretKey: username
      remoteRef:
        key: my-app/username
    - secretKey: password
      remoteRef:
        key: my-app/password
```

This will:
1. Fetch the secrets from your external store every 15 minutes
2. Create/update a Kubernetes `Secret` named `my-secret` with the data
3. The Secret controller will manage the Secret lifecycle

## Monitoring

The External Secrets operator exposes Prometheus metrics which are automatically scraped if your Prometheus is configured to scrape ServiceMonitors.

```bash
# Query External Secrets metrics
kubectl port-forward -n external-secrets svc/external-secrets 8080:8080
curl http://localhost:8080/metrics
```

## Troubleshooting

### Check ExternalSecret Status

```bash
# Check the status of an ExternalSecret
kubectl describe externalsecret my-secret -n default

# Check the events
kubectl get events -n default --field-selector involvedObject.name=my-secret
```

### View operator logs

```bash
# View External Secrets operator logs
kubectl logs -n external-secrets -l app.kubernetes.io/name=external-secrets -f
```

### Common Issues

1. **Secret not syncing**: Check the SecretStore connection and authorization
2. **Wrong secret values**: Verify the `remoteRef.key` paths exist in your external store
3. **RBAC errors**: Ensure the ServiceAccount has proper permissions

## Updating

The HelmRelease is configured with:
- `interval: 1h` - checks for updates every hour
- `crds: CreateReplace` - CRDs are automatically updated

To manually trigger an update:

```bash
flux reconcile helmrelease external-secrets -n external-secrets
```

## References

- [External Secrets Operator Documentation](https://external-secrets.io)
- [Helm Chart](https://github.com/external-secrets/external-secrets/tree/main/deploy/charts/external-secrets)
- [Supported Secret Stores](https://external-secrets.io/latest/introduction/overview/)
