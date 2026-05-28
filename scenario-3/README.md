# Scenario 3 — Cross-cluster sidecar → sidecar over east-west

Fully self-contained. Does not rely on any artifact from scenarios 1 or 2.

## What it builds

```text
cluster-0 / client-c0                 cluster-1 / backend-c1
─────────────────────────             ─────────────────────────────────
client pod  ──┐                       ┌── ew-gateway pod (this scenario)
              │  HTTP plaintext       │       :15443 AUTO_PASSTHROUGH
              ▼                       │       LoadBalancer LB IP
        ┌─ client sidecar             │
        │   resolves backend host     │       │ SNI =
        │   via XCP-generated SE      │       │   outbound_.80_._.backend...
        │   endpoints =               │       ▼
        │   <ew-gateway LB IP>:15443  │   backend sidecar
        │                             │       :15006 inbound mTLS
        │   ── mesh mTLS dial ─────────────────►│
        │                             │       │ decrypts mTLS
        │                             │       ▼ forwards plaintext :8080
        │                             │   whoami :8080
        └─────────────────────────────┘
```

Networks are different (`nw-cluster-0` vs `nw-cluster-1`), so the caller's
sidecar cannot dial the backend's pod IP directly. The east-west gateway in
cluster-1 (deployed by this scenario) is the bridge.

## Apply order

```bash
# 1. cluster-0 (client)
kubectl --kubeconfig ~/.kube/cluster-0 apply -f 01-cluster-0-client/

# 2. cluster-1 (backend + east-west gateway)
kubectl --kubeconfig ~/.kube/cluster-1 apply -f 02-cluster-1-backend/

# 3. XCP Central (Workspace + WorkspaceSetting)
kubectl --kubeconfig ~/.kube/cluster-0 apply -f 03-xcp-config/
```

## Curl

```bash
kubectl --kubeconfig ~/.kube/cluster-0 -n client-c0 \
  exec deploy/client -c curl -- \
  curl -sv http://backend.backend-c1.svc.cluster.local/
```

Expected body contains `Name: backend-cluster-1`. The `X-Forwarded-Client-Cert`
header should show exactly **one** mTLS hop — `URI=spiffe://cluster.local/ns/client-c0/sa/client`,
because the east-west gateway auto-passthroughs without terminating.

## Dump (tctl collect)

The committed `dump/` was captured with these commands in parallel (1-minute
START → wait → END window, with curls fired in the middle):

```sh
# Shell 1 — cluster-1 dump (captures the ew-gateway and backend sidecar)
KUBECONFIG=~/.kube/cluster-1 tctl --config ~/Desktop/tctl-admin.config.yaml \
  collect-minimal \
  --hostname backend.backend-c1.svc.cluster.local --namespace backend-c1 \
  --duration 1m --disable-archive \
  -o ./scenario-3/dump/cluster-1

# Shell 2 — cluster-0 dump
#   For sidecar→sidecar, the CALLER side has no Service owning the hostname,
#   so --namespace client-c0 returned an empty 01-envoy/. Point --namespace
#   at the DESTINATION namespace (backend-c1) so tctl finds the hostname owner
#   and includes the client sidecar in the capture. Same hostname both sides.
KUBECONFIG=~/.kube/cluster-0 tctl --config ~/Desktop/tctl-admin.config.yaml \
  collect-minimal \
  --hostname backend.backend-c1.svc.cluster.local --namespace backend-c1 \
  --duration 1m --disable-archive \
  -o ./scenario-3/dump/cluster-0

# In a 3rd shell during the window, exec curl from the client pod a few times:
for i in 1 2 3 4 5; do
  kubectl --kubeconfig ~/.kube/cluster-0 -n client-c0 \
    exec deploy/client -c curl -- \
    curl -sS http://backend.backend-c1.svc.cluster.local/ -o /dev/null -w "%{http_code}\n"
  sleep 3
done
```

Note: the dump committed at the moment of writing used `--namespace client-c0`
for cluster-0 and therefore came back without `01-envoy/` on that side — only
cluster-1 captured envoys (ew-gateway and backend sidecar, which is enough to
verify the path). The form above is the corrected one for next time.

## What this scenario installs (and what it tears down with itself)

* cluster-0 namespace `client-c0` + client Deployment+SA (sidecar via ns label)
* cluster-1 namespace `backend-c1` containing:
  * backend Deployment+Service+SA (sidecar via ns label)
* cluster-1 namespace `istio-system`:
  * `GatewayDeployment type: EASTWEST` named `ew-gateway` — the cluster-1
    east-west bridge. Note: the CR lives in `istio-system` (the only
    namespace `xcp-operator-edge` watches for install CRDs) but the pod
    is deployed in `backend-c1` via `spec.namespace`. Deleting the
    `backend-c1` namespace also tears down the gateway pod; the CR can
    be deleted separately.
* Central namespace `xcp-system`:
  * `Workspace xc-ws` spanning both app namespaces
  * `WorkspaceSetting` designating the east-west gateway above and exposing
    `app=backend` services for cross-cluster routing

### Gotchas if you replicate this manifest by hand

Two things are easy to get wrong on a fresh XCP install:

1. **The GatewayDeployment CR must live in `istio-system`.** `xcp-operator-edge`
   ignores install CRDs in other namespaces — the CR will be created but its
   `.status` stays empty (no pod, no service). The pod runs in `spec.namespace`,
   which is independent and can be wherever you like (`backend-c1` here).

2. **`spec.kubeSpec.service` must be present (even empty).** If you omit it,
   the operator's port-merge webhook short-circuits at `if s.Service == nil`
   and never adds the 15443 (ISTIO-mTLS) port to the Service. The Service
   ends up with helm defaults (80/443/15021) and the cluster-1 edge logs:
   ```
   no required Gateway port (15443 or 15008) for the possible gateway service
   ```
   refusing to treat it as an east-west bridge. Result: no `gwinternal-*`
   ServiceEntry on cluster-0, and the curl fails with `Could not resolve host`.

With `type: EASTWEST` + an empty `service:` stanza, the resulting LB Service
has exactly `tls-istio-mtls=15443` and `status-port=15021` — the minimal
east-west surface.

Deleting the two app namespaces + the Workspace/WorkspaceSetting + the
ew-gateway CR in `istio-system` tears down the entire scenario.
