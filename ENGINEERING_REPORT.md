# Reservoir — Production Infrastructure Engineering Report

Reservoir is a malware-analysis platform: a user uploads an ELF binary, three
independent analysis workers run against it, and an LLM-generated verdict is
served back through a web UI. This document is the DevOps/infrastructure
README for the whole system — five repositories, two clusters, one external
VM. It describes what exists, why it was built the way it was, and what has
not been done yet.

Written 2026-07-22 against the state of all five repos at that date. Treat it
as a snapshot: re-verify against `git log` before relying on specifics like
image tags or commit hashes.

---

## 1. High-level architecture

Everything runs in a single Kubernetes cluster (dev: k3d locally; prod: k3s
on a single Yandex/on-prem-style VM at `10.93.27.36`), except the LLM
summarizer, which is a deliberately external Yandex Cloud VM outside the
cluster's network. Flux CD reconciles cluster state from this repo; nothing
is applied by hand.

### Request path

```
┌─────────┐   HTTPS    ┌───────────────────────┐
│ Browser │ ─────────▶ │  ingress-nginx          │
└─────────┘            │  (infrastructure ns)    │
                        └───────────┬─────────────┘
                     path: /        │        path: /api
                        ┌───────────┴───────────┐
                        ▼                        ▼
             ┌────────────────────┐   ┌────────────────────┐
             │ frontend :3000      │   │ backend :8000        │
             │ React 19 / Vite SPA │   │ FastAPI              │
             │ (apps ns)           │   │ (apps ns)            │
             └────────────────────┘   └──────────┬────────────┘
                                                   │
                       ┌───────────────────────────┼───────────────────────────┐
                       ▼                            ▼                           ▼
             ┌──────────────────┐        ┌──────────────────┐       ┌───────────────────────┐
             │ PostgreSQL        │        │ Redis             │       │ Kubernetes API          │
             │ (Bitnami chart)   │        │ (Bitnami chart)   │       │ -> creates a HelmRelease │
             │ apps ns           │        │ apps ns           │       │ object in the jobs ns    │
             └────────────────────┘        └────────────────────┘       └────────────┬─────────────┘
```

The backend does **not** call the Kubernetes Job API directly. It creates a
Flux `HelmRelease` custom resource (`K8sJobLauncher` in
`backend/app/services/job_launcher_service.py`), and Flux's own
helm-controller performs the actual install. This was an intentional choice:
job launches inherit Flux's reconciliation/retry semantics for free, and
`flux get helmreleases -A` becomes a real-time view of every in-flight
analysis job — no separate tooling needed to answer "what jobs are running
right now."

### Job fan-out (one upload -> three workers)

```
        Flux reconciles the HelmRelease
                      │
                      ▼
        ┌──────────────────────────────┐
        │ charts/job-to-run (jobs ns)    │   <- one chart, three task types
        └───┬─────────────┬──────────┬──┘
   taskType=static  taskType=sandbox  taskType=ml
            │              │              │
            ▼              ▼              ▼ (waits for static, see §4)
   ┌─────────────────┐ ┌─────────────────┐ ┌──────────────────────┐
   │ reverse           │ │ auto-yara         │ │ ml-models              │
   │ ELF structural     │ │ synthesizes a     │ │ relays the static      │
   │ analysis           │ │ YARA rule from    │ │ report to an external  │
   │ (headers, sections,│ │ the sample        │ │ LLM and relays its     │
   │ disasm, checksec)  │ │                   │ │ verdict back           │
   └─────────┬──────────┘ └─────────┬─────────┘ └──────────┬──────────────┘
             │ S3 (if report >1MB)  │                        │ HTTPS, public IP
             ▼                      ▼                        ▼
   ┌────────────────────────────────────┐        ┌─────────────────────────────┐
   │ Garage — S3-compatible object store  │        │ Yandex Cloud VM (external)    │
   │ (infrastructure ns)                  │        │ Ollama + Qwen3.5:4b behind    │
   └────────────────────────────────────┘        │ Flask/Gunicorn on :8000        │
             ▲                                    │ not managed by this repo       │
             │                                    └─────────────────────────────┘
   POST result, bearer-authed with worker-callback-secret
             │
   ┌─────────┴─────────────────────────────────────────────┐
   │ backend's internal callback endpoint (/internal/...)    │
   └───────────────────────────────────────────────────────┘
```

Each worker Job is a fire-and-forget pod: it downloads the sample from S3
(except `ml`, which never touches the raw binary), does its analysis, uploads
the report to S3 only if it exceeds 1&nbsp;MB, and POSTs its result back to
the backend over a shared bearer secret. `ttlSecondsAfterFinished: 300`
cleans the pod up automatically; nothing needs to garbage-collect finished
Jobs.

---

## 2. Repository organization — poly-repo scheme

Five independent repositories, each with its own CI, deployed as separate
container images but converging on one Kubernetes cluster:

| Repo | Owns | Deploys as | CI pushes to |
|---|---|---|---|
| **gitops-deployment** | All K8s manifests, the one shared worker Helm chart, Flux bootstrap for both clusters. No application code. | *(not a container — this is the desired-state source of truth)* | n/a (target of every other repo's CI) |
| **backend** | FastAPI REST API, auth, Alembic migrations, job orchestration, `K8sJobLauncher` | `ghcr.io/reservoir-sandbox/backend` | `apps/backend/deployment.yaml` |
| **frontend** | React/Vite SPA | `ghcr.io/reservoir-sandbox/frontend` | `apps/frontend/deployment.yaml` |
| **reverse** | `static` worker — ELF structural analysis | `ghcr.io/reservoir-sandbox/reverse` | `charts/job-to-run/values.yaml` → `workers.static.tag` |
| **auto-yara** | `sandbox` worker — YARA rule synthesis | `ghcr.io/reservoir-sandbox/auto-yara` | `charts/job-to-run/values.yaml` → `workers.sandbox.tag` |
| **ml-models** | `ml` worker — relays the static report to the external LLM and forwards its verdict | `ghcr.io/reservoir-sandbox/ml-models` | `charts/job-to-run/values.yaml` → `workers.ml.tag` |

**Why poly-repo instead of a monorepo**: each analysis worker has an
independent release cadence and its own toolchain (Python + capstone for
`reverse`, YARA tooling for `auto-yara`, a thin S3/HTTP relay for
`ml-models`), and the frontend/backend split is a standard two-team
boundary. GitOps then becomes the seam that reunifies all of it: every repo
treats `gitops-deployment` as its deploy target, and nobody needs write
access to anyone else's source, only to the shared chart/manifest fields
relevant to their own image.

**A naming trap worth flagging**: `ml-models` still contains a large amount
of dead weight from before the LLM-summarizer pivot — a full local
`scikit-learn` training pipeline (`models/file_analysis_baseline.joblib`,
`data/`, `scripts/*.sh`, a `Reservoir/` package, a `requirements.txt` with
`lief`/`scikit-learn`/`flask`) that the deployed worker no longer uses at
all. The actual deployed artifact is `worker_entrypoint.py` plus
`requirements-worker.txt` (`aioboto3`, `aiohttp` — nothing else). Even the
external LLM VM's own handoff doc says it outright: *"`ml-models` is only a
Docker Compose service name; it is not a model or repository."* This repo
should either be pruned to just the worker, or explicitly split — right now
a new contributor opening it will reasonably assume the sklearn pipeline is
live in production, and it is not.

---

## 3. CI/CD pipelines — step-by-step lifecycle

All five repos converge on the same pattern, GitHub Actions in the source
repo pushing an image and then committing directly into
`gitops-deployment`:

```
 developer            source repo CI               gitops-deployment          Flux
 pushes to   ────▶   1. checkout            ────▶   (target repo)      ────▶  reconciles
 main                2. build + push image                                    on next poll
                         to ghcr.io/reservoir-        6. yq-patch the
                         sandbox/<name>:<sha>            relevant manifest/
                      3. checkout gitops-              chart value
                         deployment (PAT_TOKEN)      7. commit "chore(...):
                      4. locate the manifest/            auto-update ... tag"
                         chart-value field            8. git push
                      5. (backend only) lint/
                         mypy/pytest gate first
```

Concretely, per repo:

- **backend**: `lint-test` job (black, ruff, mypy, pytest) gates a
  `build-and-push-docker` job that only runs on `push` to `main` (not on
  PRs). It patches **both** `spec.template.spec.containers[].image` and
  `spec.template.spec.initContainers[].image` in
  `apps/backend/deployment.yaml` in one `yq` call — the init container runs
  the Alembic migration ahead of the app container, so both must move in
  lockstep or a deploy can run new app code against an unmigrated schema.
- **frontend**: single job, builds, pushes, patches
  `apps/frontend/deployment.yaml`'s single container image.
- **reverse / auto-yara / ml-models**: identical shape — build, push,
  `yq eval -i ".workers.<taskType>.tag = \"$SHA\""` against
  `charts/job-to-run/values.yaml`. No test/lint gate before the push (unlike
  backend) — worth tightening, see §4.

Image tags are always the full commit SHA (`type=sha,format=long,prefix=`
via `docker/metadata-action`), never `latest` or a branch name. Combined with
Flux's Git-source-of-truth model, this means **the state of
`gitops-deployment`'s `main` branch fully determines what's running in
prod** — there is no drift between "what's deployed" and "what a `git log`
of this repo shows," which is the entire point of GitOps and was preserved
end-to-end rather than left as an afterthought.

Every source-repo CI needs a repo-scoped `PAT_TOKEN` secret (write access to
`gitops-deployment`) configured in its own GitHub Actions settings — this is
manual, per-repo, out-of-band setup that isn't itself declarative.

**Known race condition** (hit in practice during this project, documented in
`README.md`'s "Analysis worker images" section): if two of these CI runs
finish within roughly the same minute, the second's `git push` can lose a
non-fast-forward race against the first's auto-commit. Nothing currently
retries this automatically — the fix today is manual (`gh run list`, then
re-apply the tag bump by hand). A cheap real fix would be a `git pull
--rebase` immediately before the commit-and-push step in all five workflows;
this has not been implemented.

---

## 4. Specific engineering solutions, and the fight against future failures

This section is the "here's what actually broke, and what we did about it"
record — each of these was found through live end-to-end testing against the
real cluster, not code review, and each fix has a durable trace (a comment,
a README note, or a structural change) so it doesn't silently regress.

**S3 key mismatch (backend).** The upload path computed one S3 object key for
the actual `put_object` call and recorded a *different* key in Postgres.
Every download after the first would 404. Fixed by collapsing both to one
`object_name` variable (`app/services/sample_service.py`) so it's
structurally impossible for them to diverge again.

**S3 region mismatch (all three workers + backend).** Garage is configured
with a custom region (`local-cluster`), not AWS's default `us-east-1`. boto3
normally self-heals a wrong region via a redirect hint in the XML error body
of `GetObject`/`ListObjects` — but `HeadObject` has no response body, and
`download_file()`/`download_fileobj()` call `head_object` internally first,
so the redirect never fires and every download failed with a `400`. Root
cause found via Garage's own server logs (`Authorization header malformed,
unexpected scope`). Fixed by pinning `region_name="local-cluster"` on every
S3 client construction, in all three worker repos and in the backend's
`app/db/s3.py`. This is the kind of bug that is invisible in unit tests and
only surfaces against the real object store — a strong argument for keeping
the dev cluster's Garage instance real rather than mocked (see §7).

**Python 3.12 / capstone incompatibility (reverse).** `capstone==5.0.1`
imports `distutils`, removed from the stdlib in 3.12. Base image was already
`python:3.12-slim`. Fixed by bumping to `capstone>=5.0.9`. Cheap fix, but a
reminder that pinned dependency versions need to be re-validated against
base-image Python version bumps, not just left alone because "it worked
before."

**SHA256 dedup can wedge a sample forever.** `sample_service.py` dedupes
uploads by SHA-256: if the same file is already known,
`get_active_by_sample_and_version` reuses the existing job instead of
re-launching. That lookup treats `PENDING`/`RUNNING`/`COMPLETED` as "active"
*indefinitely* — there is no staleness timeout. A job that gets stuck
(cluster hiccup, evicted pod, worker crash mid-run) permanently absorbs every
future re-upload of that exact file; the UI just shows the old stuck job
forever with no way to force a retry. This was observed directly during
testing (a stuck job silently ate a re-upload that should have re-triggered
analysis) and worked around with a manual, scoped DB/S3 wipe rather than a
code fix. **This is still open** — a staleness cutoff or an explicit
"force re-run" affordance is the natural fix and has not been built.

**ML task must wait for static to finish.** Originally all three tasks
launched in parallel. Once `ml`'s job became "summarize the static report"
rather than "run an independent classifier," it needed static's output as
input. This is now an explicit dependency in
`backend/app/services/job_service.py`: `apply_task_result` calls
`_maybe_launch_ml` only when the `static` task reaches a terminal state, and
if `static` failed, `ml` is marked `FAILED` immediately with **no Job ever
launched** — there's deliberately nothing to summarize, so there's no reason
to spend a pod, a Garage round-trip, or a call to the external LLM finding
that out the slow way. The static report is handed to `ml` either inline
(`STATIC_REPORT` env var, JSON-encoded) or as an S3 pointer
(`STATIC_REPORT_S3_KEY`) depending on size — same inline/S3-if-too-big
pattern used for worker→backend callbacks, applied symmetrically here.

**Frontend hard-reload white screen (`base: './'` vs `'/'`).** Vite's
relative asset base broke direct navigation/hard-reload on nested SPA routes
(`/report/12`): the browser resolved `./assets/...` relative to the *current*
path rather than the site root, nginx's SPA fallback then served the HTML
document at the URL where JS was expected, and React never mounted. Fixed by
switching `vite.config.ts`'s `base` to `'/'`. Verified with an actual
`vite build` and `curl` against the deployed asset paths, not just "it
compiles."

**Unhandled external-service response shape (React error #31).** The LLM
summarizer's documented contract (`LLM_integration.md`) says
`evidence[].examples` is `string[]`. Its own `deterministic_fallback` path —
triggered when the LLM's output fails internal "grounding validation" — was
observed putting a raw object in there instead, which React then tried to
render directly as a child and crashed the entire tree (no `ErrorBoundary`
existed anywhere in the app, so one bad field took down the whole page, not
just one card). Root-caused by reading the actual stored task result out of
Postgres and matching it against the guide's documented (but
unimplemented) handling for `deterministic_fallback`: show "inconclusive,"
gate the raw failure reason behind `user.role === "admin"`. Fixed in
`frontend/src/components/MLReport/{types,index}.tsx`, plus a general
defensive `typeof ex === "string" ? ex : JSON.stringify(ex)` coercion on the
normal rendering path so a *future* contract violation degrades instead of
crashing.

**Still open: no `ErrorBoundary`.** The fix above patches the one shape that
was actually observed breaking. Nothing stops the next unanticipated shape
from a third-party integration (or a bug in our own backend) from
white-screening the app again. A top-level `ErrorBoundary` around the report
view is cheap insurance that hasn't been added.

---

## 5. Scaling strategy

Current state is deliberately **single-node, not highly available**, because
this is an MVP on one VM — but the design tries not to foreclose scaling
later:

- **Stateless-by-construction app tier.** Both `frontend` and `backend` are
  ordinary `Deployment`s behind `Service`s; nothing pins a request to a
  specific pod. Bumping `replicas` on either is a one-line manifest change
  with no code implications, *once* the single-node capacity constraint
  below is lifted.
- **The real scaling bottleneck today is the storage tier.** Postgres,
  Redis, and Garage all currently sit on `local-path` (k3s's
  bundled single-node local-path-provisioner — see
  `infrastructure/storage/local-path-storageclass.yaml`), which ties every
  stateful pod to whichever node's disk it landed on. This is fine for one
  VM; it is the first thing that has to change (a real StorageClass with
  network-attached volumes, or moving Postgres/Redis to managed services)
  before `replicas > 1` or multi-node makes sense for the stateful pieces.
- **Analysis workers already scale horizontally for free.** Because each
  upload spawns its *own* Job (not a shared long-running worker pool), the
  natural unit of concurrency is "however many Jobs the cluster's spare
  capacity can schedule at once." No manual capacity planning was needed to
  support N simultaneous analyses — it falls out of the architecture. The
  actual limit is the `resources.requests`/`limits` in
  `charts/job-to-run/values.yaml` (200m/1000m CPU, 256Mi/1Gi memory per
  worker pod) against whatever the node has free, and the external LLM VM's
  single-threaded CPU-bound inference (~2-3 minutes per call per
  `LLM_integration.md`) is the actual serialization point for the `ml` task
  specifically — that VM has no scaling story at all right now, not even a
  documented one.
- **No autoscaling configured anywhere** — no `HorizontalPodAutoscaler` on
  backend/frontend, no cluster-autoscaler (there's nothing to autoscale onto;
  it's one VM). This is consistent with "MVP on one VM," not an oversight,
  but should be named explicitly as a gap rather than silently assumed away
  once real traffic shows up.
- **Two clusters exist (`dev-cluster`, `prod-cluster`) reconciling from the
  same `./apps`/`./infrastructure` trees**, which means the path to a real
  multi-environment setup (and eventually blue/green or canary prod
  clusters) is already half-built structurally — it's a matter of adding a
  third `clusters/<name>/` directory and pointing Flux bootstrap at it, not
  a redesign.

---

## 6. Security — and its gaps

What's actually in place:

- **Secrets are SOPS/age-encrypted at rest in Git.** Any file matching
  `*-secret.yaml` has its `data`/`stringData` fields encrypted
  (`.sops.yaml`'s `encrypted_regex: ^(data|stringData)$`); the private age
  key is never committed and is loaded into each cluster out-of-band as a
  `Secret` named `sops-age`. `apps/backend/worker-callback-secret.yaml` is a
  correct example of this in practice.
- **NetworkPolicies are default-deny-by-omission and explicitly scoped per
  workload.** `backend`'s policy only allows ingress from `ingress-nginx` and
  from the `jobs` namespace (for worker callbacks), and egress is
  enumerated port-by-port (Postgres 5432, Redis 6379, Garage 3900, the K8s
  API on both its ClusterIP *and* real node IP — a detail worth noting since
  this cluster's NetworkPolicy enforcement evaluates the post-DNAT
  destination for Service traffic, so both had to be allow-listed). The
  `jobs` namespace's worker pods get their own egress-only policy that
  explicitly carves out a single `/32` for the external LLM VM
  (`infrastructure/jobs/networkpolicy.yaml`) rather than allowing broad
  internet egress.
- **Least-privilege RBAC for job launching.** The backend's ServiceAccount
  can only `create`/`get`/`list` `HelmRelease` objects in the `jobs`
  namespace (`infrastructure/jobs/rbac.yaml`) — it cannot create arbitrary
  pods, read secrets it doesn't own, or touch other namespaces. Similarly,
  the Garage bootstrap Job's ServiceAccount is scoped to `create`/`get`
  /`update`/`patch` on exactly one named Secret (`backend-s3-credentials`)
  in exactly the two namespaces that need it.
- **Non-root containers.** The backend Dockerfile builds a dedicated
  `fastapi` system user and drops root before `CMD` — a small but real
  hardening step that's easy to skip and wasn't.
- **Credential rotation is designed for, not ad hoc**: the Garage bootstrap
  Job reuses an existing key if the Secret it finds still validates against
  Garage's own admin API, rather than minting (and thereby invalidating) a
  new one on every reconcile.

What's missing or actively risky — the "security or lack thereof" the report
title promises:

- **`prod-config` is a live, plaintext Kubernetes admin credential, committed
  to this repository, in Git history, right now.** It's a full kubeconfig
  for `https://10.93.27.36:6443` with an embedded client certificate and
  private key — not a `*-secret.yaml` file, so SOPS never touches it, and
  it's not in `.gitignore`. Anyone with read access to this repo (including
  its full history — see commit `d86a084`) has cluster-admin over the
  production VM. The certificate currently in the working tree differs from
  the one last committed (`git status` shows `M prod-config`), meaning it's
  actively being used and edited in place — which also means the *old*
  cert/key pair, sitting permanently in Git history, may still be valid
  unless it was explicitly revoked. **This needs to be rotated, removed from
  history (not just deleted going forward), and added to `.gitignore`
  immediately** — it is the single highest-severity finding in this report.
- **No TLS termination configured at the ingress.** `apps/frontend/ingress.yaml`
  explicitly sets `nginx.ingress.kubernetes.io/ssl-redirect: "false"`, and
  neither ingress declares a `tls:` block or references a cert-manager
  `Certificate`. Whatever HTTPS behavior was observed during testing is
  coming from ingress-nginx's default self-signed fallback certificate, not
  a managed one — there's no cert-manager `ClusterIssuer` anywhere in this
  repo. Fine for an MVP behind a raw IP; not fine to carry into anything
  with a real domain and real users without revisiting.
- **The external LLM VM is a real, if narrow, attack-surface expansion.**
  Traffic to it leaves the cluster over the public internet to a single
  `/32`. `LLM_integration.md` itself recommends binding the summarizer to a
  private VPC IP and restricting inbound by security group — the current
  setup (public IP, security-group rule opened manually, "outside this
  repo") is a known, documented, and accepted deviation from that
  recommendation for the MVP, but it does mean the summarizer's own
  authentication (there doesn't appear to be any beyond network-level
  filtering) is the only thing standing between the internet and an LLM
  inference endpoint, if that security-group rule is ever accidentally
  widened.
- **No admission control / pod security standards enforced** — nothing sets
  namespace-level Pod Security Standards (`restricted`/`baseline`), so
  privilege-escalation or hostPath mounts aren't structurally blocked at the
  cluster level, only by convention (which the backend Dockerfile follows,
  but nothing enforces on the worker images built by three separate teams'
  CI).
- **CI's `PAT_TOKEN` is a broad, long-lived, repo-scoped GitHub PAT** shared
  across (at minimum) `backend`, `reverse`, `auto-yara`, `ml-models` — a
  compromise of any one repo's Actions environment gets write access to
  `gitops-deployment`, i.e. to production deploy state. A fine-grained,
  per-repo deploy key or a short-lived OIDC-based GitHub App token would
  narrow this meaningfully; not implemented.
- **`flux-run.sh` had a hardcoded PAT** used for dev-cluster bootstrap. It's
  gitignored and never reached Git history, but the README already flags it
  as "treat it as compromised, rotate it" — worth repeating here as a
  pattern to never reproduce for prod, which correctly uses an env var
  instead.

---

## 7. Dev-layer testing process

There is a real dev cluster, not just a plan for one: `clusters/dev-cluster`
runs on `k3d` locally (`k3d-start.sh`), reconciling from the **exact same**
`./apps` and `./infrastructure` Kustomize trees as prod
(`clusters/prod-cluster`). This is the load-bearing design decision in this
whole repo: there is structurally no way for dev and prod to declare
different application manifests, because they're the same files. What can
differ is per-cluster Flux bootstrap config and whatever's only reachable
from one network (e.g. the external LLM VM's security-group allowlist would
need dev's egress IP added separately — not yet done, meaning `ml` task
testing against the *real* summarizer currently only works from prod's
egress IP).

How testing actually happened during this project (worth recording, since
none of it is automated yet):

1. **Live, manual end-to-end runs against the real prod cluster** were the
   primary verification method for every fix in this report — SSH into the
   VM, upload a real test payload (`reverse/example/example.elf`), watch
   `kubectl get jobs -n jobs`/`flux get helmreleases -A`, confirm the
   callback landed in Postgres, confirm the report is fetchable from the
   frontend. This is how every bug in §4 was actually found — none of them
   would have been caught by a unit test, because they were all integration
   failures between real components (S3 region negotiation, K8s Job
   scheduling, an external HTTP contract, a browser's asset-resolution
   behavior).
2. **The backend has real unit tests** (`backend/tests/unit/`, `pytest`,
   gated in CI) — but they cover auth/security/schema logic, not the
   job-orchestration or worker-callback paths that produced every bug found
   this session. There is no integration test suite that spins up Garage +
   a fake worker + the backend together.
3. **The three worker repos have no CI test gate at all** before their image
   push (unlike backend's `lint-test` job) — `reverse`, `auto-yara`, and
   `ml-models`'s workflows go straight from checkout to build-and-push. A
   worker that fails to even import (as happened with `auto-yara`'s
   `NameError` from a missing top-level import, fixed in `1018cf9`) would
   only be caught by someone manually running a real job against it, exactly
   as happened here.
4. **Cleanup is manual and DB/S3-aware, not scripted.** Clearing test data
   between runs meant targeted deletes across the CASCADE FK chain
   (`samples`→`jobs`→`job_tasks`, `samples`→`user_samples`→
   `user_sample_jobs`) plus an S3 object wipe against Garage — done by hand
   each time via `kubectl`/`psql`/S3 API calls, not a reusable "reset the dev
   environment" script or Job.

**What this adds up to**: the dev cluster is a faithful, low-friction mirror
of prod's declarative state (this part works well and was a deliberate,
successful design choice), but the actual *test process* on top of it is
still "a person drives it by hand against a real cluster." That's exactly
how every real bug in this system has been found so far, which says
something in its favor — but it doesn't scale past one engineer's attention,
and none of it runs automatically on a PR. The highest-leverage next step
here isn't more unit tests in isolation; it's a scripted integration flow
(upload a known sample against the dev cluster, assert on the final report
shape) that turns "SSH in and watch it happen" into something CI can run on
every worker-repo PR before an image ever reaches `main`.

---

## Summary of open items (quick reference)

| Priority | Item | Where |
|---|---|---|
| Critical | Plaintext cluster-admin kubeconfig committed to Git (`prod-config`) | gitops-deployment |
| High | No TLS/cert-manager on either ingress; `ssl-redirect: "false"` | gitops-deployment |
| High | SHA256 dedup has no staleness timeout — a stuck job wedges a sample forever | backend |
| Medium | No `ErrorBoundary` in the frontend — any unanticipated response shape can white-screen the whole app | frontend |
| Medium | Worker repos' CI has no test gate before image push | reverse, auto-yara, ml-models |
| Medium | No integration test suite covering job orchestration / worker callbacks | backend |
| Medium | Shared long-lived `PAT_TOKEN` across all source repos | all |
| Low | `ml-models` repo is ~90% dead legacy sklearn code alongside the real worker | ml-models |
| Low | CI auto-commit race can lose a tag bump under concurrent pushes | all worker repos |
| Low | No autoscaling / single-node stateful storage — expected for MVP, but undocumented as a conscious tradeoff until now | gitops-deployment |
