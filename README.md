# Reservoir — Infrastructure Engineering Report

DevOps specification for Reservoir: what runs, what deploys it, and what
decisions were made building the cluster side of this project. Scope is
infrastructure — manifests, charts, CI/CD, cluster config — not application
internals.

Stack: GitHub Actions (CI), GHCR (image registry), Kubernetes / k3s
(runtime), Flux CD (GitOps controller), Kustomize (manifest composition),
Helm (chart templating and release management).

---

## 1. High-level architecture

The project consists of:

- 2 applications: frontend, backend
- 3 analysis scripts, each run as a Job: `reverse` (static analysis),
  `auto-yara` (sandbox / YARA rule synthesis), `ml-models` (LLM summarizer
  relay)
- 6 infrastructure services: PostgreSQL, Redis, Garage (S3-compatible object
  store), ingress-nginx, Flux, external LLM summarizer

Two Flux-managed clusters reconcile from the same `./apps` and
`./infrastructure` Kustomize trees: `clusters/dev-cluster` (k3d, local) and
`clusters/prod-cluster` (k3s, single VM `10.93.27.36`). Same manifests, no
per-environment fork. The LLM summarizer runs on a separate Yandex Cloud VM
outside the cluster network, reached over its public IP.

### Request path

```
Browser
  |  HTTPS
  v
ingress-nginx (infrastructure ns)
  |
  +-- path /      -> frontend :3000  (React/Vite SPA, apps ns)
  |
  +-- path /api   -> backend :8000   (FastAPI, apps ns)
                        |
          +-------------+-------------+
          |             |             |
          v             v             v
     PostgreSQL       Redis      Kubernetes API
     (apps ns)       (apps ns)   -> backend creates a HelmRelease (jobs ns)
```

### Job fan-out (one upload, three workers)

```
Flux reconciles the HelmRelease
  |
  v
charts/job-to-run (jobs ns)              <- one chart, three task types
  |
  +-- taskType=static  -> reverse worker    -> Garage (S3, if report > 1 MB)
  |
  +-- taskType=sandbox -> auto-yara worker  -> Garage
  |
  +-- taskType=ml      -> ml-models worker  -> external LLM VM (public IP)
       (only launched once static is terminal - see chart notes below)

all three POST their result to the backend's internal callback endpoint,
bearer-authed against worker-callback-secret
```

Each worker Job is fire-and-forget: `restartPolicy: Never`, `backoffLimit: 2`,
`ttlSecondsAfterFinished: 300` — the Job self-cleans, nothing else has to
garbage-collect finished pods.

**Script-as-a-Job.** Every analysis run is a Kubernetes `Job`, one per
`(sample, taskType)` pair, templated from a single shared chart,
`charts/job-to-run`. `taskType` (`static` / `sandbox` / `ml`) selects the
worker image via `(index .Values.workers .Values.taskType)` in
`templates/job.yaml`, and, only when `taskType: ml`, the template renders an
extra block of env vars (`RESERVOIR_SUMMARIZER_URL`,
`RESERVOIR_SUMMARIZER_TIMEOUT_SECONDS`, `STATIC_REPORT`,
`STATIC_REPORT_S3_KEY`). One chart, one values file, three worker images.

**Backend -> Flux -> Helm -> Job.** The backend does not call the Kubernetes
Job API directly. It creates a Flux `HelmRelease` object (`K8sJobLauncher`,
backend's own service) targeting `charts/job-to-run`; Flux's helm-controller
performs the `helm install`. Consequences of that design:

- job launches inherit Flux's reconciliation and retry semantics
- `flux get helmreleases -A` lists every in-flight analysis job
- the backend's Kubernetes RBAC surface is `create`/`get`/`list` on
  `HelmRelease` only — it never touches the Job/Pod API directly

**Why Helm.** The chart's parameterization (`taskType`, `workers.<type>`) is
what lets one chart serve three different worker images from one values
file, and gives every launched job a release name and history Flux already
tracks — instead of the backend templating raw Job YAML per task type by
hand.

---

## 2. Poly-repo architecture and CI/CD pipeline

Five repositories, each with its own CI, converging on one cluster:

| Repo | Owns | CI writes to |
|---|---|---|
| **gitops-deployment** | All K8s manifests, the shared `job-to-run` chart, Flux bootstrap for both clusters | — (target of every other repo's CI) |
| **backend** | FastAPI app, Alembic migrations, `K8sJobLauncher` | `apps/backend/deployment.yaml` |
| **frontend** | React/Vite SPA | `apps/frontend/deployment.yaml` |
| **reverse** | `static` worker | `charts/job-to-run/values.yaml` -> `workers.static.tag` |
| **auto-yara** | `sandbox` worker | `charts/job-to-run/values.yaml` -> `workers.sandbox.tag` |
| **ml-models** | `ml` worker | `charts/job-to-run/values.yaml` -> `workers.ml.tag` |

`gitops-deployment` holds every manifest and chart; each app/worker repo owns
its own CI and only writes back the field for its own image. Nobody needs
write access to another repo's source.

### Lifecycle, on push to `main`

```
source repo CI:
  1. build image, push to ghcr.io/reservoir-sandbox/<repo>:<commit-sha>
     (backend only: gated by lint-test - black, ruff, mypy, pytest)
  2. checkout gitops-deployment (PAT_TOKEN)
  3. yq-patch the relevant manifest / chart value with the new tag
  4. commit as github-actions[bot], push to gitops-deployment main

gitops-deployment main -> Flux polls -> reconciles -> new image running
```

Per repo:

- **backend**: `yq` patches both `containers[].image` and
  `initContainers[].image` in `apps/backend/deployment.yaml` in one call —
  the init container runs the Alembic migration ahead of the app container,
  so both move in lockstep.
- **frontend**: patches `apps/frontend/deployment.yaml`'s single container
  image.
- **reverse / auto-yara / ml-models**: `yq eval -i
  ".workers.<taskType>.tag = \"$SHA\""` against
  `charts/job-to-run/values.yaml`.

Image tags are always the full commit SHA (`docker/metadata-action`,
`type=sha,format=long`), never `latest` or a branch name — `main` in
`gitops-deployment` fully determines what's running.

Each source repo's CI needs its own repo-scoped `PAT_TOKEN` secret (write
access to `gitops-deployment`); set up manually, per organization, out of band.

**Known gap:** two CI runs landing within the same push window can race on
the `git push` into `gitops-deployment` (non-fast-forward); no automatic
retry exists, the tag bump is currently re-applied by hand when it happens.

---

## 3. Security and its gaps

### In place

| Control | Where | What it does |
|---|---|---|
| Secrets encryption | `.sops.yaml` (SOPS + age) | Encrypts `data`/`stringData` in any `*-secret.yaml`; age private key never committed, loaded per cluster as the `sops-age` Secret. |
| NetworkPolicy | `apps/backend/networkpolicy.yaml`, `infrastructure/jobs/networkpolicy.yaml` | Default-deny. Backend ingress: only from `ingress-nginx` and the `jobs` namespace. Backend egress: Postgres 5432, Redis 6379, Garage 3900, K8s API 6443 on both the ClusterIP (`10.43.0.1/32`) and the node IP (`10.93.27.36/32`) — required because this cluster's NetworkPolicy enforcement evaluates the post-DNAT destination for Service traffic. `jobs` namespace egress whitelists a single `/32` for the external LLM VM. |
| RBAC | `infrastructure/jobs/rbac.yaml`, `infrastructure/s3/garage-backend-key-init-job.yaml` | Backend ServiceAccount: `create`/`get`/`list` on `HelmRelease` in `jobs` ns only. Garage bootstrap ServiceAccount: `create`/`get`/`update`/`patch` on one named Secret (`backend-s3-credentials`) in `apps` + `jobs` ns. |
| Non-root containers | `backend/Dockerfile` | Dedicated `fastapi` system user, `USER fastapi` before `CMD`. |

### Gaps

| Gap | Where | State |
|---|---|---|
| No TLS termination | `apps/frontend/ingress.yaml`, `apps/backend/ingress.yaml` | `ssl-redirect: "false"`, no `tls:` block, no cert-manager `ClusterIssuer` anywhere in the repo. |
| External LLM VM on a public IP | `charts/job-to-run/values.yaml` (`summarizer.url`), `LLM_integration.md` | Doc recommends a private VPC IP with security-group restriction; deployment uses the public IP with a manually opened security-group rule. |
| No Pod Security Standards | `infrastructure/security/namespaces.yaml` | No `pod-security.kubernetes.io/*` labels set at namespace level. |
| Shared `PAT_TOKEN` | CI secrets in `backend`, `reverse`, `auto-yara`, `ml-models` | One repo-scoped PAT with write access to `gitops-deployment`, reused as a CI secret across four repos. |
| Hardcoded PAT in `flux-run.sh` | Local-only, gitignored, dev-cluster bootstrap | Never reached Git history; prod bootstrap uses a `GITHUB_TOKEN` env var instead. |

---

## 4. Specific engineering solutions

| Problem | Where | What was done |
|---|---|---|
| `ml` task needs the `static` task's output before it can run | `charts/job-to-run/templates/job.yaml` | The `taskType: ml` branch is the only one that renders `STATIC_REPORT` / `STATIC_REPORT_S3_KEY` / summarizer env vars. Backend only creates the `ml` HelmRelease after the `static` task reaches a terminal state. |
| Garage bootstrap re-run would otherwise rotate credentials on every reconcile | `infrastructure/s3/garage-backend-key-init-job.yaml` | The Job checks whether the access key already in the Secret still validates against Garage's admin API before minting a new one — re-running it is idempotent. |
| CI auto-commit race (§2) | All 5 repos' workflows | Documented, not yet fixed; no `git pull --rebase` retry before push. |

---

## 5. Scaling strategy

| Tier | Current state |
|---|---|
| App (frontend, backend) | Stateless `Deployment` + `Service`, `replicas: 1`. |
| Storage (Postgres, Redis, Garage) | Single-node `local-path` StorageClass (`infrastructure/storage/local-path-storageclass.yaml`). |
| Analysis workers | One Job per upload (`charts/job-to-run`); concurrency bounded by per-worker `resources` in `values.yaml` (200m/1000m CPU, 256Mi/1Gi memory requests/limits) against node capacity. |
| LLM summarizer | Single external VM, single-threaded Ollama inference (~2-3 min/call per `LLM_integration.md`). |
| Autoscaling | None configured — no `HorizontalPodAutoscaler`, no cluster-autoscaler. |
| Environments | Two Flux clusters (`dev-cluster` k3d, `prod-cluster` k3s) reconcile the same `./apps`/`./infrastructure` trees. |
