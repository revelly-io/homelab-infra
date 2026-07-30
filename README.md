# homelab-infra

Helmfile-managed k8s stacks for a data platform (MinIO, PostgreSQL, Polaris, Airflow, Spark, Trino, Kafka). An always-on base stays up; workload stacks toggle with mise — `core:up`, `orchestration:down`, etc.

Workloads (DAGs, Spark apps, dbt) live in [data-platform](https://github.com/revelly-io/data-platform). **This repo is cluster infrastructure only.**

Designed for a single-node k3s cluster (~27 GiB allocatable). Traefik is disabled at k3s install; ingress comes from the platform stack.

## Two ways to run

| Goal | Repo | How |
|------|------|-----|
| Spark + Iceberg on your **Mac** (Docker, no k8s) | [data-platform](https://github.com/revelly-io/data-platform) | `mise run infra:up`, then `--env local` |
| Full stack on **your k8s cluster** | homelab-infra + data-platform | steps below; same `--env` name in both repos |

## What works today

```bash
mise trust
mise install   # helm, helmfile, kubectl, jq
export KUBECONFIG=/path/to/your/kubeconfig
cp environments/example.yaml environments/homelab.yaml   # pick any name
# edit environments/homelab.yaml for your cluster
```

The `platform` and `core` stacks, the root helmfile and the mise tasks are in the repo. `observability` and the remaining `workloads/*` stacks are not yet — see [stacks.md](docs/architecture/stacks.md) for what exists today.

## `--env` ties both repos together

Pick one name (`homelab`, `dev`, `lab`, …). Use the same string everywhere:

| | |
|--|--|
| homelab-infra | `environments/<env>.yaml`, `mise run core:up --env <env>` → `helmfile -e <env>` |
| data-platform | `--env <env>`, `spark_app/config/<env>/`, `.env.<env>` |
| MinIO | `minio.warehouseBucket` → data-platform warehouse `s3a://<name>`; `logsBucket` and `artifactsBucket` created alongside |

Example (`homelab`):

```
mise run core:up --env homelab        (homelab-infra)
        │
        ├─► helmfile -e homelab
        │       └─► environments/homelab.yaml
        │
        └─► cluster  (MinIO bucket "homelab", …)

mise run spark-app --env homelab …    (data-platform)
        │
        ├─► spark_app/config/homelab/global-config.yaml
        ├─► .env.homelab
        └─► warehouse s3a://homelab
```

## Bring up on your cluster

### 1. Kubernetes

Bring your own cluster (k3s, k3d, kind, bare metal). You need `kubectl` via kubeconfig — nothing cluster-specific is committed here.

Recommended: k3s with `--disable traefik` (Traefik comes from the platform stack).

### 2. Environment file

Edit `environments/<env>.yaml` — at minimum `env`, `cluster.storageClass`, `ingress.hostSuffix`, `minio.bucket` (= `env`), and `sizing`. See [environments/example.yaml](environments/example.yaml).

Private env files are gitignored; only `example.yaml` is committed.

### 3. Credentials

Passwords live in `.env` (gitignored) — nothing is committed, no encryption key to keep. `mise run secrets:sync` turns `.env` into Kubernetes Secrets that the charts read by name; `core:up` and `orchestration:up` run it first.

```bash
cp .env.example .env      # then fill in the passwords
```

| Secret | Set from `.env` | Namespaces | Used by |
|--------|-----------------|------------|---------|
| `postgres-app` | `POSTGRES_APP_PASSWORD` (user fixed to `app`) | `core`, `orchestration` | Polaris, Airflow, later MLflow |
| `minio-root` | `MINIO_ROOT_USER` / `_PASSWORD` | `core`, `orchestration` | MinIO, Airflow (remote logs) |
| `airflow-admin` / `airflow-viewer` | `AIRFLOW_ADMIN_*` / `AIRFLOW_VIEWER_*` | `orchestration` | Airflow web UI (admin + read-only) |

The true PostgreSQL superuser (`postgres`) keeps a random CNPG-generated password, separate from `.env`. Read any secret with `mise run secrets:show <name>` (add `-n <namespace>` if not `core`).

Changing a password: edit `.env`, `mise run secrets:sync`, then re-run the consuming stack's `:up`. PostgreSQL is the exception — its role password is only set at first bootstrap, so `core:down` then `core:up` is needed.

### 4. Deploy stacks

```bash
mise run bootstrap
mise run platform:up --env homelab        # always on — quotas, Traefik, operators
mise run core:up --env homelab            # always on — MinIO, PostgreSQL, Polaris
mise run orchestration:up --env homelab   # toggled — Airflow
```

`mise tasks ls` lists what exists. Stacks that are designed but not written yet have no task.

Tiers, profiles, tear-down, `--purge`, deps, and memory: [docs/architecture/stacks.md](docs/architecture/stacks.md).

### 5. Wire data-platform *(after MVP)*

In [data-platform](https://github.com/revelly-io/data-platform), use the same `--env`. For `homelab`, edit the bundled `spark_app/config/homelab/` and create `.env.homelab` (use `.env.local.example` as a structural reference for k8s endpoints).

```bash
mise run spark-app --app_name sample.orders_summary --env homelab --ymd 2026-07-14 --hms 100000
```

Different env name? Copy `spark_app/config/homelab/` → `spark_app/config/<name>/` and add `.env.<name>`.

## Layout

```
homelab-infra/
├── mise.toml
├── helmfile.yaml                 # (MVP)
├── environments/example.yaml
├── stacks/
│   ├── platform/                 # always on — ingress, quotas, every operator
│   ├── observability/            # always on
│   ├── core/                     # always on — MinIO, PostgreSQL, Polaris
│   └── workloads/{orchestration,compute,stream,ml,governance}/   # toggled
└── docs/architecture/stacks.md
```

Each stack folder: `helmfile.yaml`, `values/`, `manifests/`.

## Related

- **[data-platform](https://github.com/revelly-io/data-platform)** — workloads
- **k8s-infra** — deprecated predecessor
