# Scenario 4 — Cross-cluster k8s Service call forced through a transit Gateway

A sidecar-injected client in cluster-0 dials a **real cluster-local
Kubernetes Service FQDN** in cluster-2 (`echo.echo-tr.svc.cluster.local`) —
and because network reachability forbids cluster-0 from talking to
cluster-2 directly, XCP routes the request through a **transit `Gateway`**
sitting in cluster-1.

The shape — sidecar client / transit `Gateway` with `transit: true` /
terminal `Gateway` with `serviceDestination` — matches the canonical
pattern that every XCP e2e uses (`xcp/test/e2e/tests/templates/*` with
`transit: true` or `trafficMode: TRANSIT`). Two deliberate departures from
the e2es:

* `hostname:` on both Gateways is the **real k8s svc FQDN** instead of a
  logical name like `echo.tetrate.io`. The e2es always use a logical name;
  XCP treats `hostname:` as an opaque identifier so either works.
* cluster-1's `Cluster` CR is **not** marked `tier1Cluster: true`. The
  e2es always set that on the transit cluster; this local XCP build runs
  with relaxed tier1 enforcement (same caveat scenario-1's README calls out
  at the bottom).

```text
                 cluster-0                cluster-1                  cluster-2
   curl    ┐    ┌─────────┐               ┌─────────────┐            ┌──────────────┐
   pod     │    │ client  │  mesh-mTLS    │ transit     │ mesh-mTLS  │ terminal     │
   (sidecar┼exec│ sidecar │ ─ :15443 ───▶ │  Gateway    │ ─ :15443 ─▶│  Gateway     │
   inject) │    │ SE for  │               │ http:       │            │ http:        │
           ▼    │ echo.   │               │  hostname:  │            │  hostname:   │
                │ echo-tr.│               │  echo.…     │            │   echo.…     │
                │ svc.    │               │  transit:   │            │  port: 8080  │
                │ cluster │               │  true       │            │  serviceDest:│
                │ .local  │               │  cluster-   │            │   echo svc   │
                │         │               │  Dest:      │            │     │        │
                │         │               │  cluster-2  │            │     ▼        │
                │         │               └─────────────┘            │ echo sidecar │
                │         │                                          │  :15006      │
                │         │                                          │     │        │
                │         │                                          │     ▼        │
                │         │                                          │ echo pod     │
                │         │                                          │ whoami :8080 │
                └─────────┘                                          └──────────────┘
```

* Client just calls `curl http://echo.echo-tr.svc.cluster.local/`. From the
  client's pov it's an ordinary k8s svc DNS lookup. The result is the same
  whoami response it would get if echo lived in cluster-0 itself.
* Two `Gateway` CRs make this work, plus the GlobalSetting reachability map:
  * **transit Gateway** in cluster-1 declares the FQDN with
    `transit: true` and `clusterDestination: cluster-2`. XCP auto-binds it
    on the pod's :15443 mesh-mTLS listener.
  * **terminal Gateway** in cluster-2 declares the SAME FQDN with
    `port: 8080` (a regular ingress server) and
    `serviceDestination: echo-tr/echo.echo-tr.svc.cluster.local:80` — the
    real cluster-local Service. The Gateway pod (`echo-tr-gateway`) takes
    inbound mesh-mTLS on :15443 and routes the decrypted HTTP to the svc.
* On every workspace cluster XCP renders a `ServiceEntry` for the FQDN.
  The cluster-0 sidecar sees two candidate endpoints (cluster-1 transit
  Gateway, cluster-2 terminal Gateway). The `networkReachability` rule
  (cluster-0 → cluster-1 only) eliminates the cluster-2 candidate, so
  the sidecar routes through the transit.
* Why does the **terminal Gateway** exist if the destination is a real
  k8s Service? Because the transit machinery wires up its outbound cluster
  with a `DestinationRule` subset selector that includes a label like
  `xcp.tetrate.io/svc-port-80: "true"`. Only XCP-managed Gateway CR endpoints
  carry that label — the bare-Service east-west propagation path
  (scenario-3's `WorkspaceSetting.defaultEastWestGatewaySettings`) does
  NOT. Pairing transit with bare east-west propagation gets you a 503 UH at
  the transit hop (no healthy upstream). Confirmed by experiment — see the
  inline comment in
  [03-cluster-2-backend/03-gatewaydeployment-backend.yaml](03-cluster-2-backend/03-gatewaydeployment-backend.yaml).

## Cluster roles

| Cluster     | Workload                                                                            | Namespace   |
|-------------|-------------------------------------------------------------------------------------|-------------|
| `cluster-0` | sidecar-injected `client` Deployment (curl)                                         | `client-tr` |
| `cluster-1` | transit `Gateway` data plane (no public ports)                                      | `transit`   |
| `cluster-2` | terminal `Gateway` data plane (LB :15443) + real `echo` Service + Deployment        | `echo-tr`   |

## Prerequisites

* All three kind clusters up: `cluster-0`, `cluster-1`, `cluster-2`. See
  [`../onboarding-local.md`](../onboarding-local.md) for cluster-2.
* XCP Central running on `cluster-0` (`xcp-system`).
* XCP Edge + istiod `stable` on every cluster.
* `kubectl --kubeconfig ~/.kube/cluster-{0,1,2}` works.

## Apply order

```bash
# 1. cluster-0 (curl client, sidecar-injected)
kubectl --kubeconfig ~/.kube/cluster-0 apply -f 01-cluster-0-client/

# 2. cluster-1 (transit Gateway data plane)
kubectl --kubeconfig ~/.kube/cluster-1 apply -f 02-cluster-1-transit/

# 3. cluster-2 (echo workload + terminal Gateway data plane)
kubectl --kubeconfig ~/.kube/cluster-2 apply -f 03-cluster-2-backend/

# 4. XCP Central (Workspace + GlobalSetting + transit & terminal GatewayGroups + Gateways)
kubectl --kubeconfig ~/.kube/cluster-0 apply -f 04-xcp-config/
```

### ⚠️ Heads-up: `01-globalsetting.yaml` mutates a shared resource

`04-xcp-config/01-globalsetting.yaml` **overwrites** the existing
`xcp-system/global` GlobalSetting. The new contents are a strict superset
of the original — safe for scenarios 1-3 (they only need
cluster-0 ↔ cluster-1) but it does need the overwrite:

```yaml
networkReachability:
  nw-cluster-0: nw-cluster-1                # unchanged
  nw-cluster-1: nw-cluster-0,nw-cluster-2   # adds cluster-2 reachability
  nw-cluster-2: nw-cluster-1                # NEW
```

To roll back to the original:

```yaml
networkReachability:
  nw-cluster-0: nw-cluster-1
  nw-cluster-1: nw-cluster-0
```

## Smoke-testing the path

```bash
kubectl --kubeconfig ~/.kube/cluster-0 -n client-tr exec deploy/client -c curl -- \
  curl -sv http://echo.echo-tr.svc.cluster.local/
```

Expected response: traefik/whoami output containing `Name: echo-cluster-2-transit`
(set in [03-cluster-2-backend/02-echo-app.yaml](03-cluster-2-backend/02-echo-app.yaml#L27)).

To prove the request actually passes through the transit hop, tail the
transit Gateway log on cluster-1 during the curl:

```bash
kubectl --kubeconfig ~/.kube/cluster-1 -n transit logs -l app=transit-tier1-gateway -f
```

Each curl should produce one access-log line on that pod.

## Files

```text
scenario-4/
├── 01-cluster-0-client/
│   ├── 01-namespace.yaml                          client-tr (sidecar inject)
│   └── 02-client.yaml                             curl Deployment
├── 02-cluster-1-transit/
│   ├── 01-namespace.yaml                          transit (sidecar inject not required)
│   └── 02-gatewaydeployment-transit-tier1.yaml    transit Gateway pod (port 15443)
├── 03-cluster-2-backend/
│   ├── 01-namespace.yaml                          echo-tr (sidecar inject for echo)
│   ├── 02-echo-app.yaml                           echo Deployment + Service (app=echo, port 80→8080)
│   └── 03-gatewaydeployment-backend.yaml          terminal Gateway pod (LB :15443)
└── 04-xcp-config/
    ├── 01-globalsetting.yaml                      networkReachability (forces the transit)
    ├── 02-workspace.yaml                          transit-ws covers all three namespaces
    ├── 03-gatewaygroup-transit.yaml               group for cluster-1/transit
    ├── 04-gateway-transit.yaml                    Gateway, http transit:true, hostname=echo.echo-tr...
    ├── 05-gatewaygroup-backend.yaml               group for cluster-2/echo-tr
    └── 06-gateway-backend.yaml                    Gateway, http port:8080, hostname=echo.echo-tr…, serviceDestination=real svc
```

## Dump (tctl collect)

Captured with three parallel `tctl collect-minimal --duration 1m` runs and
a curl loop fired from the client pod during the 1-minute window:

```sh
# Shell 1 — cluster-0 (sidecar client side)
KUBECONFIG=~/.kube/cluster-0 tctl --config ~/Desktop/tctl-admin.config.yaml \
  collect-minimal \
  --hostname echo.echo-tr.svc.cluster.local --namespace client-tr \
  --duration 1m --disable-archive \
  -o ./scenario-4/dump/cluster-0

# Shell 2 — cluster-1 (transit Gateway)
KUBECONFIG=~/.kube/cluster-1 tctl --config ~/Desktop/tctl-admin.config.yaml \
  collect-minimal \
  --hostname echo.echo-tr.svc.cluster.local --namespace transit \
  --duration 1m --disable-archive \
  -o ./scenario-4/dump/cluster-1

# Shell 3 — cluster-2 (terminal Gateway + echo workload)
KUBECONFIG=~/.kube/cluster-2 tctl --config ~/Desktop/tctl-admin.config.yaml \
  collect-minimal \
  --hostname echo.echo-tr.svc.cluster.local --namespace echo-tr \
  --duration 1m --disable-archive \
  -o ./scenario-4/dump/cluster-2

# Shell 4: during the 1-minute window, fire curls from the client pod:
for i in $(seq 1 15); do
  kubectl --kubeconfig ~/.kube/cluster-0 -n client-tr exec deploy/client -c curl -- \
    curl -s http://echo.echo-tr.svc.cluster.local/ -o /dev/null -w "req=$i HTTP=%{http_code}\n"
  sleep 3
done
```

The `tctl collect-minimal --hostname X --namespace Y` scoping captures
proxies that **serve** hostname X on a listener. The terminal Gateway pod
serves the FQDN with an HCM listener on :15443 (and the route to the real
svc), so it shows up in the dump. The echo sidecar does NOT serve the
FQDN — its inbound listeners are port-based on :15006 / :8080, and only its
*outbound* xDS state mentions the FQDN. To inspect the echo sidecar's view,
run a full non-minimal `tctl collect` for cluster-2 in addition.

## How the chain forms

XCP renders per-cluster `ServiceEntry` + `DestinationRule` for the hostname
`echo.echo-tr.svc.cluster.local`. The endpoints differ by source cluster:

* **`cluster-0`** — sidecar SE; endpoints are cluster-2's terminal Gateway
  (declares the FQDN with port 8080) and cluster-1's transit Gateway
  (declares the FQDN with `transit: true`). Reachability
  (`nw-cluster-0: nw-cluster-1` only) eliminates the cluster-2 candidate,
  leaving cluster-1 transit. So the sidecar dials cluster-1 transit
  Gateway on :15443.
* **`cluster-1` / `transit` Gateway** — `transit: true` auto-binds its envoy
  on :15443 mesh-mTLS for the FQDN. Inbound mTLS terminates, HCM applies
  the `clusterDestination: cluster-2` route. The DR subset selector
  (`xcp.tetrate.io/cluster: cluster-2` + `xcp.tetrate.io/svc-port-80`)
  matches the endpoint XCP rendered for the cluster-2 terminal Gateway —
  so the next-hop dial is to that Gateway's :15443.
* **`cluster-2` / terminal Gateway** — receives mesh-mTLS on :15443,
  decrypts, the HCM route applies `serviceDestination: echo-tr/echo…:80`,
  forwards plain HTTP to the real `echo` Service. The echo pod's sidecar
  wraps the gateway→pod hop in auto-mTLS.

`networkReachability` is what forces the chain. Set
`nw-cluster-0: nw-cluster-1,nw-cluster-2` and the sidecar in cluster-0
would dial the cluster-2 terminal Gateway directly — skipping the transit
entirely.

## Tear down

```bash
kubectl --kubeconfig ~/.kube/cluster-0 delete -f 04-xcp-config/   # except globalsetting
kubectl --kubeconfig ~/.kube/cluster-2 delete -f 03-cluster-2-backend/
kubectl --kubeconfig ~/.kube/cluster-1 delete -f 02-cluster-1-transit/
kubectl --kubeconfig ~/.kube/cluster-0 delete -f 01-cluster-0-client/
```

Restore the original `GlobalSetting`:

```bash
cat <<'EOF' | kubectl --kubeconfig ~/.kube/cluster-0 apply -f -
apiVersion: xcp.tetrate.io/v2
kind: GlobalSetting
metadata: { name: global, namespace: xcp-system }
spec:
  networkSettings:
    networkReachability:
      nw-cluster-0: nw-cluster-1
      nw-cluster-1: nw-cluster-0
EOF
```

## Reference

Modelled on the eight XCP e2e templates that use `transit: true` or
`trafficMode: TRANSIT`. All of them share the same invariants:

* Sidecar-injected in-mesh client (never an external curl, never the
  client cluster's own Gateway).
* `Gateway` (or older `Tier1Gateway`) with `transit: true` on the
  middle hop, route via `clusterDestination` naming the next cluster.
* Terminal `Gateway` (or `IngressGateway`) with `serviceDestination` to
  the cluster-local k8s Service.
* `GlobalSetting.networkReachability` partitioning client and service
  clusters so they can only reach each other through the transit.

The e2e most directly equivalent to scenario-4 is
`xcp/test/e2e/tests/templates/bridged-multicluster-ugw-test-with-jwt-tier1.yaml`
(test `inlineAuthzForMeshInternalTier1Test` in
`xcp/test/e2e/multicluster/security/security_api.go`). It runs over the
4-cluster `MultiNetworkTopology`; scenario-4 collapses the same flow into
3 clusters by reusing cluster-0 as the client (the e2e uses cluster-2).

What scenario-4 deliberately does NOT use: scenario-3's
`WorkspaceSetting.defaultEastWestGatewaySettings` + bare-Service east-west
exposure. That mechanism's SE endpoints lack the `xcp.tetrate.io/svc-port-N`
label that the transit Gateway's DR subset selector requires, so the two
mechanisms do not compose — confirmed by experiment (503 UH no_healthy_upstream
at the transit hop). See the inline comment in
[03-cluster-2-backend/03-gatewaydeployment-backend.yaml](03-cluster-2-backend/03-gatewaydeployment-backend.yaml).
