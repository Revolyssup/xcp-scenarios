# Minimal diagnostic dump

- Hostname: `echo-term.tetrate.io`
- Namespace: `echo-term`
- Service name: `echo-term`
- Cluster role: **workload** (local Service echo-term/echo-term with 1 backing pod(s))
- Collected: 2026-05-25T10:52:45Z

## What was found

- Local Service: yes
- Backing pods: 1 [echo-term/echo-term-58c79569c6-kkjfl]
- Gateway pods: 1 [echo-term/echo-term-gateway-6bc8b5ffdd-v9xkm]
- Istiod pods: 1 [istio-system/istiod-stable-b4c7c467b-2sqsx]
- XCP edge pods: 1 [istio-system/edge-bdb796b6f-xxdp2]
- Matched Istio CRs: 18
  - VirtualService echo-term/vs-echo-term-tetrate-io
  - Gateway echo-term/echo-t2-term-gw
  - DestinationRule xcp-multicluster/global-gateway-echo-term-tetrate-io
  - ServiceEntry xcp-multicluster/global-gateway-echo-term-tetrate-io
  - EnvoyFilter echo-term/default-allow-echo-t2-term-gw-external-batch0
  - EnvoyFilter echo-term/default-upstream-traffic-setting
  - EnvoyFilter echo-term/tier2-term-gg-default-upstream-traffic-setting
  - EnvoyFilter istio-system/global-forward-xfcc-15443
  - EnvoyFilter istio-system/global-gw-best-practice
  - EnvoyFilter istio-system/global-mx-downstream-egress-gateway-filter
  - EnvoyFilter istio-system/global-mx-downstream-gateway-filter
  - EnvoyFilter istio-system/global-periodical-access-log
  - EnvoyFilter istio-system/global-request-id-propagation
  - EnvoyFilter istio-system/global-tcp-keepalive-envoy-filter
  - EnvoyFilter istio-system/global-xfcc-extractor-sidecar
  - EnvoyFilter istio-system/global-xfcc-guard-15443
  - IstioOperator istio-system/xcp-iop-stable
  - IstioOperator istio-system/xcpgw-echo-term-gateway
- XCP config CRs: 5
  - Workspace istio-system/term-ws
  - GlobalSetting istio-system/global
  - GatewayGroup istio-system/tier2-term-gg
  - IngressGateway istio-system/echo-t2-term-gw
  - EdgeXcp istio-system/edgexcp

## Revisions

Captured istiod replicas (filtered to revisions in use by this hostname's data-plane / gateway pods):

- `default`
- `stable`

## Collectors

- `envoy` (01-envoy): ok, 10 file(s)
- `istio-config` (02-istio-config): ok, 18 file(s)
- `istiod` (05-istiod): ok, 9 file(s)
- `k8s` (04-k8s): ok, 4 file(s)
- `xcp-config` (03-xcp-config): ok, 5 file(s)
- `xcp-edge` (06-xcp-edge): ok, 9 file(s)

## Gold-thread hints

1. `01-envoy/<pod>/stats.txt` — non-zero `no_cluster_found` points at a missing route/cluster (the classic 503); `upstream_cx_connect_fail`, `upstream_cx_none_healthy` point at L4/TCP upstream failure.
2. `01-envoy/<pod>/clusters.json` — search for the destination cluster; absent cluster or endpoints with `health_flags` set explains UH/UF.
3. `01-envoy/<pod>/config_dump.json` — route config and listener filter-chains; for tier-1 this is the authoritative cross-cluster artifact.
4. `02-istio-config/` — the raw Istio CRs that shape routing for this host.
5. `03-xcp-config/` — the XCP edge config CRs (`*.xcp.tetrate.io`): the Workspace and *Group containers plus the XCP gateway/routing intent. Empty if XCP has not materialised config on this cluster — cross-check the edge's live applied config in `06-xcp-edge/debug-appliedconfigz.json`.
6. `06-xcp-edge/debug-gateways.json` — whether the XCP edge knows the host.
