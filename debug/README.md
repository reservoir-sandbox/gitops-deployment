# Debug tools

Ad-hoc manifests/scripts for troubleshooting the live cluster. Nothing in
this directory is referenced by any `kustomization.yaml` — Flux never
applies it, and `kubectl kustomize apps|infrastructure` never picks it up.
Apply/delete these manually when needed.

## `apitest-pod.yaml` / `apitest.sh`

A disposable, unrestricted pod (no `NetworkPolicy` selects it) used to test
raw TCP/HTTP connectivity to a given host:port from inside the cluster.

Useful for isolating "a `NetworkPolicy` is blocking this" from "the target
is genuinely unreachable" — run this unrestricted pod against the same
target, and/or `kubectl exec` into the *restricted* pod and try the same
connection from there, and compare.

Origin: diagnosing why the backend's egress rule for the `kubernetes`
Service ClusterIP (`10.43.0.1:443`) still produced `Connection refused`.
This cluster's embedded k3s `NetworkPolicy` engine turned out to enforce on
the **post-DNAT** destination (the real node/API-server IP, `10.93.27.36:6443`)
rather than the ClusterIP — see `apps/backend/networkpolicy.yaml` for the
resulting egress rule.

**Usage:**

```bash
# edit the curl target in apitest-pod.yaml first if testing something else
./debug/apitest.sh
# or manually:
kubectl apply -f debug/apitest-pod.yaml
kubectl logs pod/apitest -n default
kubectl delete -f debug/apitest-pod.yaml
```
