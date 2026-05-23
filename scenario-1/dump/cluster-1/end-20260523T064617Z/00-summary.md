# Minimal diagnostic dump

- Hostname: `echo.tetrate.io`
- Namespace: `echo`
- Service name: `echo`
- Cluster role: **workload** (local Service echo/echo with 1 backing pod(s))
- Collected: 2026-05-23T06:46:17Z

## What was found

- Local Service: yes
- Backing pods: 1 [echo/echo-6d455455d9-mjsm9]
- Gateway pods: 1 [echo/echo-gateway-5f4f69f4cf-55m67]
- Istiod pods: 1 [istio-system/istiod-stable-b4c7c467b-2sqsx]
- XCP edge pods: 1 [istio-system/edge-bdb796b6f-xxdp2]
- Matched Istio CRs: 15
  - VirtualService echo/vs-echo-tetrate-io
  - Gateway echo/echo-t2-gw
  - ServiceEntry xcp-multicluster/global-gateway-echo-tetrate-io
  - EnvoyFilter echo/default-allow-echo-t2-gw-external-batch0
  - EnvoyFilter echo/default-upstream-traffic-setting
  - EnvoyFilter echo/tier2-gg-default-upstream-traffic-setting
  - EnvoyFilter istio-system/global-forward-xfcc-15443
  - EnvoyFilter istio-system/global-gw-best-practice
  - EnvoyFilter istio-system/global-mx-downstream-egress-gateway-filter
  - EnvoyFilter istio-system/global-mx-downstream-gateway-filter
  - EnvoyFilter istio-system/global-periodical-access-log
  - EnvoyFilter istio-system/global-request-id-propagation
  - EnvoyFilter istio-system/global-tcp-keepalive-envoy-filter
  - EnvoyFilter istio-system/global-xfcc-extractor-sidecar
  - EnvoyFilter istio-system/global-xfcc-guard-15443
- XCP config CRs: 4
  - Workspace istio-system/passthrough-ws
  - GlobalSetting istio-system/global
  - GatewayGroup istio-system/tier2-gg
  - IngressGateway istio-system/echo-t2-gw

## Collectors

- `envoy` (01-envoy): ok, 10 file(s)
- `istio-config` (02-istio-config): ok, 15 file(s)
- `istiod` (05-istiod): ok, 7 file(s)
- `k8s` (04-k8s): ok, 4 file(s)
- `xcp-config` (03-xcp-config): ok, 4 file(s)
- `xcp-edge` (06-xcp-edge): ok, 9 file(s)

## Gold-thread hints

1. `01-envoy/<pod>/stats.txt` — non-zero `no_cluster_found` points at a missing route/cluster (the classic 503); `upstream_cx_connect_fail`, `upstream_cx_none_healthy` point at L4/TCP upstream failure.
2. `01-envoy/<pod>/clusters.json` — search for the destination cluster; absent cluster or endpoints with `health_flags` set explains UH/UF.
3. `01-envoy/<pod>/config_dump.json` — route config and listener filter-chains; for tier-1 this is the authoritative cross-cluster artifact.
4. `02-istio-config/` — the raw Istio CRs that shape routing for this host.
5. `03-xcp-config/` — the XCP edge config CRs (`*.xcp.tetrate.io`): the Workspace and *Group containers plus the XCP gateway/routing intent. Empty if XCP has not materialised config on this cluster — cross-check the edge's live applied config in `06-xcp-edge/debug-appliedconfigz.json`.
6. `06-xcp-edge/debug-gateways.json` — whether the XCP edge knows the host.
