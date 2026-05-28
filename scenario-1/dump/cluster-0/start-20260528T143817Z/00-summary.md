# Minimal diagnostic dump

- Hostname: `echo.tetrate.io`
- Namespace: `echo`
- Service name: `echo`
- Cluster role: **tier1** (matched Tier1Gateway CR istio-system/tier1-echo-gw)
- Collected: 2026-05-28T14:38:17Z

## What was found

- Local Service: no
- Backing pods: 0 []
- Gateway pods: 1 [tier1/tier1-gateway-596d499c9c-f67n9]
- Istiod pods: 1 [istio-system/istiod-stable-65bd95ccc-pqdvh]
- XCP edge pods: 1 [istio-system/edge-668cdd6dc9-r9jpw]
- Matched Istio CRs: 15
  - VirtualService tier1/tier1-echo-gw-echo
  - Gateway tier1/tier1-echo-gw
  - DestinationRule tier1/tier1-tier1-echo-gwecho-tetrate-io
  - ServiceEntry xcp-multicluster/global-gateway-echo-tetrate-io
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
  - IstioOperator istio-system/xcpgw-tier1-gateway
- XCP config CRs: 11
  - Workspace istio-system/passthrough-ws
  - Workspace xcp-system/passthrough-ws
  - GlobalSetting istio-system/global
  - GlobalSetting xcp-system/global
  - GatewayGroup istio-system/tier1-gg
  - GatewayGroup xcp-system/tier1-gg
  - GatewayGroup xcp-system/tier2-gg
  - IngressGateway xcp-system/echo-t2-gw
  - Tier1Gateway istio-system/tier1-echo-gw
  - Tier1Gateway xcp-system/tier1-echo-gw
  - EdgeXcp istio-system/edgexcp

## Revisions

Captured istiod replicas (filtered to revisions in use by this hostname's data-plane / gateway pods):

- `stable`

## Tier-1 cross-cluster note

NOTE: tier-1 rewrites the HTTP authority — confirm the real tier-2 hostname in `01-envoy/<tier1-pod>/config_dump.json` route config (look at `route.rewrite.authority` and the `outbound|...` cluster name) before re-running the dump there. TCP/TLS-passthrough is SNI-routed and not rewritten.

## Collectors

- `envoy` (01-envoy): ok, 5 file(s)
- `istio-config` (02-istio-config): ok, 15 file(s)
- `istiod` (05-istiod): ok, 8 file(s)
- `k8s` (04-k8s): ok, 1 file(s)
- `xcp-config` (03-xcp-config): ok, 11 file(s)
- `xcp-edge` (06-xcp-edge): ok, 9 file(s)

## Gold-thread hints

1. `01-envoy/<pod>/stats.txt` — non-zero `no_cluster_found` points at a missing route/cluster (the classic 503); `upstream_cx_connect_fail`, `upstream_cx_none_healthy` point at L4/TCP upstream failure.
2. `01-envoy/<pod>/clusters.json` — search for the destination cluster; absent cluster or endpoints with `health_flags` set explains UH/UF.
3. `01-envoy/<pod>/config_dump.json` — route config and listener filter-chains; for tier-1 this is the authoritative cross-cluster artifact.
4. `02-istio-config/` — the raw Istio CRs that shape routing for this host.
5. `03-xcp-config/` — the XCP edge config CRs (`*.xcp.tetrate.io`): the Workspace and *Group containers plus the XCP gateway/routing intent. Empty if XCP has not materialised config on this cluster — cross-check the edge's live applied config in `06-xcp-edge/debug-appliedconfigz.json`.
6. `06-xcp-edge/debug-gateways.json` — whether the XCP edge knows the host.
