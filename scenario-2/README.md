# Scenario 2 — TLS termination at Tier1

## Topology (contrast with Scenario 1)

```
                                     ┌── tier1-term/tier1-term-gateway ──┐
                                     │   * Listener :18443                │
                                     │   * TERMINATES echo-term.tetrate.io│
client ── HTTPS :18443 ─────────────▶│     (cert: echo-term-edge-tls)     │
                                     │   * Forwards via Istio mTLS :15443 │
                                     └────────────────┬───────────────────┘
                                                      │   (east-west mTLS,
                                                      │    ALPN istio)
                                                      ▼
                                     ┌── echo-term/echo-term-gateway ─────┐
                                     │   * IngressGateway (NOT an E/W gw)  │
                                     │   * Listener :15443 — TERMINATES    │
                                     │     the inbound mTLS, runs HCM,     │
                                     │     routes by HTTP Host header to   │
                                     │     echo-term.echo-term.svc :80     │
                                     └────────────────┬────────────────────┘
                                                      │   (fresh upstream
                                                      │    mesh mTLS,
                                                      │    auto-mTLS)
                                                      ▼
                                     ┌── echo-term/echo-term pod ──────────┐
                                     │   * istio-proxy (sidecar)           │
                                     │   * whoami :8080 (PLAIN HTTP)       │
                                     └─────────────────────────────────────┘
```

* Only the Tier1 gateway terminates the public TLS.
* Every other hop is mesh mTLS, but **terminated and re-originated at each
  L7 hop** — not one end-to-end mTLS tunnel. Two mTLS hops total:
  tier1 → echo-term-gateway, and echo-term-gateway → echo-term sidecar.
* `echo-term-gateway` is **not** an east-west gateway. It's a regular
  XCP `IngressGateway`. Tier1 reaches it on `:15443` because the
  cross-cluster `ServiceEntry` for `echo-term.tetrate.io` has
  `location: MESH_INTERNAL` with an endpoint at `<lb-ip>:15443` and a
  SPIFFE SAN — that combination makes the Tier1 sidecar originate Istio
  mTLS to that endpoint. XCP auto-provisions the matching `:15443`
  HCM listener on the IngressGateway pod for the hostname.
* The workload has **no cert, no TLS code**.

## Apply order

```bash
# 1. cluster-0 (Tier1 + edge cert)
kubectl --kubeconfig ~/.kube/cluster-0 apply -f 01-cluster-0-tier1/

# 2. cluster-1 (Tier2 gateway + echo workload)
kubectl --kubeconfig ~/.kube/cluster-1 apply -f 02-cluster-1-tier2/

# 3. XCP Central (Workspace, GatewayGroups, Tier1Gateway, IngressGateway)
kubectl --kubeconfig ~/.kube/cluster-0 apply -f 03-xcp-config/
```

## Curl

```bash
# from the macOS host once the Tier1 LB IP is assigned
T1_IP=$(kubectl --kubeconfig ~/.kube/cluster-0 -n tier1-term \
          get svc tier1-term-gateway -o jsonpath='{.spec.loadBalancer.ingress[0].ip}')

curl -ksv --resolve "echo-term.tetrate.io:18443:${T1_IP}" \
  https://echo-term.tetrate.io:18443/

# if the kind LB IP is not routable from macOS, run inside cluster-0:
T1_CIP=$(kubectl --kubeconfig ~/.kube/cluster-0 -n tier1-term \
           get svc tier1-term-gateway -o jsonpath='{.spec.clusterIP}')
kubectl --kubeconfig ~/.kube/cluster-0 run curl-test --rm -i --restart=Never \
  --image=curlimages/curl:8.10.1 -- \
  curl -ksv --resolve "echo-term.tetrate.io:18443:${T1_CIP}" \
  https://echo-term.tetrate.io:18443/
```

Expected response body contains `Name: echo-term-cluster-1`.

## Dump (tctl collect)

The committed `dump/` was captured with these commands in parallel (1-minute
window — START snapshot, wait, END snapshot — with curls fired in the middle
so the END's `01-envoy/.../proxy.log` carries the access-log lines):

```sh
# Shell 1 — cluster-0 (terminates TLS at tier1-term-gateway)
KUBECONFIG=~/.kube/cluster-0 tctl --config ~/Desktop/tctl-admin.config.yaml \
  collect-minimal \
  --hostname echo-term.tetrate.io --namespace tier1-term \
  --duration 1m --disable-archive \
  -o ./scenario-2/dump/cluster-0

# Shell 2 — cluster-1 (Tier2 + echo workload, both in echo-term ns)
KUBECONFIG=~/.kube/cluster-1 tctl --config ~/Desktop/tctl-admin.config.yaml \
  collect-minimal \
  --hostname echo-term.tetrate.io --namespace echo-term \
  --duration 1m --disable-archive \
  -o ./scenario-2/dump/cluster-1

# In a 3rd shell during the 1-minute window, fire the curl from above
# a few times (every 3s) so the END snapshot captures matching log entries.
```

`collect-minimal --hostname X --namespace Y` scopes envoy capture to proxies that
own hostname `X` in namespace `Y`. Each side uses its own namespace because
each side has a gateway in that namespace serving the hostname.
