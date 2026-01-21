# Project Context: gawbul-gitops

## Overview
This is a GitOps repository managing a home lab Kubernetes cluster named "eniac". It utilizes **Flux CD** to automatically synchronize the cluster state with this Git repository. The project follows a structured directory layout separating infrastructure from applications, enforcing dependency chains for reliable deployments.

## Architecture & Directory Structure

The repository is organized into three main areas:

*   **`clusters/`**: The entry point for Flux.
    *   `eniac/`: Configuration for the specific cluster.
        *   `flux-system/`: Core Flux components.
        *   `infrastructure.yaml`: Kustomization definitions for infrastructure layers with dependencies.
        *   `app.yaml`: Kustomization definitions for application layers.

*   **`infrastructure/`**: Core services required for the cluster to function.
    *   `controllers/`: Fundamental operators (Cert-Manager, MetalLB).
    *   `configs/`: Configurations and secrets (ClusterIssuers, MetalLB IP pools).
    *   `ingress/`: Ingress controllers (Traefik).
    *   `notifications/`: Alerting and notifications (Discord).
    *   `base/`: Shared infrastructure resources (Namespaces).

*   **`apps/`**: User-facing applications and services.
    *   `base/`: Common application resources.
    *   `eniac/`: Cluster-specific HelmReleases and GitRepositories.
    *   `config/`: Application configurations (OpenTelemetry, Secrets).

## Key Technologies

*   **Flux CD**: The GitOps engine. Uses `Kustomization`, `HelmRelease`, and `GitRepository` CRDs.
*   **Kustomize**: Used for manifest generation and patching.
*   **Helm**: Package management, driven by Flux HelmReleases.
*   **SOPS**: Secret management. Secrets are encrypted in the repo and decrypted by Flux using AGE keys.
*   **Pre-commit**: Enforces checks like YAML validation and private key detection.

## Development Workflows

### 1. Adding Infrastructure
1.  Create a `HelmRelease` or manifest in the appropriate `infrastructure/` subdirectory.
2.  Register it in the local `kustomization.yaml`.
3.  Ensure it aligns with the dependency order: `controllers` -> `configs` -> `ingress` -> `notifications`.

### 2. Adding Applications
1.  Create a `HelmRelease` in `apps/eniac/`.
2.  Add the resource to `apps/eniac/kustomization.yaml`.
3.  If configuration/secrets are needed, add them to `apps/config/` (encrypted with SOPS if sensitive).

### 3. Managing Secrets
*   **Never commit plain text secrets.**
*   Use `sops` with AGE to encrypt secrets.
*   Flux is configured with `decryption.provider: sops` for specific Kustomizations (e.g., `infra-configs`, `apps-config`).
*   **Encrypt:** `sops -e -i path/to/secret.yaml`
*   **Decrypt:** `sops -d path/to/secret.yaml`

## Common Commands

### Validation
```bash
# Validate manifests (client-side dry-run)
kubectl apply --dry-run=client -k apps/eniac
kubectl apply --dry-run=client -k infrastructure/controllers

# Run pre-commit checks manually
pre-commit run --all-files
```

### Flux Status
```bash
flux get kustomizations
flux get helmreleases --all-namespaces
flux get sources git
```

## Conventions & Rules

*   **GitOps Principle**: Do not manually `kubectl apply` changes to the cluster. Commit to Git and let Flux sync.
*   **Namespaces**:
    *   `gawbul`: Default application namespace.
    *   `flux-system`: Flux components.
    *   `monitoring`: Observability stack.
    *   `traefik`, `cert-manager`, `metallb-system`: Infrastructure components.
*   **Dependencies**: Respect the `dependsOn` fields in `clusters/eniac/*.yaml` files.
*   **Patching**: Use Kustomize patches in `clusters/eniac/` for environment-specific overrides (e.g., Let's Encrypt servers).
