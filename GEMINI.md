# Project Overview

This is a GitOps repository for managing a Kubernetes cluster named "eniac". It uses FluxCD to automatically deploy and configure applications and infrastructure.

The repository is structured into three main parts:

*   `apps`: Contains the Kubernetes manifests for the applications running in the cluster.
*   `infrastructure`: Contains the manifests for the core infrastructure services.
*   `clusters`: Defines the FluxCD custom resources that point to the `apps` and `infrastructure` directories.

## Technologies

*   **Kubernetes:** The container orchestration platform.
*   **FluxCD:** The GitOps tool used to keep the cluster in sync with the repository.
*   **Kustomize:** Used to customize Kubernetes manifests for different environments.
*   **Traefik:** The ingress controller, responsible for routing external traffic to the services in the cluster.
*   **cert-manager:** Manages TLS certificates for the ingress resources.
*   **MetalLB:** Provides load balancing for the services.
*   **Prometheus, Grafana, Loki, Jaeger:** The monitoring and observability stack.
*   **SOPS:** Used to encrypt secrets in the repository.

## Applications

*   **BOINC:** A platform for volunteer computing.
*   **Grafana:** A monitoring and observability platform.
*   **Jaeger:** A distributed tracing system.
*   **Loki:** A log aggregation system.
*   **kube-prometheus-stack:** A collection of Kubernetes manifests for Prometheus and Grafana.
*   **Reloader:** A tool to automatically restart pods when their configmaps or secrets are updated.
*   **opentelemetry-operator:** Manages the OpenTelemetry Collector.

# Building and Running

This is a GitOps repository, so there is no "build" or "run" command in the traditional sense. The cluster is automatically synchronized with the contents of this repository by FluxCD.

To apply the manifests to a cluster, you would typically use `kubectl apply -k` with the appropriate kustomization directory. For example, to deploy the applications to the "eniac" cluster, you would run:

```bash
kubectl apply -k apps/eniac
```

However, since this repository is managed by FluxCD, changes are applied automatically when they are pushed to the Git repository.

# Development Conventions

*   **Kustomize:** All applications and infrastructure components are managed using Kustomize. Each component has a `base` directory with the core manifests, and environment-specific customizations are applied in separate directories.
*   **Secrets:** Secrets are encrypted using SOPS and stored in the Git repository. The `sops-age` secret in the cluster is used to decrypt the secrets.
*   **Namespaces:** Applications are deployed to the `gawbul` namespace. Infrastructure components are deployed to their own namespaces (e.g., `traefik`, `cert-manager`).
