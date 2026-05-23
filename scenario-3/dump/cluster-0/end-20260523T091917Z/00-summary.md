# Minimal diagnostic dump

- Hostname: `backend.backend-c1.svc.cluster.local`
- Namespace: `client-c0`
- Service name: `backend`
- Cluster role: **standalone** (no local Service for hostname and no Tier1Gateway CR; may be a tier-1 cluster in classic mode — confirm via gateway config_dump)
- Collected: 2026-05-23T09:19:17Z

## What was found

- Local Service: no
- Backing pods: 0 []
- Gateway pods: 0 []
- Istiod pods: 1 [istio-system/istiod-stable-65bd95ccc-pqdvh]
- XCP edge pods: 1 [istio-system/edge-668cdd6dc9-r9jpw]
- Matched Istio CRs: 12
  - DestinationRule xcp-multicluster/global-drinternal-backend-backend-c1-svc-cluster-local
  - ServiceEntry xcp-multicluster/global-gwinternal-backend-backend-c1-svc-cluster-local
  - EnvoyFilter client-c0/default-upstream-traffic-setting
  - EnvoyFilter istio-system/global-forward-xfcc-15443
  - EnvoyFilter istio-system/global-gw-best-practice
  - EnvoyFilter istio-system/global-mx-downstream-egress-gateway-filter
  - EnvoyFilter istio-system/global-mx-downstream-gateway-filter
  - EnvoyFilter istio-system/global-periodical-access-log
  - EnvoyFilter istio-system/global-request-id-propagation
  - EnvoyFilter istio-system/global-tcp-keepalive-envoy-filter
  - EnvoyFilter istio-system/global-xfcc-extractor-sidecar
  - EnvoyFilter istio-system/global-xfcc-guard-15443
- XCP config CRs: 6
  - Workspace istio-system/xc-ws
  - Workspace xcp-system/xc-ws
  - WorkspaceSetting istio-system/xc-ws-settings
  - WorkspaceSetting xcp-system/xc-ws-settings
  - GlobalSetting istio-system/global
  - GlobalSetting xcp-system/global

## Collectors

- `envoy` (01-envoy): empty, 0 file(s)
- `istio-config` (02-istio-config): ok, 12 file(s)
- `istiod` (05-istiod): ok, 5 file(s)
- `k8s` (04-k8s): ok, 1 file(s)
- `xcp-config` (03-xcp-config): ok, 6 file(s)
- `xcp-edge` (06-xcp-edge): ok, 9 file(s)

## Gold-thread hints

1. `01-envoy/<pod>/stats.txt` — non-zero `no_cluster_found` points at a missing route/cluster (the classic 503); `upstream_cx_connect_fail`, `upstream_cx_none_healthy` point at L4/TCP upstream failure.
2. `01-envoy/<pod>/clusters.json` — search for the destination cluster; absent cluster or endpoints with `health_flags` set explains UH/UF.
3. `01-envoy/<pod>/config_dump.json` — route config and listener filter-chains; for tier-1 this is the authoritative cross-cluster artifact.
4. `02-istio-config/` — the raw Istio CRs that shape routing for this host.
5. `03-xcp-config/` — the XCP edge config CRs (`*.xcp.tetrate.io`): the Workspace and *Group containers plus the XCP gateway/routing intent. Empty if XCP has not materialised config on this cluster — cross-check the edge's live applied config in `06-xcp-edge/debug-appliedconfigz.json`.
6. `06-xcp-edge/debug-gateways.json` — whether the XCP edge knows the host.
