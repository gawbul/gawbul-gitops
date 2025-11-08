# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a GitOps repository for managing a Kubernetes cluster named "eniac" using FluxCD for continuous deployment. The repository follows a structured approach with automatic synchronization between Git and the cluster state.

## Architecture

### Directory Structure

- `clusters/eniac/`: FluxCD configuration for the eniac cluster
  - `flux-system/`: Core FluxCD components and Git sync configuration
  - `infrastructure.yaml`: Defines infrastructure deployment pipeline
  - `app.yaml`: Defines application deployment pipeline

- `infrastructure/`: Core infrastructure services deployed in order:
  1. `controllers/`: Infrastructure controllers (cert-manager, MetalLB)
  2. `configs/`: Configuration resources (TLS secrets, MetalLB IP pools, ClusterIssuers)
  3. `ingress/`: Ingress controller (Traefik)
  4. `notifications/`: Flux notification providers (Discord alerts)
  5. `base/`: Common infrastructure resources (namespaces)

- `apps/`: Application deployments
  - `base/`: Common application resources (namespaces, base configurations)
  - `eniac/`: Cluster-specific application HelmReleases and GitRepositories
  - `config/`: Application configurations (OpenTelemetry collector, instrumentation)

### Deployment Pipeline

FluxCD orchestrates deployments with dependencies:
1. Infrastructure controllers → Infrastructure configs → Ingress → Notifications
2. Applications depend on infrastructure notifications being ready
3. Application configs deployed after applications

### Key Technologies

- **FluxCD**: GitOps operator (Kustomization, HelmRelease, GitRepository resources)
- **Kustomize**: Manifest customization with base/overlay pattern
- **Helm**: Package management via FluxCD HelmRelease resources
- **SOPS**: Secret encryption (decryption provider configured in FluxCD Kustomizations)
- **Traefik**: Ingress controller with OTLP metrics/tracing
- **cert-manager**: TLS certificate automation with Let's Encrypt
- **MetalLB**: LoadBalancer IP allocation (192.168.50.204-254)
- **Observability Stack**: Prometheus, Grafana, Loki, Jaeger, OpenTelemetry

## Common Commands

### Validate Manifests
```bash
# Validate all Kubernetes manifests
kubectl apply --dry-run=client -k apps/eniac
kubectl apply --dry-run=client -k infrastructure/controllers

# Validate specific kustomization
kubectl apply --dry-run=client -k apps/config
```

### Check FluxCD Status
```bash
# Check Flux Kustomizations
flux get kustomizations

# Check Helm releases
flux get helmreleases --all-namespaces

# Check Git repositories
flux get sources git
```

### Manual Apply (Not Recommended)
FluxCD automatically syncs the repository. Manual applies are rarely needed:
```bash
# Apply infrastructure
kubectl apply -k infrastructure/controllers
kubectl apply -k infrastructure/configs
kubectl apply -k infrastructure/ingress

# Apply applications
kubectl apply -k apps/eniac
kubectl apply -k apps/config
```

### SOPS Secret Management
```bash
# Decrypt a secret (requires SOPS and AGE key)
sops -d infrastructure/configs/discord-secret.yaml

# Encrypt a secret
sops -e -i infrastructure/configs/new-secret.yaml
```

### Pre-commit Hooks
```bash
# Install pre-commit hooks
pre-commit install

# Run manually
pre-commit run --all-files
```

## Development Workflow

### Adding New Applications

1. Create HelmRelease or GitRepository in `apps/eniac/`
2. Add resource to `apps/eniac/kustomization.yaml`
3. If secrets needed, create encrypted SOPS file in `apps/config/`
4. Commit and push - FluxCD will apply automatically

### Adding Infrastructure Components

1. Create HelmRelease in appropriate `infrastructure/` subdirectory
2. Add to corresponding kustomization.yaml
3. Respect dependency order: controllers → configs → ingress → notifications
4. Use patches in `clusters/eniac/infrastructure.yaml` for environment-specific overrides

### Secrets Management

- All secrets encrypted with SOPS using AGE
- FluxCD Kustomizations with secrets have `decryption.provider: sops` configured
- Decryption key stored in `sops-age` secret in flux-system namespace
- Never commit unencrypted secrets

### Namespace Conventions

- Applications: `gawbul` namespace (default) or application-specific
- Infrastructure: Component-specific namespaces (traefik, cert-manager, metallb-system)
- Monitoring: `monitoring` namespace for observability stack

## Important Notes

- **GitOps Philosophy**: Changes should be committed to Git, not applied with kubectl
- **FluxCD Reconciliation**: Default interval is 1h for infrastructure, 10m for apps
- **Dependencies**: Respect the deployment order defined in infrastructure.yaml and app.yaml
- **Patches**: Environment-specific changes use Kustomize patches in cluster-level files
- **HelmRelease Versions**: Use semantic versioning constraints (e.g., "1.x" for cert-manager, "*" for latest)
