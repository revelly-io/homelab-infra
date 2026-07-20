# homelab-infra

Helmfile-managed k8s stacks for a data platform (MinIO, PostgreSQL, Polaris, Airflow, Spark operator). Toggle stacks with mise — `de:up`, `spark:down`, etc.

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
mise install   # helm, helmfile, kubectl, sops, age
export KUBECONFIG=/path/to/your/kubeconfig
cp environments/example.yaml environments/homelab.yaml   # pick any name
# edit environments/homelab.yaml for your cluster
```

Helmfile, stack charts, mise deploy tasks (`platform:up`, `bootstrap`, …), and SOPS wiring are **not in the repo yet** (MVP in progress). The env model and layout below are the target contract — deploy commands will work once those land.

## `--env` ties both repos together

Pick one name (`homelab`, `dev`, `lab`, …). Use the same string everywhere:

| | |
|--|--|
| homelab-infra | `environments/<env>.yaml`, `mise run de:up --env <env>` → `helmfile -e <env>` |
| data-platform | `--env <env>`, `spark_app/config/<env>/`, `.env.<env>` |
| MinIO | bucket name = `<env>` → data-platform warehouse `s3a://<env>` |

Example (`homelab`):

```
mise run de:up --env homelab          (homelab-infra)
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

### 3. Secrets (SOPS + age)

```bash
age-keygen -o ~/.config/sops/age/keys.txt
```

`.sops.yaml` and `stacks/*/secrets.sops.yaml` arrive with MVP. Each operator uses their own age key unless you share recipients.

### 4. Deploy stacks *(after MVP)*

```bash
mise run bootstrap
mise run platform:up --env homelab
mise run obs:up --env homelab
mise run de:up --env homelab
mise run spark:up --env homelab
```

Bring-up order, tear-down, `--purge`, deps, and memory: [docs/architecture/stacks.md](docs/architecture/stacks.md).

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
├── stacks/{platform,observability,data,spark,…}/
└── docs/architecture/stacks.md
```

Each stack folder: `helmfile.yaml`, `values/`, `manifests/`, `secrets.sops.yaml`.

## Related

- **[data-platform](https://github.com/revelly-io/data-platform)** — workloads
- **k8s-infra** — deprecated predecessor
