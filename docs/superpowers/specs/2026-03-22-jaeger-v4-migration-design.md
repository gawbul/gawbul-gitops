# Jaeger v4 Migration Design

## Context

The Jaeger Helm chart is currently pinned to `3.x` (Jaeger v1), which deploys separate collector, query, and agent components with a Bitnami Elasticsearch subchart for storage. The Jaeger Helm chart v4 represents a complete architectural overhaul — Jaeger v2 uses a single unified binary built on the OpenTelemetry Collector framework.

### Current State

- **Chart**: `jaeger` v3.x from `https://jaegertracing.github.io/helm-charts`
- **Storage**: Elasticsearch via Bitnami subchart (nfs-client, 8Gi per node)
- **Collector**: Separate deployment exposing OTLP gRPC (4317) and HTTP (4318)
- **Query UI**: Separate deployment at basePath `/jaeger`, ingress on `monitoring.home.gawbul.io`
- **OTel Collector**: Sends traces to `http://jaeger-collector:4317`
- **Grafana**: Configured with `JAEGER_AGENT_PORT: "5775"` for sending its own traces

### Decisions

- Switch storage from Elasticsearch to **Badger** (embedded, no external dependency)
- Pin chart to **4.6.x** (chart is marked experimental)
- **Historical traces will be lost** (acceptable)
- Use **local-path** storageClass for Badger (not nfs-client — Badger uses `flock` which is unreliable on NFS)

## Design

### Files Modified

#### 1. `apps/eniac/jaeger.yaml` — Full Rewrite

The HelmRepository remains unchanged. The HelmRelease values are completely replaced.

**Chart version**: `"4.6.x"`

**New values structure**:

```yaml
jaeger:
  ingress:
    enabled: true
    ingressClassName: traefik
    pathType: Prefix
    path: /jaeger
    hosts:
      - monitoring.home.gawbul.io
    tls:
      - secretName: monitoring-tls
        hosts:
          - monitoring.home.gawbul.io

# Disable ES maintenance jobs (not using Elasticsearch)
esIndexCleaner:
  enabled: false
esRollover:
  enabled: false
esLookback:
  enabled: false

# PVC for Badger data (chart has no native persistence support)
extraObjects:
  - apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: jaeger-badger-data
      namespace: monitoring
    spec:
      storageClassName: local-path
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 8Gi

userconfig:
  extensions:
    healthcheckv2:
      use_v2: true
      http: {}
    jaeger_storage:
      backends:
        primary_store:
          badger:
            directories:
              keys: /badger/data/keys
              values: /badger/data/values
            maintenance_interval: 5m
    jaeger_query:
      base_path: /jaeger
      storage:
        traces: primary_store

  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: 0.0.0.0:4317
        http:
          endpoint: 0.0.0.0:4318

  processors:
    batch: {}

  exporters:
    jaeger_storage_exporter:
      trace_storage: primary_store

  service:
    extensions: [healthcheckv2, jaeger_storage, jaeger_query]
    pipelines:
      traces:
        receivers: [otlp]
        processors: [batch]
        exporters: [jaeger_storage_exporter]
    telemetry:
      metrics:
        level: detailed
        readers:
          - pull:
              exporter:
                prometheus:
                  host: 0.0.0.0
                  port: 8888
```

**Mounting the PVC**: The chart does not support `extraVolumes`/`extraVolumeMounts`. A FluxCD `postRenderers` Kustomize patch is used on the HelmRelease to inject the volume and volumeMount into the rendered Deployment:

```yaml
spec:
  postRenderers:
    - kustomize:
        patches:
          - target:
              kind: Deployment
              name: jaeger
            patch: |
              - op: add
                path: /spec/template/spec/volumes/-
                value:
                  name: badger-data
                  persistentVolumeClaim:
                    claimName: jaeger-badger-data
              - op: add
                path: /spec/template/spec/containers/0/volumeMounts/-
                value:
                  name: badger-data
                  mountPath: /badger/data
```

**Removed sections**: `provisionDataStore`, `storage`, `elasticsearch`, `collector`, `query`.

#### 2. `apps/config/opentelemetry-collector.yaml` — Endpoint Rename

Line 117: Change `http://jaeger-collector:4317` to `http://jaeger:4317`.

The unified Jaeger v2 service is named after the HelmRelease (`jaeger`) rather than having component-specific service names.

**Note**: The `jaeger` receiver (lines 56-58) in the OTel Collector config is unrelated to the Jaeger backend — it receives incoming traces in Jaeger Thrift/gRPC format from applications that use the legacy Jaeger SDK. This receiver remains valid and unchanged.

### Other References — No Changes Needed

- **Grafana `JAEGER_AGENT_PORT: "5775"`** (`apps/eniac/kube-prometheus-stack.yaml:65`): The unified Jaeger v2 binary still exposes port 5775 for legacy Jaeger agent protocol. No change needed.
- **Current `tls` config bug**: The existing jaeger.yaml has `tls[0].host` (singular) instead of `hosts` (plural). The new config corrects this to use the standard `hosts` field.

### What Gets Removed from the Cluster

After FluxCD reconciles:

- Bitnami Elasticsearch StatefulSets (master + data) and their PVCs
- Separate collector and query Deployments
- Separate collector and query Services
- References to `bitnamilegacy/*` container images

### What Gets Created

- Single Jaeger Deployment running the unified `jaegertracing/jaeger` binary
- Single Service exposing all ports (OTLP 4317/4318, query UI 16686, health 13133, agent 5775/6831/6832)
- PVC for Badger data (8Gi, local-path) via `extraObjects`
- Volume mount patched in via FluxCD `postRenderers`
- ConfigMap with the `userconfig` OTel Collector-format config

### Risks

| Risk | Severity | Mitigation |
|------|----------|------------|
| Chart marked "experimental" | Low | Pinned to 4.6.x |
| Historical traces lost | Accepted | User confirmed acceptable |
| `userconfig` overrides entire default config | Medium | Config includes all required sections (health check, receivers, exporters, telemetry) |
| `postRenderers` patch fragility | Low | Targets specific Deployment by name; will fail visibly if chart changes structure |
| OTel Collector endpoint change | Low | Single line change, clear dependency |
| Badger data on local-path (node-local) | Low | Traces are operational data, not critical; node failure loses history which is acceptable |

### Deployment Order

No special ordering needed — both files are in the apps Kustomization which FluxCD reconciles together. The brief window where the OTel Collector references the old service name will result in temporary trace export failures until both resources reconcile, which is acceptable.
