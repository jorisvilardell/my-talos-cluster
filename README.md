# My Talos Cluster

A production-ready Kubernetes cluster built on Talos Linux with GitOps principles using ArgoCD.

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [Initial Setup](#initial-setup)
- [Cluster Bootstrapping](#cluster-bootstrapping)
- [Disaster Recovery](#disaster-recovery)
- [Operations](#operations)
- [Troubleshooting](#troubleshooting)

## Architecture Overview

This cluster implements a GitOps workflow with the following components:

- **Talos Linux**: Immutable Kubernetes distribution
- **ArgoCD**: GitOps continuous delivery tool
- **Traefik**: Ingress controller and reverse proxy
- **Longhorn**: Distributed block storage
- **SOPS + Age**: Secret encryption at rest

### Directory Structure

```
.
├── argocd/
│   └── root-app.yaml           # ArgoCD App-of-Apps pattern
├── infra-core/
│   ├── Chart.yaml              # Core infrastructure Helm chart
│   └── values.yaml             # ArgoCD, Traefik, base config
├── cluster-config/
│   └── templates/              # Namespace definitions, RBAC
├── longhorn/
│   └── values.yaml             # Distributed storage config
└── apps/
    └── ouerdia/                # Application deployments
```

## Prerequisites

- **kubectl**: Kubernetes CLI tool
- **helm**: Kubernetes package manager (v3+)
- **age**: Encryption tool for SOPS
- **sops**: Secret encryption CLI
- Access to your Talos cluster (kubeconfig)

## Initial Setup

### 1. Generate Age Encryption Key

```bash
# Generate a new age key pair
age-keygen -o age.key

# Save the public key for .sops.yaml configuration
cat age.key | grep "# public key:"
```

### 2. Configure SOPS

Create or update `.sops.yaml` in your repository root with your age public key.

### 3. Store Your Private Key Securely

⚠️ **Critical**: Your `age.key` file is the master key for all secrets. Store it securely:

- Use a password manager
- Keep encrypted backups
- Never commit it to Git

## Cluster Bootstrapping

### Full Bootstrap (Fresh Cluster)

Follow these steps in order when setting up a new cluster or recovering from a complete failure:

#### Step 1: Create the Infrastructure Namespace

```bash
kubectl create namespace infra-system
```

#### Step 2: Install the Infra-Core Chart

This deploys ArgoCD, Traefik, and core infrastructure components:

```bash
helm upgrade --install infra-core ./infra-core \
  --namespace infra-system \
  --create-namespace
```

#### Step 3: Create the SOPS Secret (Critical!) 🗝️

ArgoCD needs your Age private key to decrypt secrets in the Git repository.

According to `infra-core/values.yaml`, ArgoCD expects:
- **Secret name**: `sops-age-key`
- **Key name**: `keys.txt`

```bash
# Replace 'age.key' with your actual private key file path
kubectl create secret generic sops-age-key \
  --namespace infra-system \
  --from-file=keys.txt=age.key
```

**Verify the secret was created:**

```bash
kubectl get secret sops-age-key -n infra-system
```

#### Step 4: Deploy the Root Application

This activates ArgoCD's App-of-Apps pattern, which will automatically deploy all other applications:

```bash
kubectl apply -f argocd/root-app.yaml
```

#### Step 5: Monitor ArgoCD Sync

Wait for ArgoCD to synchronize all applications:

```bash
# Check pod status
kubectl get pods -n infra-system

# Watch ArgoCD applications
kubectl get applications -n infra-system

# Access ArgoCD UI (if configured)
kubectl port-forward svc/argocd-server -n infra-system 8080:443
```

## Disaster Recovery

### Scenario: Accidental Namespace Deletion

**Don't panic!** If you've accidentally deleted the `infra-system` namespace or need to rebuild from scratch, follow the [Cluster Bootstrapping](#cluster-bootstrapping) steps above.

### Why SOPS Secret is Critical

Without the SOPS Age key secret, ArgoCD will:
- ✅ Start successfully
- ❌ **Fail to decrypt any encrypted secrets**
- ❌ Applications will remain in sync errors

Always restore the SOPS secret **before** deploying the root application.

### Troubleshooting SOPS Decryption Issues

If applications are stuck with "Failed to decrypt" errors:

```bash
# Check ArgoCD repo-server logs
kubectl logs -n infra-system -l app.kubernetes.io/name=argocd-repo-server

# Verify the secret exists and has the correct key name
kubectl get secret sops-age-key -n infra-system -o yaml

# Restart the repo-server to pick up secret changes
kubectl rollout restart deployment argocd-repo-server -n infra-system
```

## Operations

### Accessing ArgoCD UI

```bash
# Get the initial admin password
kubectl -n infra-system get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Port forward to access UI
kubectl port-forward svc/argocd-server -n infra-system 8080:443

# Open https://localhost:8080
```

### Managing Encrypted Secrets

```bash
# Encrypt a new secret file
sops --encrypt --age <your-age-public-key> secret.yaml > secret.sops.yaml

# Edit an encrypted secret
sops secret.sops.yaml

# Decrypt for viewing
sops --decrypt secret.sops.yaml
```

### Updating Applications

All applications are managed via Git. To update:

1. Modify the appropriate values file in your Git repository
2. Commit and push changes
3. ArgoCD will automatically detect and sync changes

For immediate sync:

```bash
# Sync a specific application
kubectl patch app <app-name> -n infra-system \
  -p '{"operation":{"initiatedBy":{"username":"admin"},"sync":{"revision":"HEAD"}}}' \
  --type merge

# Or use ArgoCD CLI
argocd app sync <app-name>
```

### Longhorn Storage

Longhorn provides distributed block storage. After bootstrap, verify:

```bash
# Check Longhorn namespace
kubectl get pods -n longhorn-system

# Access Longhorn UI (if exposed via Ingress)
# Or port-forward: kubectl port-forward -n longhorn-system svc/longhorn-frontend 8000:80
```

## Troubleshooting

### ArgoCD Application Stuck in Progressing

```bash
# View application details
kubectl describe application <app-name> -n infra-system

# Check sync status
argocd app get <app-name>

# Force sync
argocd app sync <app-name> --force
```

### Namespace Already Exists Errors

If ArgoCD complains about existing namespaces (like `longhorn-system`), ensure your `cluster-config` includes proper namespace definitions with:

```yaml
annotations:
  argocd.argoproj.io/sync-options: ServerSideApply=true
```

This allows ArgoCD to adopt existing resources.

### Storage Issues

```bash
# Check PVCs
kubectl get pvc --all-namespaces

# Check Longhorn volumes
kubectl get volumes.longhorn.io -n longhorn-system

# View Longhorn node status
kubectl get nodes.longhorn.io -n longhorn-system
```

### Complete Cluster Reset

⚠️ **Warning**: This will destroy all data!

```bash
# Delete all ArgoCD applications
kubectl delete applications --all -n infra-system

# Delete the infra-core release
helm uninstall infra-core -n infra-system

# Delete the namespace
kubectl delete namespace infra-system

# Follow the bootstrapping steps to rebuild
```

## Best Practices

1. **Always backup your Age private key** - Store it in multiple secure locations
2. **Use Git branches** - Test changes in a branch before merging to main
3. **Monitor ArgoCD sync status** - Set up alerts for sync failures
4. **Regular cluster backups** - Use Velero or similar tools
5. **Document custom configurations** - Keep this README updated with your specific setup
6. **Test disaster recovery** - Periodically verify your bootstrap process works

## SRE Tips

- **Monitor `argocd-repo-server` logs** when diagnosing sync issues - this is where SOPS decryption happens
- **Use ArgoCD sync waves** to control deployment order (via annotations)
- **Enable auto-sync carefully** - Consider manual approval for production
- **Keep your Age key rotation schedule** - Plan for key rotation procedures

## Contributing

When making changes:

1. Test in a development cluster first
2. Update relevant documentation
3. Encrypt any new secrets with SOPS
4. Use meaningful commit messages
5. Verify ArgoCD can sync your changes

## License

[Specify your license here]

## Support

[Add your support/contact information here]