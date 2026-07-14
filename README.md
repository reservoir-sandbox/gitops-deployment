# gitops-deployment

Flux-managed GitOps repo for the Reservoir stack (frontend, backend, Postgres,
Redis, Garage S3, nginx ingress). Two clusters reconcile from this repo:

| Cluster | Purpose | Runtime |
|---|---|---|
| `clusters/dev-cluster` | local sandbox | k3d (`k3d-start.sh`) |
| `clusters/prod-cluster` | production | k3s on a VM |

Both point at the same `./apps` and `./infrastructure` trees, so anything
merged to `main` rolls out to both.

## Layout

- `infrastructure/` — namespaces, Helm repo sources, ingress-nginx,
  StorageClass, Garage (S3) and its bucket/key provisioning job.
- `apps/` — frontend, backend, Postgres, Redis.
- `charts/job-to-run/` — the one Helm chart used for all three analysis
  worker `Job`s the backend launches per sample (see
  [Analysis worker images](#analysis-worker-images-chartsjob-to-run)).
- `clusters/<name>/flux-system/` — per-cluster Flux bootstrap + the two
  top-level `Kustomization`s (`cluster-infrastructure`, `cluster-apps`).
- `.sops.yaml` — encryption rule: any `*-secret.yaml` file has its
  `data`/`stringData` fields encrypted with age; everything else in the file
  stays plaintext so diffs stay readable.

## Secrets (SOPS + age)

Files matching `*-secret.yaml` are committed **encrypted**. Flux's
kustomize-controller decrypts them at apply time using an age private key
stored as a `Secret` named `sops-age` in the `flux-system` namespace of each
cluster — that secret is created out-of-band, never committed.

- Encrypt a new secret file: `sops --encrypt --in-place path/to/foo-secret.yaml`
- Edit an existing one in place: `sops path/to/foo-secret.yaml`
- Decrypt to inspect: `sops --decrypt path/to/foo-secret.yaml`

`sops`/`age` are in the nix devShell (`direnv allow` or `nix develop`).

The current age keypair's **public** key is in `.sops.yaml`. The **private**
key was generated once during setup and was never committed — it must be
loaded into any cluster (dev or prod) that needs to decrypt, and kept
somewhere durable (password manager / secret store) since losing it means
re-encrypting every secret file with a new key. It has already been loaded
into the running dev-cluster.

## Deploying to a fresh production VM

Prerequisites on your workstation: `flux`, `sops`, `age`, `kubectl` (all in
the nix devShell), a GitHub PAT with repo access for bootstrap, and SSH
access to the VM.

1. **Install k3s on the VM** (Traefik disabled — we run our own ingress-nginx):
   ```sh
   curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server \
    --disable=traefik \
    --disable=servicelb \
    --disable=local-storage \
    --disable=metrics-server \
    --kubelet-arg=--cgroup-driver=systemd" sh -
   ```
   Copy `/etc/rancher/k3s/k3s.yaml` back to your workstation as your
   kubeconfig (swap `127.0.0.1` for the VM's address), or run the remaining
   steps over SSH with `kubectl` on the VM directly.

2. **Bootstrap Flux**, pointing at the prod path:
   ```sh
   export GITHUB_TOKEN=<pat with repo scope>
   flux bootstrap github \
     --owner=reservoir-sandbox --repository=gitops-deployment \
     --branch=main --path=./clusters/prod-cluster --token-auth
   ```
   This creates the `flux-system` namespace/controllers and commits
   `gotk-components.yaml`/`gotk-sync.yaml` into
   `clusters/prod-cluster/flux-system/` (the `apps.yaml`/`infrastructure.yaml`
   Kustomizations already exist in the repo).

3. **Load the SOPS decryption key** — do this before step 2's Kustomizations
   reconcile, or they'll fail with a decryption error:
   ```sh
   kubectl -n flux-system create secret generic sops-age \
     --from-file=age.agekey=/path/to/age.agekey
   ```
   Get `age.agekey` (the private key) from wherever it was stored when
   generated — see [Secrets](#secrets-sops--age) above. Reuse the same key
   for every cluster rather than minting a new one per environment, unless
   you deliberately want to scope secrets per-cluster (in which case give
   each its own `path_regex`/key pair in `.sops.yaml`).

4. **Verify**:
   ```sh
   flux get kustomizations -A
   kubectl get pods -A
   ```
   `ingress-nginx` gets a `LoadBalancer` Service; k3s's built-in ServiceLB
   binds it straight to the VM's ports 80/443 — no MetalLB needed for a
   single-node setup. Once pods are ready:
   ```sh
   curl http://<vm-ip>/           # frontend
   curl http://<vm-ip>/api/health/live   # backend
   ```

## Storage

`infrastructure/storage/local-path-storageclass.yaml` declares the
`local-path` StorageClass explicitly (provisioner: k3s's bundled
local-path-provisioner) instead of relying silently on k3s's default. Its
spec is intentionally identical to what k3s creates on its own — don't change
`reclaimPolicy` without disabling k3s's built-in `local-storage` addon first,
or the two will fight on every k3s restart / Flux reconcile.

Local-path volumes live on a single node's disk with no redundancy. For
anything you can't afford to lose (Postgres data, Garage bucket contents),
set up off-box backups — e.g. a CronJob doing `pg_dump` into the `garage`
bucket, or periodic volume snapshots — this repo doesn't do that yet.

## Analysis worker images (`charts/job-to-run`)

Each sample upload spawns one-shot Kubernetes `Job`s — one per `TaskType`
(`static`, `sandbox`, `ml`) — via a Flux `HelmRelease` that the backend
creates at runtime (see `backend`'s `K8sJobLauncher`). All three use the
single chart at `charts/job-to-run`, which picks the worker image by task
type from `charts/job-to-run/values.yaml`:

| Task type | Source repo | What it does |
|---|---|---|
| `static` | `reverse` | ELF structural analysis (headers, sections, disassembly, checksec) |
| `sandbox` | `auto-yara` | Generates a YARA detection rule from the sample |
| `ml` | `ml-models` | Forwards the static report to an external LLM summarizer and relays its verdict |

`static` and `sandbox` launch immediately, in parallel, like any other task.
`ml` does not — it consumes `static`'s output, so the backend only launches
it once `static` reaches a terminal state
(`JobService._maybe_launch_ml` in the backend repo). If `static` fails, `ml`
is marked failed immediately with no Job launched (nothing to summarize).

`static`/`sandbox` implement the shared worker contract: download the sample
from S3 via `S3_OBJECT_KEY`, run the analysis, upload the report to S3 if
it's over 1MB, then `POST` the result to the backend's internal callback
endpoint using `WORKER_CALLBACK_SECRET`. `ml` skips the S3 sample download
entirely — it never touches the raw binary — and instead reads the static
report the backend hands it (`STATIC_REPORT` inline, or `STATIC_REPORT_S3_KEY`
if the static report itself was too big to inline), POSTs it to
`RESERVOIR_SUMMARIZER_URL/api/v1/summarize-static`, and relays the response
verbatim as its own result (this is exactly the `MLReportData` shape the
frontend's `MLReport` component expects — see `LLM_integration.md`).
`yara_matches` is intentionally omitted from that request: `auto-yara`
*generates* a rule from the sample, it doesn't *match* against a corpus of
known signatures, so it has nothing in the shape the summarizer's optional
`yara_matches` field expects.

Common env vars (`S3_ACCESS_KEY`, `S3_SECRET_KEY`, `S3_ENDPOINT_URL`,
`S3_BUCKET_NAME`, `S3_OBJECT_KEY`, `TASK_ID`, `BACKEND_CALLBACK_URL`,
`WORKER_CALLBACK_SECRET`) are injected by the chart's `templates/job.yaml`
for every task type; S3 credentials come from the `backend-s3-credentials`
Secret, which `infrastructure/s3/garage-backend-key-init-job.yaml` mirrors
into both the `apps` and `jobs` namespaces. `RESERVOIR_SUMMARIZER_URL`,
`RESERVOIR_SUMMARIZER_TIMEOUT_SECONDS`, `STATIC_REPORT`, and
`STATIC_REPORT_S3_KEY` are only injected for `taskType: ml`.

### LLM summarizer (external, not managed by this repo)

The summarizer runs on a Yandex Cloud VM outside this cluster — see
`LLM_integration.md` for the full DevOps handoff (systemd units, log
locations, deploy process). This cluster is **not** in the Yandex VPC, so
`charts/job-to-run/values.yaml`'s `summarizer.url` points at its **public**
IP (`158.160.219.46:8000`), not the private one the guide recommends for
same-VPC setups.

**Manual step required, outside this repo**: the Yandex VM's security group
must allow inbound TCP 8000 from this cluster's egress IP. Nothing here can
configure that — it's Yandex Cloud console/CLI access on the VM owner's
side. `infrastructure/jobs/networkpolicy.yaml` only controls the outbound
side (cluster → VM); it doesn't help if the VM's own firewall still rejects
the connection.

**Updating a worker script**: edit the relevant repo (`reverse`,
`auto-yara`, or `ml-models`), push to `main`. Each has a `.github/workflows/ci.yml`
that builds and pushes the image to `ghcr.io/reservoir-sandbox/<repo>:<sha>`,
then opens a commit against **this** repo bumping
`charts/job-to-run/values.yaml`'s `workers.<taskType>.tag` — the same
auto-tag-bump pattern the `backend` repo uses for its own deployment. Flux
picks it up from there; no manual manifest edits needed. Each of those three
repos needs a `PAT_TOKEN` secret (repo-scoped GitHub PAT with write access to
`gitops-deployment`) configured in its own GitHub Actions settings, same as
`backend` already has.

If two of these CI runs land within the same ~minute, the second's
auto-commit push can lose a non-fast-forward race against the first (seen in
practice) — check `gh run list --repo reservoir-sandbox/<repo>` after
pushing more than one at once, and manually re-apply the tag bump in
`charts/job-to-run/values.yaml` if a run failed on that step.

## Known gap

`flux-run.sh` (gitignored, not committed) has a hardcoded GitHub PAT used for
the dev-cluster bootstrap. It never reached git history, but treat it as
compromised anyway (rotate it) and don't repeat the pattern for prod — pass
`GITHUB_TOKEN` as an env var from your shell/secret manager instead, as
step 2 above does.
