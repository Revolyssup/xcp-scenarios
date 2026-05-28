# Onboard a new kind cluster to the local XCP demo

This repo currently runs a two-cluster local setup:

| Cluster     | Kubeconfig              | Role                                              |
|-------------|-------------------------|---------------------------------------------------|
| `cluster-0` | `~/.kube/cluster-0`     | XCP **Central** (`xcp-system`) **+** XCP edge      |
| `cluster-1` | `~/.kube/cluster-1`     | XCP **Edge** only                                  |

Both were brought up by [../backup/deploy-xcp-multicluster.sh](../backup/deploy-xcp-multicluster.sh)
against `../xcp/test/e2e` kind tooling — there is **no TSB Management Plane**
in this setup, only XCP. The MP-style onboarding doc at
`~/backup/onboarding.md` does **not** apply here; this file is its local-XCP
equivalent.

This doc walks through adding `cluster-2` (a third edge) to the existing pair.
The same flow extends to `cluster-3` (the XCP repo's `MAX_CLUSTERS=4`).

## How the current onboarding differs from MP-style

| MP-style (`~/backup/onboarding.md`)               | Local XCP (this doc)                            |
|---------------------------------------------------|-------------------------------------------------|
| `tctl install cluster-service-account`            | n/a — auth is a static **JWT + CA** from XCP e2e certs |
| `tctl install manifest control-plane` → secrets   | Copy 2 Secrets (`xcp-central-auth-ca`, `xcp-central-auth-jwt`) |
| `ControlPlane` CR + `tsb-operator-control-plane`  | `EdgeXcp` CR + `xcp-operator-edge` (no tsb-operator) |
| Image pull secret from `containers.dl.tetrate.io` | Local kind registry `localhost:5000` (no auth) |
| `cert-manager` rendered by tsb-operator           | `cert-manager` installed directly from XCP e2e manifest |
| MP host = `*.cloud.tetrate.com:443`               | MP host = `<xcp-central LB IP>:9080`            |
| Per-SA `imagePullSecrets` patching (private reg.) | Not needed — `localhost:5000` is unauthenticated |

## High-level flow

```
Pick next free index (cluster-2)
         ↓
Create kind cluster + metrics-server + metallb
         ↓
Connect cluster-2's docker node to cluster-0 / cluster-1
(pod-subnet routes + MetalLB IP routes; both directions)
         ↓
Install cert-manager (XCP e2e manifest)
         ↓
Copy `xcp-central-auth-ca` + `xcp-central-auth-jwt` from cluster-1 → cluster-2
         ↓
Apply edge RBAC + xcp-operator-edge Deployment
         ↓
Apply EdgeXcp CR (centralHost = xcp-central LB on cluster-0:9080, centralSni = e2e-test-xcp-central.tetrate.io)
         ↓
Register cluster on Central:  apply `Cluster` CR in `xcp-system` on cluster-0
         ↓
Wait for `edge` Deployment Ready and `istiod-stable` Ready
```

## Step 0 — Environment

```bash
# Pin these once per shell.
export NEW_IDX=2                              # next free cluster index (0,1 already used)
export NEW_CLUSTER="cluster-${NEW_IDX}"
export NEW_KUBECONFIG="$HOME/.kube/${NEW_CLUSTER}"

# Existing setup
export CENTRAL_IDX=0
export CENTRAL_KUBECONFIG="$HOME/.kube/cluster-${CENTRAL_IDX}"
export EXISTING_EDGE_KUBECONFIG="$HOME/.kube/cluster-1"   # any working edge

export XCP_DIR="$HOME/dev/tetrateio/xcp"
export HUB="localhost:5000"
export TAG="ashish"                           # whatever the existing edge uses

export XCP_NAMESPACE="xcp-system"
export ISTIO_NAMESPACE="istio-system"
export XCP_MULTICLUSTER_NAMESPACE="xcp-multicluster"

# Topology — these MUST match XCP e2e's per-index assignments.
# Source: $XCP_DIR/test/e2e/scripts/lib.sh   (CLUSTER_POD_SUBNETS / CLUSTER_SVC_SUBNETS)
export POD_SUBNET=10.30.0.0/16                # idx 2 (idx 3 = 10.40.0.0/16)
export SVC_SUBNET=10.255.30.0/24              # idx 2 (idx 3 = 10.255.40.0/24)
export REGION=us-west1                        # idx 2 differs from idx 0/1 on purpose
export ZONE=us-west1-A
export SUBZONE=subzone-1
```

Capture XCP central's reachable address (cluster-0 publishes `xcp-central`
as a MetalLB LoadBalancer — this is the address the new edge dials):

```bash
KUBECONFIG="$CENTRAL_KUBECONFIG" kubectl -n "$XCP_NAMESPACE" \
  get svc xcp-central -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
# e.g. 172.18.255.155 — used as central host below
export CENTRAL_LB_IP="$(KUBECONFIG=$CENTRAL_KUBECONFIG kubectl -n $XCP_NAMESPACE \
  get svc xcp-central -o jsonpath='{.status.loadBalancer.ingress[0].ip}')"
```

## Step 1 — Create the kind cluster

Two options. Both produce `~/.kube/cluster-${NEW_IDX}` with the docker
container IP substituted in (so the host can `kubectl` against it).

### 1a. Fast path: re-run XCP's kind setup with NUM_CLUSTERS bumped

> **Caveat:** XCP's `create-kind-clusters.sh` runs `kind delete cluster` for
> each name before creating it — **it will recreate cluster-0 and cluster-1
> too**, wiping their state. Only do this if you're OK starting over. For an
> *additive* onboard, use 1b.

```bash
NUM_CLUSTERS=$((NEW_IDX + 1)) make -C "$XCP_DIR/test/e2e" kind-clusters
```

### 1b. Additive path (recommended) — bring up only the new cluster

```bash
# 1) Write the kind config (mirrors what XCP e2e renders for idx=NEW_IDX).
KIND_CFG="/tmp/kind-${NEW_CLUSTER}.yaml"
cat >"$KIND_CFG" <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  labels:
    topology.kubernetes.io/region: ${REGION}
    topology.kubernetes.io/zone: ${ZONE}
    topology.istio.io/subzone: ${SUBZONE}
kubeadmConfigPatches:
  - |
    apiVersion: kubeadm.k8s.io/v1beta4
    kind: ClusterConfiguration
    apiServer:
      extraArgs:
        - name: service-account-issuer
          value: "kubernetes.default.svc"
        - name: service-account-signing-key-file
          value: "/etc/kubernetes/pki/sa.key"
    etcd:
      local:
        dataDir: "/tmp/lib/etcd"
containerdConfigPatches:
  - |-
    [plugins."io.containerd.grpc.v1.cri".registry]
      config_path = "/etc/containerd/certs.d"
networking:
  podSubnet: ${POD_SUBNET}
  serviceSubnet: ${SVC_SUBNET}
EOF

# 2) Create the cluster.
kind create cluster --name "$NEW_CLUSTER" --config "$KIND_CFG" --wait 5m

# 3) Wire the local docker registry into containerd's certs.d, same as XCP e2e.
REG_PORT=5000
for node in $(kind get nodes --name="$NEW_CLUSTER"); do
  docker exec "$node" mkdir -p "/etc/containerd/certs.d/localhost:${REG_PORT}"
  cat <<EOF | docker exec -i "$node" cp /dev/stdin "/etc/containerd/certs.d/localhost:${REG_PORT}/hosts.toml"
[host."http://local-docker-registry:5000"]
EOF
done
docker network connect kind local-docker-registry 2>/dev/null || true

# 4) Substitute the docker container IP into the kubeconfig (so kubectl from
#    the host works without --internal). XCP e2e does the same step.
CONTAINER_IP=$(docker inspect "${NEW_CLUSTER}-control-plane" \
  --format '{{ .NetworkSettings.Networks.kind.IPAddress }}')
kind get kubeconfig --name "$NEW_CLUSTER" \
  | sed "s/${NEW_CLUSTER}-control-plane/${CONTAINER_IP}/g" >"$NEW_KUBECONFIG"

KUBECONFIG="$NEW_KUBECONFIG" kubectl get nodes
```

## Step 2 — Metrics-server + MetalLB

The edge needs a LoadBalancer (for east-west gateway later) and metrics-server
makes the cluster feel like the existing ones.

```bash
KUBECONFIG="$NEW_KUBECONFIG" kubectl apply \
  -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.8.1/components.yaml
KUBECONFIG="$NEW_KUBECONFIG" kubectl -n kube-system patch deployment metrics-server --type=json \
  --patch='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'

KUBECONFIG="$NEW_KUBECONFIG" kubectl apply \
  -f https://raw.githubusercontent.com/metallb/metallb/v0.15.3/config/manifests/metallb-native.yaml
KUBECONFIG="$NEW_KUBECONFIG" kubectl wait --for condition=established --timeout=60s crd/ipaddresspools.metallb.io
KUBECONFIG="$NEW_KUBECONFIG" kubectl -n metallb-system rollout status deploy/controller --timeout=10m
```

Pick a 20-IP slice for the new cluster from the kind docker subnet that
isn't already claimed by cluster-0/cluster-1. The simplest way is to look
at what the existing clusters got and pick the next free chunk:

```bash
# What's already taken
for i in 0 1; do
  KUBECONFIG="$HOME/.kube/cluster-$i" kubectl -n metallb-system \
    get ipaddresspool default -o jsonpath='{.spec.addresses}{"\n"}'
done
# e.g. ["172.18.255.155-172.18.255.174"] and ["172.18.255.175-172.18.255.194"]
# → pick 172.18.255.195-172.18.255.214 for cluster-2.

LB_RANGE="172.18.255.195-172.18.255.214"     # adjust based on what's free

cat <<EOF | KUBECONFIG="$NEW_KUBECONFIG" kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata: { name: default, namespace: metallb-system }
spec: { addresses: ["$LB_RANGE"] }
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata: { name: default, namespace: metallb-system }
spec: { ipAddressPools: [default] }
EOF
```

## Step 3 — Cross-cluster routing

For the new cluster to reach `xcp-central`'s LoadBalancer IP (which lives on
the cluster-0 docker network) and for east-west traffic to work, the docker
nodes need static routes between each other's pod and LB subnets. XCP e2e
does this in `connect_kind_clusters` in `lib.sh`. Repeat that wiring for
the new pair.

```bash
NEW_NODE="${NEW_CLUSTER}-control-plane"
NEW_DOCKER_IP=$(docker inspect -f '{{ .NetworkSettings.Networks.kind.IPAddress }}' "$NEW_NODE")
NEW_LB_RANGE="$LB_RANGE"
NEW_POD_SUBNET="$POD_SUBNET"

for PEER in cluster-0 cluster-1; do
  PEER_NODE="${PEER}-control-plane"
  PEER_KCFG="$HOME/.kube/${PEER}"
  PEER_DOCKER_IP=$(docker inspect -f '{{ .NetworkSettings.Networks.kind.IPAddress }}' "$PEER_NODE")
  PEER_POD_SUBNET=$(KUBECONFIG="$PEER_KCFG" kubectl get nodes -o jsonpath='{.items[0].spec.podCIDR}')
  PEER_LB_ADDRS=$(KUBECONFIG="$PEER_KCFG" kubectl -n metallb-system get ipaddresspool default -o jsonpath='{.spec.addresses[0]}')
  # PEER_LB_ADDRS looks like "172.18.255.155-172.18.255.174"
  PEER_LB_START="${PEER_LB_ADDRS%-*}"
  PEER_LB_END="${PEER_LB_ADDRS#*-}"

  # Pod subnet routes (both directions)
  docker exec "$NEW_NODE"  ip route add "$PEER_POD_SUBNET" via "$PEER_DOCKER_IP" 2>/dev/null || true
  docker exec "$PEER_NODE" ip route add "$NEW_POD_SUBNET"  via "$NEW_DOCKER_IP"   2>/dev/null || true

  # LB IP routes — summarize the start-end range into one or more CIDRs.
  while read -r CIDR; do
    docker exec "$NEW_NODE" ip route add "$CIDR" via "$PEER_DOCKER_IP" 2>/dev/null || true
  done < <(python3 -c "
from ipaddress import summarize_address_range, IPv4Address
for n in summarize_address_range(IPv4Address('$PEER_LB_START'), IPv4Address('$PEER_LB_END')):
    print(n.compressed)
")
  # And the reverse for the new cluster's pool
  NEW_LB_START="${NEW_LB_RANGE%-*}"
  NEW_LB_END="${NEW_LB_RANGE#*-}"
  while read -r CIDR; do
    docker exec "$PEER_NODE" ip route add "$CIDR" via "$NEW_DOCKER_IP" 2>/dev/null || true
  done < <(python3 -c "
from ipaddress import summarize_address_range, IPv4Address
for n in summarize_address_range(IPv4Address('$NEW_LB_START'), IPv4Address('$NEW_LB_END')):
    print(n.compressed)
")
done
```

**Sanity check** before going further — from the new cluster's node, you
should be able to ping the central LB IP:

```bash
docker exec "${NEW_CLUSTER}-control-plane" ping -c1 "$CENTRAL_LB_IP"
```

If this fails, fix the routes; the edge will never connect otherwise.

## Step 4 — cert-manager

XCP e2e bundles a pinned cert-manager manifest. Use the same one.

```bash
KUBECONFIG="$NEW_KUBECONFIG" kubectl apply \
  -f "$XCP_DIR/pkg/test/framework/components/operator/central/templates/0-cert-manager.yaml"
KUBECONFIG="$NEW_KUBECONFIG" kubectl -n cert-manager rollout status deploy/cert-manager           --timeout=8m
KUBECONFIG="$NEW_KUBECONFIG" kubectl -n cert-manager rollout status deploy/cert-manager-cainjector --timeout=8m
KUBECONFIG="$NEW_KUBECONFIG" kubectl -n cert-manager rollout status deploy/cert-manager-webhook    --timeout=8m
```

## Step 5 — Namespaces + auth secrets (copied from an existing edge)

The original deploy script generates the central CA + JWT from
`$XCP_DIR/pkg/test/framework/components/operator/certs/jwt/central-ca.crt`
and a Go helper (`central.CreateJWT`). Those values are identical for every
edge in this setup, so the simplest path is to **copy the two Secrets** from
`cluster-1` (or any working edge) — no Go build required.

```bash
KUBECONFIG="$NEW_KUBECONFIG" kubectl create ns "$ISTIO_NAMESPACE"            --dry-run=client -o yaml | KUBECONFIG="$NEW_KUBECONFIG" kubectl apply -f -
KUBECONFIG="$NEW_KUBECONFIG" kubectl create ns "$XCP_MULTICLUSTER_NAMESPACE" --dry-run=client -o yaml | KUBECONFIG="$NEW_KUBECONFIG" kubectl apply -f -

for SEC in xcp-central-auth-ca xcp-central-auth-jwt; do
  KUBECONFIG="$EXISTING_EDGE_KUBECONFIG" kubectl -n "$ISTIO_NAMESPACE" get secret "$SEC" -o yaml \
    | grep -vE '^\s*(uid|resourceVersion|creationTimestamp|namespace|ownerReferences|annotations):' \
    | sed '/^  ownerReferences:/,/^[^ ]/d' \
    | KUBECONFIG="$NEW_KUBECONFIG" kubectl -n "$ISTIO_NAMESPACE" apply -f -
done
```

If you'd rather regenerate them from scratch, see
[../backup/deploy-xcp-multicluster.sh](../backup/deploy-xcp-multicluster.sh)
functions `generate_edge_token` and `render_edge_bootstrap` (it base64s
`certs/jwt/central-ca.crt` and runs a tiny Go program to emit a JWT).

## Step 6 — Edge RBAC + operator

The repo's e2e ships a Helm-templated RBAC manifest; strip the templates and
apply.

```bash
sed "s|{{ .Values.CustomIstioNamespace }}|$ISTIO_NAMESPACE|g" \
  "$XCP_DIR/pkg/test/framework/components/operator/edge/templates/3-edge-certs-and-rbac.yaml" \
  | grep -v '{{' \
  | KUBECONFIG="$NEW_KUBECONFIG" kubectl apply -f -

KUBECONFIG="$NEW_KUBECONFIG" kubectl -n "$ISTIO_NAMESPACE" wait \
  --for=condition=Ready --timeout=5m secret/xcp-edge-selfsigned-ca

cat <<EOF | KUBECONFIG="$NEW_KUBECONFIG" kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: xcp-operator-edge
  namespace: $ISTIO_NAMESPACE
  labels: { app: xcp-operator }
spec:
  ports:
    - { name: http-metrics, port: 8383, targetPort: 8383 }
    - { name: webhook,      port: 443,  targetPort: 8443 }
  selector: { app: xcp-operator }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: xcp-operator-edge
  namespace: $ISTIO_NAMESPACE
  labels: { app: xcp-operator }
spec:
  replicas: 1
  selector: { matchLabels: { app: xcp-operator } }
  template:
    metadata:
      labels: { app: xcp-operator }
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port:   "8080"
        prometheus.io/path:   /metrics
    spec:
      serviceAccountName: xcp-operator-edge
      containers:
        - name: xcp-operator
          image: $HUB/xcp-operator:$TAG
          imagePullPolicy: Always
          args: [edge-xcp, --deployment-name, xcp-operator-edge, --log_output_level, all:debug]
          env:
            - { name: ENABLE_ISTIOD_AND_GATEWAY_RECONCILIATION, value: "true" }
            - { name: LEADER_ELECTION_NAMESPACE, valueFrom: { fieldRef: { fieldPath: metadata.namespace } } }
            - { name: POD_NAME,                  valueFrom: { fieldRef: { fieldPath: metadata.name } } }
            - { name: POD_NAMESPACE,             valueFrom: { fieldRef: { fieldPath: metadata.namespace } } }
            - { name: OPERATOR_NAME,             value: xcp-operator }
            - { name: CSR_SIGNER,                value: issuers.cert-manager.io/$ISTIO_NAMESPACE.xcp-edge-csr-signer }
            - { name: CA_SECRET_NAME,            value: xcp-edge-selfsigned-ca }
            - { name: CERT_PROVIDER,             value: cert-manager }
            - { name: ACCESS_LOG_FLUSH_INTERVAL, value: 100ms }
          volumeMounts:
            - { name: tmp,       mountPath: /tmp }
            - { name: root-cert, mountPath: /var/run/secrets/xcp-operator-edge-webhook-ca, readOnly: true }
      volumes:
        - { name: tmp, emptyDir: {} }
        - { name: root-cert, secret: { secretName: xcp-edge-selfsigned-ca } }
EOF

KUBECONFIG="$NEW_KUBECONFIG" kubectl -n "$ISTIO_NAMESPACE" rollout status deployment/xcp-operator-edge --timeout=8m
KUBECONFIG="$NEW_KUBECONFIG" kubectl wait --for=condition=established --timeout=5m crd/edgexcps.install.xcp.tetrate.io
```

## Step 7 — EdgeXcp CR

This is the moment the edge actually dials Central.

```bash
cat <<EOF | KUBECONFIG="$NEW_KUBECONFIG" kubectl apply -f -
apiVersion: install.xcp.tetrate.io/v1alpha1
kind: EdgeXcp
metadata:
  namespace: $ISTIO_NAMESPACE
  name: edgexcp
spec:
  hub: $HUB
  logLevels: all:debug
  xcpEdgeClusterName: $NEW_CLUSTER
  xcpCentralHost: "${CENTRAL_LB_IP}:9080"
  xcpMulticlusterNamespace: $XCP_MULTICLUSTER_NAMESPACE
  centralAuthMode: JWT
  centralAuthJwt:
    secret: xcp-central-auth-jwt
    centralCaSecret: xcp-central-auth-ca
    centralSni: e2e-test-xcp-central.tetrate.io   # **inside** centralAuthJwt, NOT at spec level
  istioTrustDomain: cluster.local
  tier1Cluster: false
  isolationBoundaries:
    - name: global
      revisions:
        - { name: stable }
  enableHttpMeshInternalIdentityPropagation: true
  components:
    istio:
      centralProvidedCaCert: true
      enableConfigStatus: true
      trustDomain: cluster.local
    edgeServer:
      kubeSpec:
        deployment:
          env:
            - { name: XCP_DEBOUNCE_AFTER,                value: 1s }
            - { name: XCP_DEBOUNCE_MAX,                  value: 10s }
            - { name: XCP_CLUSTER_STATE_DEBOUNCE_AFTER,  value: 1s }
            - { name: XCP_CLUSTER_STATE_DEBOUNCE_MAX,    value: 10s }
            - { name: ENABLE_GATEWAY_CONFIG_DIAGNOSTIC_INFO, value: "true" }
            - { name: ENABLE_XCP_CONFIG_STATUS,          value: "true" }
            - { name: ENABLE_ENHANCED_EAST_WEST_ROUTING, value: "true" }
        overlays:
          - kind: Deployment
            name: edge
            patches:
              - { path: spec.template.spec.containers.[name:edge].image, value: $HUB/xcpd:$TAG }
              - { path: spec.template.spec.containers.[name:edge].imagePullPolicy, value: Always }
EOF

KUBECONFIG="$NEW_KUBECONFIG" kubectl -n "$ISTIO_NAMESPACE" rollout status deployment/edge          --timeout=10m
KUBECONFIG="$NEW_KUBECONFIG" kubectl -n "$ISTIO_NAMESPACE" rollout status deployment/istiod-stable --timeout=10m
```

## Step 8 — Register the cluster on Central

Central keeps an inventory in `xcp-system`. Adding the new cluster's CR
there is what flips it from "edge dialed in" to "edge participates in
multi-cluster". Existing entries:

```bash
KUBECONFIG="$CENTRAL_KUBECONFIG" kubectl -n "$XCP_NAMESPACE" get cluster
# NAME        AGE
# cluster-0   ...
# cluster-1   ...
```

Add `cluster-2`:

```bash
cat <<EOF | KUBECONFIG="$CENTRAL_KUBECONFIG" kubectl apply -f -
apiVersion: xcp.tetrate.io/v2
kind: Cluster
metadata:
  name: $NEW_CLUSTER
  namespace: $XCP_NAMESPACE
spec:
  istioTrustDomain: cluster.local
  network: nw-${NEW_CLUSTER}
EOF
```

## Step 9 — Verify

```bash
# 1) Edge talked to Central — XCP-managed `Cluster` row appears in istio-system
KUBECONFIG="$NEW_KUBECONFIG" kubectl -n "$ISTIO_NAMESPACE" get cluster
# Expect: cluster-0, cluster-1, cluster-2 (all three known)

# 2) Central sees the new edge in its cluster state
KUBECONFIG="$CENTRAL_KUBECONFIG" kubectl -n "$ISTIO_NAMESPACE" get cluster $NEW_CLUSTER -o yaml \
  | grep -E "discoveredLocality|istioVersions|xcpVersions"

# 3) Edge logs show registration_with_central reasons being processed
KUBECONFIG="$NEW_KUBECONFIG" kubectl -n "$ISTIO_NAMESPACE" logs deploy/edge --tail=80 \
  | grep -iE "central|register|cluster-state"

# 4) Pods Running on the new cluster
KUBECONFIG="$NEW_KUBECONFIG" kubectl -n "$ISTIO_NAMESPACE" get pods
# Expect: edge-*, istiod-stable-*, xcp-operator-edge-* all Running
```

## Learnings while putting this together

These are the things that bit me / surprised me and aren't obvious from the
existing scripts. Updating as I go.

1. **MP-style `~/backup/onboarding.md` doesn't transfer cleanly.** No `tctl`,
   no `tsb-operator-control-plane`, no `containers.dl.tetrate.io` image pull
   secret, no `ControlPlane` CR — the local setup is *XCP-only*. Mapping in
   the table at the top of this doc.

2. **`make kind-clusters` is destructive on existing clusters.** XCP's
   `setup_kind_cluster` calls `kind delete cluster --name ${NAME}` for **every
   index up to NUM_CLUSTERS** before creating it. If cluster-0/cluster-1
   are healthy, going via `NUM_CLUSTERS=3 make kind-clusters` will recreate
   them and you lose all in-cluster state. Use the additive path (1b).

3. **Per-cluster index has fixed subnet/region assignments.** They live in
   `$XCP_DIR/test/e2e/scripts/lib.sh`:
   - idx 0: pod `10.10.0.0/16`, svc `10.255.10.0/24`, `us-east1 / us-east1-A / subzone-1`
   - idx 1: pod `10.20.0.0/16`, svc `10.255.20.0/24`, `us-east1 / us-east1-B / subzone-x`
   - idx 2: pod `10.30.0.0/16`, svc `10.255.30.0/24`, `us-west1 / us-west1-A / subzone-1`
   - idx 3: pod `10.40.0.0/16`, svc `10.255.40.0/24`, `us-east1 / us-east1-A / subzone-2`
   You'll *probably* get away with deviating, but the existing demo scenarios
   may rely on the locality labels (esp. cluster-2 being the only `us-west1`).

4. **Hard cap of 4 clusters.** `MAX_CLUSTERS=4` in `env.sh`. Past that you
   need to extend `CLUSTER_POD_SUBNETS` / `CLUSTER_SVC_SUBNETS` and pick
   another MetalLB slice.

5. **MetalLB IP pools are not auto-partitioned.** The XCP setup script
   assigns 20 IPs from the end of the kind docker subnet to each cluster
   in sequence, but it does that *in one script invocation* via a shared
   `METALLB_IPS` array. When you add a cluster later, you have to **read
   what's already taken** and pick the next free 20-IP block yourself
   (Step 2). Don't reuse a range — MetalLB will hand out duplicates and
   ARP will break.

6. **Static routes are per-pair, manually maintained.** kind clusters share
   the `kind` docker network but NOT each other's pod CIDRs or LB ranges.
   You add `ip route` entries on each node's docker container directly. New
   cluster needs N×2 pairs of routes (pod subnet + LB CIDRs each way) for
   every existing cluster. Restarting the docker container drops these —
   they're not persisted anywhere.

7. **`xcp-central-auth-ca` / `xcp-central-auth-jwt` are shared static
   material.** They come from
   `$XCP_DIR/pkg/test/framework/components/operator/certs/jwt/central-ca.crt`
   and a hardcoded JWT subject (`central.CentralAuthJwtSubject`). Every edge
   gets the same two values — so `kubectl get secret … -o yaml | apply` from
   an existing edge is the lowest-effort way to seed the new one. No need
   to rebuild the Go helper.

8. **`centralSni` is mandatory for remote edges (anything not co-located
   with central).** The cert `certs/jwt/central.crt` is issued for
   `e2e-test-xcp-central.tetrate.io`; without setting that SNI the TLS
   handshake fails ("certificate is valid for e2e-test-xcp-central.tetrate.io,
   not 172.18.255.155"). The co-resident edge on cluster-0 uses the
   `xcp-central.$XCP_NAMESPACE.svc.cluster.local:9080` form and doesn't
   need SNI.

9. **`xcp-central` is a `LoadBalancer` Service.** Its IP comes from
   cluster-0's MetalLB pool; if you re-roll cluster-0's MetalLB pool the
   IP changes and every edge's `EdgeXcp.spec.xcpCentralHost` becomes stale.
   Re-apply the edge CR with the new IP — the EdgeXcp controller picks it
   up without restart of the operator.

10. **No image pull secret needed.** The local registry
    `localhost:5000` (the `local-docker-registry` docker container) is
    unauthenticated. The MP doc spends a lot of ink on docker-credential
    helpers and per-SA `imagePullSecrets`; none of that applies here, as
    long as kind's containerd is wired up via the `containerdConfigPatches`
    + `certs.d/hosts.toml` dance in Step 1.

11. **Images must already exist in the local registry.** `make
    docker.xcpd` / `make docker.xcp-operator` (from $XCP_DIR) push to
    `localhost:5000`. If the existing clusters were built with `TAG=ashish`,
    that tag is already in the local registry — the new kind cluster will
    pull successfully. Otherwise re-run `make -C $XCP_DIR docker.xcpd
    docker.xcp-operator HUB=localhost:5000 TAG=$TAG` first.

12. **`Cluster` CR exists in two places.** Central inventory is
    `xcp-system/<cluster-name>` (you create this — Step 8). The
    `istio-system/<cluster-name>` ones on each edge are *managed by xcp-edge*
    — don't touch them, they hold the gzip-encoded cluster state pushed by
    Central.

## Things that bit me during the real cluster-2 run (2026-05-27)

1. **`centralSni` lives inside `centralAuthJwt`, NOT at `spec.` level.** I
    initially copied it to `spec.centralSni` in the CR — the operator
    accepted the apply (the field was silently dropped by the schema), then
    dialed central with `ServerName: "<IP>"` and the TLS handshake exploded
    with `x509: cannot validate certificate for 172.18.255.155 because it
    doesn't contain any IP SANs`. The deploy script renders it nested under
    `centralAuthJwt:` and so should anything else. Fixed in Step 7 above.

2. **YAML flow-style mappings with image refs are a trap.** Writing
    `{ path: ..., value: localhost:5000/xcpd:ashish }` as a one-line flow
    map breaks the YAML parser on the colons. Use block style for the
    `overlays:` patches.

3. **`kind create cluster` writes to whatever `$KUBECONFIG` is exported in
    the shell.** My parent shell had `KUBECONFIG=~/.kube/cluster-1`
    exported, so every `kind create cluster --name cluster-2` *merged
    cluster-2's entries into cluster-1's kubeconfig and switched its
    current-context to cluster-2*. After that, `kubectl --kubeconfig
    ~/.kube/cluster-1 ...` was silently hitting cluster-2. Always run kind
    with `KUBECONFIG=/tmp/scratch.kubeconfig` set inline, then write the
    real per-cluster file via `kind get kubeconfig --name <cluster> | sed
    .../<ip>/ > ~/.kube/<cluster>`. If you suspect pollution, regenerate
    `~/.kube/cluster-*` from kind — the container-IP substitution is
    idempotent.

4. **Existing cluster-1's pod CIDR is `10.20.0.0/16` (XCP idx-1), not
    `10.30.0.0/16` as the polluted kubeconfig suggested.** Confirmed
    cluster-0 and cluster-1 do follow `lib.sh`'s idx-0/idx-1 assignment.
    The earlier doc paragraph on "the existing setup deviates from
    `lib.sh`" was wrong — it was the polluted kubeconfig fooling me. Use
    `kind get kubeconfig --name <c> --internal | grep -A1 cluster: |
    grep server` or `kubectl cluster-info dump` against a freshly-pulled
    kubeconfig to ground-truth a cluster's CIDR.

5. **kind 0.30.0 defaults to node image `kindest/node:v1.34.0`**, while
    the existing cluster-0/1 are on `v1.33.7`. Onboarding worked anyway
    (the edge and operator are version-agnostic w.r.t. the k8s API
    surface they hit). If you want a matched fleet, pass `--image
    docker.io/kindest/node:v1.33.7` on `kind create cluster`.

6. **First failure after edge dials central is `secret
    "istio-intermediate-ca-<cluster>" not found`.** Central can't mint
    the intermediate Istio CA until the cluster has a `Cluster` row in
    its inventory. **Order matters**: get the EdgeXcp CR applied first
    (it has to be there for the operator to even try connecting), THEN
    register the cluster on central (Step 8). The operator retries every
    few seconds, so the lag between Step 7 and Step 8 just delays the
    edge Deployment from being rendered — no manual restart needed.

7. **Cross-cluster route additions are silent on success and noisy on
    no-op.** `ip route add` returns "RTNETLINK answers: File exists"
    when a route is already there; that's a normal idempotent state, not
    a failure. The script's `|| true` pattern is intentional — don't
    bail on it.

8. **Use `bash -c 'echo > /dev/tcp/...'` for in-container connectivity
    checks**, not `sh`. The kind node image is dash-based and `/dev/tcp`
    is a bashism. `ping` isn't installed either.

## Quick reference — copy-paste skeleton

```bash
NEW_IDX=2
NEW_CLUSTER="cluster-$NEW_IDX"
NEW_KUBECONFIG=$HOME/.kube/$NEW_CLUSTER
EXISTING_EDGE_KUBECONFIG=$HOME/.kube/cluster-1
CENTRAL_KUBECONFIG=$HOME/.kube/cluster-0
XCP_DIR=$HOME/dev/tetrateio/xcp
HUB=localhost:5000
TAG=ashish

CENTRAL_LB_IP=$(KUBECONFIG=$CENTRAL_KUBECONFIG kubectl -n xcp-system get svc xcp-central -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# 1. Create kind cluster        (Step 1b)
# 2. metrics-server + MetalLB   (Step 2)
# 3. Cross-cluster routes       (Step 3)  ← critical, easy to forget
# 4. cert-manager               (Step 4)
# 5. Copy 2 auth Secrets        (Step 5)
# 6. RBAC + xcp-operator-edge   (Step 6)
# 7. EdgeXcp CR                 (Step 7)
# 8. Cluster CR on Central      (Step 8)
# 9. Verify                     (Step 9)
```
