# Stacks

Each stack is a namespace and a folder under `stacks/<name>/` with its own `helmfile.yaml`.

## Map

| Stack | NS | What's in it | Mode |
|-------|-----|--------------|------|
| platform | `platform` | Traefik, CNPG operator, NS + ResourceQuota | always on |
| observability | `observability` | VictoriaMetrics k8s-stack, Grafana | always on |
| data | `data` | MinIO, PostgreSQL, Polaris, Airflow | toggle |
| spark | `spark` | Spark operator, driver RBAC, smoke CR | toggle |
| storage | `storage` | JuiceFS CSI, gateway | toggle |
| query | `query` | Trino (+ StarRocks) | toggle |
| stream | `stream` | Strimzi, Kafka, kafka-ui, Flink operator | toggle |
| ml | `ml` | MLflow, Kubeflow Pipelines (standalone) | toggle |
| bi | `bi` | Superset, Elementary | toggle |
| metadata | `metadata` | OpenMetadata | toggle |

Build order for the first four: `platform → obs → data → spark` (see [Up / down](#up--down)). Later-phase stacks get folders when implemented.

## Dependencies

```
platform
    ├── observability
    ├── data  ──┬── storage
    │           ├── query
    │           ├── spark (smoke / E2E need data up)
    │           └── ml
    ├── stream   (no hard deps on data; competes for RAM)
    ├── bi
    └── metadata
```

Inside `data`, helmfile `needs:` order:

```
MinIO → PostgreSQL → Polaris → Airflow
```

Airflow uses gitSync + MinIO for logs — not JuiceFS.

## Up / down

```
up:   platform → obs → de → (storage | spark | query | …)
down: reverse — dependents first
```

`all:down` hits toggle stacks only, in reverse order. Platform and observability stay up.

| Command | What happens |
|---------|----------------|
| `de:down` | helmfile destroy; namespace + PVCs stay |
| `de:down --purge` | destroy, then delete `data` namespace (PVCs gone too) |

First bring-up (once `mise` stack tasks land): `platform:up → obs:up → de:up → spark:up`. Check pods after each step. Run `mise run diff` before apply.

## Memory (~27 GiB allocatable)

Rough numbers from the old cluster and estimates:

| Layer | GiB |
|-------|-----|
| k3s + platform + obs | ~2.5 always |
| data idle | ~4–5 |
| storage | ~0.9 |
| spark | ~0.2 operator + jobs |

Fresh k3s after reinstall was **~931 MiB** (2026-07-20). With data + storage up you're around ~8 GiB before Spark jobs.

## Where to put something new

- Cluster-wide operator, ingress, CSI → `platform`
- Always tied to one stack → add a release there
- Separate toggle, especially if it's 1 GiB+ → new stack folder

dbt stays in [data-platform](https://github.com/revelly-io/data-platform). StarRocks → `query`. Superset → `bi`.
