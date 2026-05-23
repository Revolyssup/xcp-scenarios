# Scenario 1 — Tier1 → Tier2 TLS Passthrough

An external client makes an **HTTPS** request that transits a **Tier1 gateway**
on `cluster-0` and a **Tier2 ingress gateway** on `cluster-1`, finally reaching
an echo backend. Neither gateway decrypts the request — both do **TLS
passthrough** (SNI-based routing only). TLS is terminated **end-to-end at the
echo backend**.

## Topology

```
                cluster-0 (Tier1)            cluster-1 (Tier2)
  external      ┌───────────────────┐        ┌───────────────────┐
  client  ──────▶ tier1-gateway     │        │ echo-gateway      │
  HTTPS         │ (passthrough,     │ HTTPS  │ (passthrough,     │ HTTPS
  SNI=          │  SNI route)       ├────────▶  SNI route)       ├────────▶ echo
  echo.tetrate  └───────────────────┘ SNI    └───────────────────┘  (whoami,
  .io :17443       :17443                       :17443                terminates
                                                                      TLS :9443)
```

- **No TLS termination at either gateway.** Each gateway reads only the SNI
  (`echo.tetrate.io`) from the TLS ClientHello and TCP-proxies the encrypted
  bytes onward.
- The **echo backend** (`traefik/whoami`) terminates TLS itself, using a
  self-signed cert minted by cert-manager.

## Cluster roles

| Cluster     | Kubeconfig            | Role                                          |
|-------------|-----------------------|-----------------------------------------------|
| `cluster-0` | `~/.kube/cluster-0`   | Tier1 gateway **+** hosts XCP Central (`xcp-system`) |
| `cluster-1` | `~/.kube/cluster-1`   | Tier2 ingress gateway **+** echo backend      |

## Prerequisites (assumed already installed)

- XCP Central on `cluster-0`, XCP Edge on both clusters, Istio (`stable`
  revision), cert-manager, and a LoadBalancer provider (metallb).
- The `Cluster` resources `cluster-0` / `cluster-1` and `GlobalSettings` exist
  in XCP Central — **do not delete these**.

## Directory layout

```
scenario-1/
├── 01-cluster-0-tier1/      # apply to cluster-0
│   ├── 01-namespace.yaml
│   └── 02-gatewaydeployment-tier1.yaml
├── 02-cluster-1-tier2/      # apply to cluster-1
│   ├── 01-namespace.yaml
│   ├── 02-echo-cert.yaml
│   ├── 03-echo-app.yaml
│   └── 04-gatewaydeployment-tier2.yaml
└── 03-xcp-config/           # apply to XCP Central (cluster-0, ns xcp-system)
    ├── 01-workspace.yaml
    ├── 02-gatewaygroup-tier1.yaml
    ├── 03-tier1gateway.yaml
    ├── 04-gatewaygroup-tier2.yaml
    └── 05-ingressgateway-tier2.yaml
```

`GatewayDeployment` is an XCP *install* CRD that creates the gateway Envoy
pods + Services. The XCP config in `03-xcp-config/` is what programs those
Envoys with the actual passthrough listeners.

## Deployment order

Run from the `scenario-1/` directory. Apply the data plane first, then the
XCP config.

### Step 1 — Tier1 cluster (cluster-0)

```sh
kubectl --kubeconfig ~/.kube/cluster-0 apply -f 01-cluster-0-tier1/
```

### Step 2 — Tier2 cluster (cluster-1)

```sh
# namespace + cert first
kubectl --kubeconfig ~/.kube/cluster-1 apply -f 02-cluster-1-tier2/01-namespace.yaml
kubectl --kubeconfig ~/.kube/cluster-1 apply -f 02-cluster-1-tier2/02-echo-cert.yaml

# wait for cert-manager to issue the echo-tls Secret, else the echo pod
# will be stuck ContainerCreating on the missing secret volume
kubectl --kubeconfig ~/.kube/cluster-1 -n echo wait --for=condition=Ready \
  certificate/echo-tls --timeout=60s

# echo backend + Tier2 gateway data plane
kubectl --kubeconfig ~/.kube/cluster-1 apply -f 02-cluster-1-tier2/03-echo-app.yaml
kubectl --kubeconfig ~/.kube/cluster-1 apply -f 02-cluster-1-tier2/04-gatewaydeployment-tier2.yaml
```

### Step 3 — XCP config (applied to XCP Central on cluster-0)

```sh
kubectl --kubeconfig ~/.kube/cluster-0 apply -f 03-xcp-config/
```

XCP Central validates this config and propagates it to the edges, which
translate it into Istio `Gateway`/`VirtualService`/etc. on the gateway pods.
Give it ~15–30s to converge.

## Verify

```sh
# Gateway data planes are READY
kubectl --kubeconfig ~/.kube/cluster-0 -n istio-system get gatewaydeployment
kubectl --kubeconfig ~/.kube/cluster-1 -n istio-system get gatewaydeployment

# Pods running
kubectl --kubeconfig ~/.kube/cluster-0 -n tier1 get pods
kubectl --kubeconfig ~/.kube/cluster-1 -n echo  get pods   # echo-gateway + echo

# Tier1 LoadBalancer IP (the "external" entry point)
T1_IP=$(kubectl --kubeconfig ~/.kube/cluster-0 -n tier1 \
  get svc tier1-gateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Tier1 LB IP: $T1_IP"

# Send the request as an external client. -k because the cert is self-signed;
# --resolve forces SNI = echo.tetrate.io.
curl -k --resolve echo.tetrate.io:17443:$T1_IP https://echo.tetrate.io:17443/
```

Expected: a `traefik/whoami` response body containing `Name: echo-cluster-1`
and the echo pod's `Hostname:` — proof the request reached the backend on
`cluster-1` through both gateways.

**If the LB IP is not routable from your host** (common with kind on macOS),
port-forward instead:

```sh
kubectl --kubeconfig ~/.kube/cluster-0 -n tier1 port-forward svc/tier1-gateway 17443:17443
# in another shell:
curl -k --resolve echo.tetrate.io:17443:127.0.0.1 https://echo.tetrate.io:17443/
```

## Dump (tctl collect)

The committed `dump/` was captured with these commands (in parallel),
which run a 1-minute window — START snapshot, then a wait, then END snapshot
— so the access logs in `end-*/01-envoy/.../proxy.log` show traffic sent
during the window:

```sh
# In one shell — cluster-0 (Tier1 + Central)
KUBECONFIG=~/.kube/cluster-0 tctl --config ~/Desktop/tctl-admin.config.yaml \
  collect --minimal \
  --hostname echo.tetrate.io --namespace echo \
  --until 1m --disable-archive \
  -o ./scenario-1/dump/cluster-0

# In a second shell — cluster-1 (Tier2 + echo backend)
KUBECONFIG=~/.kube/cluster-1 tctl --config ~/Desktop/tctl-admin.config.yaml \
  collect --minimal \
  --hostname echo.tetrate.io --namespace echo \
  --until 1m --disable-archive \
  -o ./scenario-1/dump/cluster-1

# During the 1-minute window, send the curl from "Verify" above
# multiple times so the END snapshot captures matching access-log lines.
```

`--minimal --hostname X --namespace Y` scopes the envoy capture to proxies
that *own* the hostname `X` in namespace `Y`. Here that's the tier1 gateway
(cluster-0/echo via the cross-cluster wiring) and the tier2 gateway + echo
pod (cluster-1/echo).

## Teardown

Reverse order — XCP config first, then data plane:

```sh
kubectl --kubeconfig ~/.kube/cluster-0 delete -f 03-xcp-config/
kubectl --kubeconfig ~/.kube/cluster-1 delete -f 02-cluster-1-tier2/
kubectl --kubeconfig ~/.kube/cluster-0 delete -f 01-cluster-0-tier1/
```

## Notes & gotchas

- **Why the echo pod has no sidecar.** In a passthrough path the Tier2 gateway
  forwards the still-encrypted stream straight to the pod; whoami terminates
  the client's TLS. A sidecar would only add an interception layer with
  nothing useful to do, so `03-echo-app.yaml` sets
  `sidecar.istio.io/inject: "false"`.
- **Self-signed cert.** The client must use `curl -k` (or trust the
  `echo-tls` CA). The cert CN/SAN is `echo.tetrate.io`, so the request SNI
  must match.
- **Tier1 / Tier2 separation.** XCP normally requires a cluster to be marked
  `tier1_cluster: true` (on its `Cluster` resource) before it will translate a
  `Tier1Gateway`. This dev environment runs without that enforcement (the
  prior setup ran a Tier1Gateway on `cluster-0` the same way). If you instead
  see an *"ignoring tier1 ingress gateway config ... in tier2 cluster"* error
  in the edge logs / config status, mark `cluster-0` with
  `spec.tier1_cluster: true`.
- **Ports.** `17443` is the passthrough listener on both gateways; `9443` is
  the TLS port the echo backend listens on. All three are wired together by
  the `route` in `05-ingressgateway-tier2.yaml`.
