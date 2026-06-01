# Minimal diagnostic dump

- Hostname: `echo.echo-tr.svc.cluster.local`
- Namespace: `echo-tr`
- Service name: `echo`
- Cluster role: **tier1** (matched unified Gateway CR xcp-system/transit-mid-gw acting as tier-1 (http[transit-hop] transit/trafficMode=TRANSIT) serving echo.echo-tr.svc.cluster.local)
- Collected: 2026-06-01T10:51:33Z

## What was found

- Local Service: no
- Backing pods: 0 []
- Gateway pods: 0 []
- Istiod pods: 1 [istio-system/istiod-stable-65bd95ccc-pqdvh]
- XCP edge pods: 1 [istio-system/edge-56555777bb-hpwq6]
- Matched Istio CRs: 12
  - DestinationRule xcp-multicluster/global-gateway-echo-echo-tr-svc-cluster-local
  - ServiceEntry xcp-multicluster/global-gateway-echo-echo-tr-svc-cluster-local
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
- XCP workspace(s): none resolved — 03-xcp-config falls back to cluster-wide listing
- XCP config CRs: 9
  - Workspace istio-system/transit-ws
  - Workspace xcp-system/transit-ws
  - GatewayGroup xcp-system/transit-backend-gg
  - GatewayGroup xcp-system/transit-mid-gg
  - Gateway xcp-system/transit-backend-gw
  - Gateway xcp-system/transit-mid-gw
  - GlobalSetting istio-system/global
  - GlobalSetting xcp-system/global
  - EdgeXcp istio-system/edgexcp

## Revisions

No `istio.io/rev` signal from data-plane or gateway pods — no filter applied. Every istiod replica found by `app=istiod` was captured.

## Tier-1 cross-cluster note

NOTE: tier-1 rewrites the HTTP authority — confirm the real tier-2 hostname in `01-envoy/<tier1-pod>/config_dump.json` route config (look at `route.rewrite.authority` and the `outbound|...` cluster name) before re-running the dump there. TCP/TLS-passthrough is SNI-routed and not rewritten.

## Collectors

- `envoy` (01-envoy): empty, 0 file(s)
- `istio-config` (02-istio-config): ok, 12 file(s)
- `istiod` (05-istiod): ok, 7 file(s)
- `k8s` (04-k8s): ok, 1 file(s)
- `xcp-config` (03-xcp-config): ok, 9 file(s)
- `xcp-edge` (06-xcp-edge): ok, 9 file(s)

## Gold-thread hints

1. `01-envoy/<pod>/stats.txt` — non-zero `no_cluster_found` points at a missing route/cluster (the classic 503); `upstream_cx_connect_fail`, `upstream_cx_none_healthy` point at L4/TCP upstream failure.
2. `01-envoy/<pod>/clusters.json` — search for the destination cluster; absent cluster or endpoints with `health_flags` set explains UH/UF.
3. `01-envoy/<pod>/config_dump.json` — route config and listener filter-chains; for tier-1 this is the authoritative cross-cluster artifact.
4. `02-istio-config/` — the raw Istio CRs that shape routing for this host.
5. `03-xcp-config/` — the XCP edge config CRs (`*.xcp.tetrate.io`): the Workspace and *Group containers plus the XCP gateway/routing intent. Empty if XCP has not materialised config on this cluster — cross-check the edge's live applied config in `06-xcp-edge/debug-appliedconfigz.json`.
6. `06-xcp-edge/debug-gateways.json` — whether the XCP edge knows the host.
