# Minimal diagnostic dump — observation window

- Hostname: `echo-term.tetrate.io`
- Namespace: `tier1-term`
- Requested window: `1m0s`
- Window: 2026-05-28T14:45:25Z → 2026-05-28T14:46:25Z

## Snapshots

- START: `start-20260528T144525Z/` — taken 2026-05-28T14:45:25Z
- END:   `end-20260528T144625Z/` — taken 2026-05-28T14:46:25Z

Each subdirectory is a complete minimal dump, identical in layout to a plain `collect-minimal` run (`01-envoy` … `06-xcp-edge`, `00-summary.md`, `manifest.json`).

## How to read the window

1. **Logs** — open `start-20260528T144525Z/01-envoy/<pod>/proxy.log` and note its **last timestamp**. Then open `end-20260528T144625Z/01-envoy/<pod>/proxy.log` and read from that timestamp onward: those lines are what the proxy did during the window.
2. **Envoy stats** — `end-20260528T144625Z/01-envoy/<pod>/stats.txt` minus `start-20260528T144525Z/01-envoy/<pod>/stats.txt` is the per-counter delta for the window (Envoy counters are monotonic).
3. **Config / CRs** — diff the `02-istio-config/` and `03-xcp-config/` layers between the two snapshots to see routing config that changed mid-window.

## Proxy continuity

OK — the same Envoy pods served the whole window. Start→end stats deltas are meaningful.
