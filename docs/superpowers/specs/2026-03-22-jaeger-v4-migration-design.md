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

- Keep **Elasticsearch** as storage backend (deployed separately via ECK operator)
- Pin Jaeger chart to **4.6.x** (chart is marked experimental)
- **Historical traces will be lost** (acceptable — index format changes between v1 and v2)
- Use **ECK (Elastic Cloud on Kubernetes)** operator to manage Elasticsearch
- Use **nfs-client** storageClass for ES data (cluster runs on Raspberry Pis with SD cards — avoid local writes)
- **Single-node** Elasticsearch deployment (sufficient for home lab tracing)

## Design

### New Infrastructure: ECK Operator

The ECK operator is deployed as an infrastructure controller (same pattern as cert-manager, MetalLB).

#### `infrastructure/base/elastic-system-namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: elastic-system
```

#### `infrastructure/controllers/eck-operator.yaml`

```yaml
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: elastic
  namespace: elastic-system
spec:
  interval: 1h
  url: https://helm.elastic.co
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: eck-operator
  namespace: elastic-system
spec:
  interval: 1h
  chart:
    spec:
      chart: eck-operator
      version: "3.x"
      sourceRef:
        kind: HelmRepository
        name: elastic
        namespace: elastic-system
      interval: 1h
  values:
    resources:
      limits:
        cpu: 500m
        memory: 512Mi
      requests:
        cpu: 100m
        memory: 150Mi
```

Update `infrastructure/base/kustomization.yaml` to add the namespace.
Update `infrastructure/controllers/kustomization.yaml` to add the operator.

### New Application: Elasticsearch CR

#### `apps/eniac/elasticsearch.yaml`

A single-node Elasticsearch managed by ECK, in the monitoring namespace.

```yaml
---
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: jaeger
  namespace: monitoring
spec:
  version: "8.17.0"
  nodeSets:
    - name: default
      count: 1
      config:
        node.store.allow_mmap: false
        cluster.routing.allocation.disk.threshold_enabled: false
        xpack.security.authc.anonymous.username: anonymous
        xpack.security.authc.anonymous.roles: superuser
        xpack.security.authc.anonymous.authz_exception: false
      podTemplate:
        spec:
          containers:
            - name: elasticsearch
              env:
                - name: ES_JAVA_OPTS
                  value: "-Xms512m -Xmx512m"
              resources:
                requests:
                  memory: "1Gi"
                  cpu: "500m"
                limits:
                  memory: "1Gi"
                  cpu: "1"
      volumeClaimTemplates:
        - metadata:
            name: elasticsearch-data
          spec:
            accessModes:
              - ReadWriteOnce
            storageClassName: nfs-client
            resources:
              requests:
                storage: 8Gi
  http:
    tls:
      selfSignedCertificate:
        disabled: true
```

Key configuration:
- `node.store.allow_mmap: false` — required on Raspberry Pi (insufficient virtual address space)
- Anonymous superuser access — avoids credential management for internal Jaeger use
- HTTP TLS disabled — plain HTTP for internal cluster communication
- 512MB heap pinned — appropriate for Pi with 4-8GB shared RAM
- nfs-client storage — avoids SD card wear

Service URL: `http://jaeger-es-http.monitoring.svc.cluster.local:9200`

Update `apps/eniac/kustomization.yaml` to add elasticsearch.yaml.

### Modified: `apps/eniac/jaeger.yaml`

The HelmRepository remains unchanged. The HelmRelease changes:

- **Chart version**: `"4.6.x"` (unchanged from previous commit)
- **Remove**: `postRenderers` (no longer need Badger volume mount)
- **Remove**: `extraObjects` (no longer need Badger PVC)
- **Change**: `esIndexCleaner.enabled: true` with 7-day retention
- **Change**: `esRollover.enabled: false`, `esLookback.enabled: false`
- **Change**: `storage` section for ES maintenance jobs
- **Change**: `userconfig.extensions.jaeger_storage` — Elasticsearch backend instead of Badger

```yaml
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: jaeger
  namespace: monitoring
spec:
  interval: 5m
  chart:
    spec:
      chart: jaeger
      version: "4.6.x"
      sourceRef:
        kind: HelmRepository
        name: jaegertracing
        namespace: monitoring
      interval: 1h
  values:
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
    storage:
      type: elasticsearch
      elasticsearch:
        host: jaeger-es-http.monitoring.svc.cluster.local
        port: 9200
        scheme: http
    esIndexCleaner:
      enabled: true
      schedule: "55 23 * * *"
      numberOfDays: 7
    esRollover:
      enabled: false
    esLookback:
      enabled: false
    userconfig:
      extensions:
        healthcheckv2:
          use_v2: true
          http: {}
        jaeger_storage:
          backends:
            primary_store:
              elasticsearch:
                server_urls:
                  - http://jaeger-es-http.monitoring.svc.cluster.local:9200
                index_prefix: jaeger
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
        extensions:
          - healthcheckv2
          - jaeger_storage
          - jaeger_query
        pipelines:
          traces:
            receivers:
              - otlp
            processors:
              - batch
            exporters:
              - jaeger_storage_exporter
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

### Modified: `apps/config/opentelemetry-collector.yaml`

Line 117: Change `http://jaeger-collector:4317` to `http://jaeger:4317` (already committed).

### Other References — No Changes Needed

- **Grafana `JAEGER_AGENT_PORT: "5775"`** (`apps/eniac/kube-prometheus-stack.yaml:65`): The unified Jaeger v2 binary still exposes port 5775 for legacy Jaeger agent protocol. No change needed.
- **OTel Collector `jaeger` receiver** (lines 56-58): Receives incoming Jaeger Thrift/gRPC traces from applications — unrelated to the backend. No change needed.

### What Gets Removed from the Cluster

- Bitnami Elasticsearch StatefulSets (master + data) and their PVCs
- Separate collector and query Deployments and Services
- References to `bitnamilegacy/*` container images
- Badger PVC (from previous commit, never deployed)

### What Gets Created

- `elastic-system` namespace with ECK operator
- Single-node Elasticsearch managed by ECK in `monitoring` namespace
- Single Jaeger Deployment running the unified `jaegertracing/jaeger` binary
- Single Jaeger Service exposing all ports (OTLP 4317/4318, query UI 16686, health 13133, agent 5775/6831/6832)
- ConfigMap with the `userconfig` OTel Collector-format config
- ES index cleaner CronJob (7-day retention)

### Deployment Order

FluxCD infrastructure deploys before apps:
1. Infrastructure controllers deploy ECK operator (installs CRDs)
2. Apps deploy Elasticsearch CR and Jaeger HelmRelease
3. Jaeger will retry connecting to ES until it's ready (standard FluxCD reconciliation)

### Risks

| Risk | Severity | Mitigation |
|------|----------|------------|
| Jaeger chart marked "experimental" | Low | Pinned to 4.6.x |
| Historical traces lost | Accepted | User confirmed acceptable |
| `userconfig` overrides entire default config | Medium | Config includes all required sections |
| ECK operator adds cluster-wide CRDs | Low | Standard operator pattern, same as cert-manager |
| Elasticsearch on NFS performance | Low | Acceptable for home lab tracing workload |
| Pi memory constraints (ES + Jaeger) | Low | ES heap pinned at 512MB, Pi has 4-8GB shared |
| Anonymous superuser on ES | Low | Internal-only, no external exposure |
