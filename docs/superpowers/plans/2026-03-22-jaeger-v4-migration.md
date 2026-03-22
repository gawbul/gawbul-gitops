# Jaeger v4 Migration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Migrate Jaeger from Helm chart v3 (Jaeger v1, Elasticsearch) to v4 (Jaeger v2, Badger storage).

**Architecture:** Replace the multi-component Jaeger v1 deployment with a single unified binary running on the OTel Collector framework. Storage switches from an Elasticsearch subchart to embedded Badger with local-path persistence. The chart's lack of native persistence is handled via `extraObjects` (PVC) and `postRenderers` (volume mount injection).

**Tech Stack:** FluxCD HelmRelease, Jaeger v2, Badger, Kustomize postRenderers

**Spec:** `docs/superpowers/specs/2026-03-22-jaeger-v4-migration-design.md`

---

### Task 1: Rewrite Jaeger HelmRelease

**Files:**
- Modify: `apps/eniac/jaeger.yaml` (full rewrite of HelmRelease values, HelmRepository unchanged)

- [ ] **Step 1: Replace HelmRelease values**

Replace the entire HelmRelease resource in `apps/eniac/jaeger.yaml` (lines 11-72). Keep the HelmRepository (lines 1-9) unchanged. The new HelmRelease:

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
    esIndexCleaner:
      enabled: false
    esRollover:
      enabled: false
    esLookback:
      enabled: false
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

- [ ] **Step 2: Validate the YAML syntax**

Run: `python3 -c "import yaml; yaml.safe_load(open('apps/eniac/jaeger.yaml'))" && echo "YAML valid"`
Expected: `YAML valid`

- [ ] **Step 3: Commit**

```bash
git add apps/eniac/jaeger.yaml
git commit -m "feat: migrate jaeger helmrelease to v4 with badger storage"
```

---

### Task 2: Update OTel Collector endpoint

**Files:**
- Modify: `apps/config/opentelemetry-collector.yaml:117`

- [ ] **Step 1: Change the Jaeger exporter endpoint**

In `apps/config/opentelemetry-collector.yaml`, line 117, change:
```yaml
        endpoint: http://jaeger-collector:4317
```
to:
```yaml
        endpoint: http://jaeger:4317
```

The unified Jaeger v2 service is named `jaeger` (the HelmRelease name), not `jaeger-collector`.

- [ ] **Step 2: Validate the YAML syntax**

Run: `python3 -c "import yaml; yaml.safe_load(open('apps/config/opentelemetry-collector.yaml'))" && echo "YAML valid"`
Expected: `YAML valid`

- [ ] **Step 3: Commit**

```bash
git add apps/config/opentelemetry-collector.yaml
git commit -m "fix: update otel collector jaeger endpoint for v2 unified service"
```

---

### Task 3: Push and create PR

- [ ] **Step 1: Push branch**

```bash
git push -u origin chore/jaeger-v4-migration
```

- [ ] **Step 2: Create PR**

```bash
gh pr create --title "feat: migrate Jaeger to v4 with Badger storage" --body "$(cat <<'EOF'
## Summary
- Migrate Jaeger Helm chart from v3 (Jaeger v1) to v4.6.x (Jaeger v2)
- Replace Elasticsearch storage with embedded Badger (local-path PVC)
- Unified binary replaces separate collector/query/agent components
- Update OTel Collector endpoint from `jaeger-collector` to `jaeger`
- Uses `postRenderers` to inject Badger PVC volume mount (chart lacks native persistence)

## Breaking Changes
- Historical traces in Elasticsearch will be lost (accepted)
- Elasticsearch StatefulSets will be removed from cluster

## Test plan
- [ ] Verify FluxCD reconciles the Jaeger HelmRelease (`flux get helmreleases -n monitoring`)
- [ ] Confirm Jaeger pod starts and is healthy
- [ ] Verify Jaeger query UI is accessible at `monitoring.home.gawbul.io/jaeger`
- [ ] Confirm OTLP trace ingestion works (check OTel Collector logs for export errors)
- [ ] Verify Badger PVC is bound and data persists across pod restarts
- [ ] Confirm Grafana can still send traces via Jaeger agent port 5775

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

---

### Post-Deployment Verification

After FluxCD reconciles (check with `flux get helmreleases -n monitoring`):

1. **Jaeger pod health**: `kubectl get pods -n monitoring -l app.kubernetes.io/name=jaeger`
2. **PVC bound**: `kubectl get pvc -n monitoring jaeger-badger-data`
3. **Query UI**: Browse to `https://monitoring.home.gawbul.io/jaeger`
4. **OTel Collector**: `kubectl logs -n monitoring -l app.kubernetes.io/name=otel-collector --tail=50 | grep -i jaeger`
5. **Elasticsearch cleanup**: Old ES StatefulSets and PVCs should be gone — verify with `kubectl get statefulsets,pvc -n monitoring | grep elasticsearch`
