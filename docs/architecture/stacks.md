# Stacks

## Model

Three things that used to be one:

| Axis | What it is | How many |
| ---- | ---------- | -------- |
| Folder | code layout under `stacks/` | few |
| Namespace | quota, RBAC, blast radius | one per domain |
| Deploy unit | what you actually toggle | a profile, composed |

A folder is not a toggle. Toggling happens by profile, so adding a tool usually means one more
release in a folder that already exists — not a new folder.

## Tiers

Grouped by lifecycle, not by function.

| Tier | Stacks | Rule |
| ---- | ------ | ---- |
| 0 | `platform`, `observability` | CRDs and node-level drivers must always exist. Never toggled. |
| 1 | `core` | Holds state. Tearing it down loses data. Never toggled. |
| 2 | `workloads/*` | Stateless or rebuildable. Toggle freely. |

Operators live in tier 0, their custom resources in tier 2. Strimzi is tier 0; the Kafka cluster
it manages is tier 2.

## Map

| Folder | NS | Tier | What's in it |
| ------ | -- | ---- | ------------ |
| `platform` | `platform` | 0 | Traefik, namespaces + ResourceQuota + LimitRange, RBAC, snapshot-controller, and every operator: CNPG, Spark, Strimzi, Flink, StarRocks |
| `observability` | `observability` | 0 | VictoriaMetrics k8s-stack, Grafana, VictoriaLogs |
| `core` | `core` | 1 | MinIO, CNPG `Cluster`, Polaris |
| `workloads/orchestration` | `orchestration` | 2 | Airflow |
| `workloads/compute` | `compute` | 2 | Trino, StarRocks, Spark driver RBAC + demo `SparkApplication`s |
| `workloads/stream` | `stream` | 2 | Kafka (Strimzi CR), kafka-ui, Kafka Connect |
| `workloads/ml` | `ml` | 2 | MLflow, Kubeflow Pipelines (standalone) |
| `workloads/governance` | `governance` | 2 | OpenMetadata, Superset |

Implemented today: `platform` and `core`. The rest get folders as they land.

Every folder holds `helmfile.yaml`, `values/`, `manifests/`.

## Profiles

Tier 0 is always up. A profile picks which tier-2 stacks join `core`.

**Planned, not wired yet** — each profile gets its mise task once every stack it composes exists.

| Profile | Stacks | For |
| ------- | ------ | --- |
| `demo:batch` | core + orchestration + compute | Airflow → Spark → Iceberg → Trino |
| `demo:streaming` | core + stream + compute | Kafka → Flink → Iceberg |
| `demo:mlops` | core + orchestration + ml | training runs, MLflow tracking |
| `demo:governance` | core + governance | OpenMetadata ingestion, Superset |

Profiles are mise tasks that compose stack helmfiles in order. To select a subset inside one
stack, use helmfile labels. Helmfile allows a single value per label key, so a release belonging
to several profiles wants boolean keys (`batch: "true"`, `mlops: "true"`) rather than
`profile: batch`.

## Dependencies

```
platform                     tier 0 — everything needs the CRDs
    ├── observability        tier 0
    └── core                 tier 1 — MinIO, PostgreSQL, Polaris
            ├── orchestration
            ├── compute      Polaris is required to read Iceberg
            ├── ml           MLflow needs PostgreSQL + MinIO
            ├── governance   OpenMetadata needs PostgreSQL
            └── stream       no hard dep on core; competes for RAM
```

Inside `core`, helmfile `needs:` order:

```
MinIO → PostgreSQL → Polaris
```

`needs:` only resolves **within one state file**. Sub-helmfiles listed under `helmfiles:` run in
listed order with no dependency graph between them, so ordering across stacks is the caller's
job. That is what the profile tasks are for.

## Up / down

```
up:   platform → observability → core → (tier 2, any order)
down: reverse — dependents first
```

`mise tasks ls` is the authoritative list — a stack gets its up/down pair when its helmfile lands,
so what you see there is what exists.

| Command | What happens |
| ------- | ------------ |
| `mise run bootstrap` | helm-diff plugin |
| `mise run platform:up` | tier 0 — run first, leave it up |
| `mise run core:up` | tier 1 — MinIO, PostgreSQL, Polaris |
| `mise run orchestration:up` | tier 2 — Airflow; needs `core` up |
| `mise run <stack>:down` | `helmfile destroy`; namespace and PVCs stay |
| `mise run <stack>:down --purge` | destroy, then delete the namespace — PVCs go with it |
| `mise run all:down` | every tier-2 stack, reverse order. Tier 0 and `core` stay up |
| `mise run diff` / `drift` / `status` | inspect before and after |
| `mise run secrets:show <name>` | print a generated credential |

`core:down --purge` deletes the lakehouse — MinIO objects and the Postgres PVC. Nothing in this
repo backs them up. Use it when you mean it.

Check pods after each step: `kubectl get pods -n <ns>`.

Habits: `mise run diff` before apply, `helmfile template` when editing values, pin every chart
version.

## Memory budget (~27 GiB allocatable)

Fresh k3s after reinstall was **931 MiB** (2026-07-20, measured). Everything else is an estimate
until the stack lands.

| Layer | GiB | |
| ----- | --- | - |
| k3s | ~0.9 | measured |
| platform | ~1.0 | Traefik plus five operators |
| observability | ~1.5 | |
| **tier 0 resident** | **~3.5** | |
| core | ~4 | MinIO + PostgreSQL + Polaris, idle |
| orchestration | ~2 | scheduler + triggerer + webserver |
| compute | ~1.5 | Trino coordinator + one worker, before Spark jobs |
| stream | ~1.5 | single-broker KRaft Kafka |
| ml | ~3 | MLflow is small; KFP standalone is most of it |
| governance | ~5 | OpenMetadata + OpenSearch |

Controllers are not what fills the node. CNPG runs around 100 MiB, Spark 200, Strimzi 350, Flink
300 — roughly 1 GiB for the whole set. Data planes are the expensive part, which is why tier 0
stays resident and tier 2 toggles.

## Where to put something new

1. Installs CRDs, runs as a DaemonSet, or watches cluster-wide → `platform`
2. Holds state you would be sad to lose → `core`
3. Sits on the critical path of every pipeline (object store, catalog) → `core`
4. Anything else → a `workloads/` namespace, labelled with the profiles that need it
5. A spec, library, or CLI rather than a service → not this repo, see below

Polaris and OpenMetadata are both called catalogs and land in different tiers. Polaris is a
technical catalog on the query path, so `core`. OpenMetadata is a discovery catalog nothing
depends on at runtime, so tier 2.

## Gotchas

**One StorageClass: `local-path`, RWO only.** A chart defaulting to `ReadWriteMany` leaves its PVC
Pending forever and the pod never starts. Override `accessModes` to `ReadWriteOnce`. On a single
node that costs nothing — RWO restricts a volume to one *node*, not one pod, so co-located pods
still share it.

**Credentials come from `.env`, not from the charts.** `mise run secrets:sync` reads `.env`
(gitignored) and writes the Secrets each namespace needs — `core` and `orchestration` both get
`postgres-app` and `minio-root`, `orchestration` also gets the Airflow accounts. The charts adopt
them by name: CNPG via `initdb.secret.name`, MinIO via `existingSecret`, Airflow's create-user job
via env. `core:up` and `orchestration:up` run `secrets:sync` first, so the Secrets exist before
the charts that need them. No password is in the repo or in a rendered manifest.

Two things to know:

- **The PostgreSQL username is fixed to `app`**, not set in `.env`. It has to equal the database
  owner CNPG creates (`initdb.owner: app`); a mismatch bootstraps a role nobody can log in as.
- **Changing the postgres password needs a core recreate.** `initdb` only runs on first bootstrap,
  so editing `.env` and re-syncing updates the Secret but not the live role. `core:down` (which
  garbage-collects the PVC) then `core:up` re-bootstraps with the new password. MinIO and Airflow
  pick up a changed password on their next `:up` — only postgres needs the recreate.

**Airflow memory.** Scheduler growth is usually the DAG parsing loop rather than a leak in
Airflow. Run the standalone dag-processor so parsing is isolated and restartable, raise
`min_file_process_interval` well above its 30s default, and set explicit memory limits on
scheduler and triggerer. On a single node an unbounded leak evicts its neighbours.

**KFP standalone ships its own MinIO and MySQL.** Override both to `core` or you end up with a
second object store.

**Kafka PVCs survive `stream:down`.** Purge unless you want the topics back.

**OpenMetadata needs OpenSearch.** Point its metadata DB at the `core` CNPG cluster; only
OpenSearch has to be new.

**Don't write logs or artifacts into the warehouse bucket.** MinIO serves three, named in
`environments/<env>.yaml` as `minio.warehouseBucket`, `logsBucket` and `artifactsBucket`. Quotas
and expiry rules are per-bucket with no prefix equivalent, so a shared bucket means runaway task
logs fill the one PVC, MinIO goes read-only, and the warehouse stops accepting writes. Split by
lifecycle only — raw/refined/mart remain prefixes inside the warehouse.

**Addresses and bucket names live in `environments/<env>.yaml`, not in values files.** A values
file describes how a fact reaches a chart; the fact itself is declared once. `endpoints.postgres`
is read by Polaris, Airflow and later MLflow — three hardcoded copies would be three chances to
miss one.

## Not in this repo

| Thing | Why | Where |
| ----- | --- | ----- |
| OpenLineage | A spec and client libraries, not a service. Configured in the producers — Airflow's `openlineage` provider, Spark's listener jar | data-platform |
| dbt | Workload | data-platform |
| Elementary | A dbt package plus a CLI that emits a static report; nothing stays running | data-platform |

If a lineage backend is ever needed, check whether the deployed OpenMetadata version already
accepts OpenLineage events before adding Marquez — otherwise it is a second copy of the same
graph.
