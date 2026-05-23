# Minimal diagnostic dump

- Hostname: `backend.backend-c1.svc.cluster.local`
- Namespace: `backend-c1`
- Service name: `backend`
- Cluster role: **workload** (local Service backend-c1/backend with 1 backing pod(s))
- Collected: 2026-05-23T09:18:19Z

## What was found

- Local Service: yes
- Backing pods: 1 [backend-c1/backend-86f5b4bc7b-6btcm]
- Gateway pods: 2 [backend-c1/ew-gateway-698bcf4fb5-zmnpx echo-term/echo-term-gateway-6bc8b5ffdd-lq86b]
- Istiod pods: 1 [istio-system/istiod-stable-b4c7c467b-2sqsx]
- XCP edge pods: 1 [istio-system/edge-bdb796b6f-xxdp2]
- Matched Istio CRs: 13
  - VirtualService backend-c1/vs-xc-ws-0
  - Gateway backend-c1/ewgen-xc-ws-0
  - DestinationRule backend-c1/dr-ew-backend-backend-c1-svc-cluster-local
  - EnvoyFilter backend-c1/default-upstream-traffic-setting
  - EnvoyFilter istio-system/global-forward-xfcc-15443
  - EnvoyFilter istio-system/global-gw-best-practice
  - EnvoyFilter istio-system/global-mx-downstream-egress-gateway-filter
  - EnvoyFilter istio-system/global-mx-downstream-gateway-filter
  - EnvoyFilter istio-system/global-periodical-access-log
  - EnvoyFilter istio-system/global-request-id-propagation
  - EnvoyFilter istio-system/global-tcp-keepalive-envoy-filter
  - EnvoyFilter istio-system/global-xfcc-extractor-sidecar
  - EnvoyFilter istio-system/global-xfcc-guard-15443
- XCP config CRs: 3
  - Workspace istio-system/xc-ws
  - WorkspaceSetting istio-system/xc-ws-settings
  - GlobalSetting istio-system/global

## Collectors

- `envoy` (01-envoy): ok, 15 file(s)
- `istio-config` (02-istio-config): ok, 13 file(s)
- `istiod` (05-istiod): ok, 8 file(s)
- `k8s` (04-k8s): ok, 4 file(s)
- `xcp-config` (03-xcp-config): ok, 3 file(s)
- `xcp-edge` (06-xcp-edge): ok, 9 file(s)

## Gold-thread hints

1. `01-envoy/<pod>/stats.txt` — non-zero `no_cluster_found` points at a missing route/cluster (the classic 503); `upstream_cx_connect_fail`, `upstream_cx_none_healthy` point at L4/TCP upstream failure.
2. `01-envoy/<pod>/clusters.json` — search for the destination cluster; absent cluster or endpoints with `health_flags` set explains UH/UF.
3. `01-envoy/<pod>/config_dump.json` — route config and listener filter-chains; for tier-1 this is the authoritative cross-cluster artifact.
4. `02-istio-config/` — the raw Istio CRs that shape routing for this host.
5. `03-xcp-config/` — the XCP edge config CRs (`*.xcp.tetrate.io`): the Workspace and *Group containers plus the XCP gateway/routing intent. Empty if XCP has not materialised config on this cluster — cross-check the edge's live applied config in `06-xcp-edge/debug-appliedconfigz.json`.
6. `06-xcp-edge/debug-gateways.json` — whether the XCP edge knows the host.
