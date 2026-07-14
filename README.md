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
   curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable=traefik" sh -
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

## Known gap

`flux-run.sh` (gitignored, not committed) has a hardcoded GitHub PAT used for
the dev-cluster bootstrap. It never reached git history, but treat it as
compromised anyway (rotate it) and don't repeat the pattern for prod — pass
`GITHUB_TOKEN` as an env var from your shell/secret manager instead, as
step 2 above does.
