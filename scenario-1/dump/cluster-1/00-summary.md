# Minimal diagnostic dump — observation window

- Hostname: `echo.tetrate.io`
- Namespace: `echo`
- Requested window: `1m0s`
- Window: 2026-05-28T14:38:18Z → 2026-05-28T14:39:19Z

## Snapshots

- START: `start-20260528T143818Z/` — taken 2026-05-28T14:38:18Z
- END:   `end-20260528T143919Z/` — taken 2026-05-28T14:39:19Z

Each subdirectory is a complete minimal dump, identical in layout to a plain `collect-minimal` run (`01-envoy` … `06-xcp-edge`, `00-summary.md`, `manifest.json`).

## How to read the window

1. **Logs** — open `start-20260528T143818Z/01-envoy/<pod>/proxy.log` and note its **last timestamp**. Then open `end-20260528T143919Z/01-envoy/<pod>/proxy.log` and read from that timestamp onward: those lines are what the proxy did during the window.
2. **Envoy stats** — `end-20260528T143919Z/01-envoy/<pod>/stats.txt` minus `start-20260528T143818Z/01-envoy/<pod>/stats.txt` is the per-counter delta for the window (Envoy counters are monotonic).
3. **Config / CRs** — diff the `02-istio-config/` and `03-xcp-config/` layers between the two snapshots to see routing config that changed mid-window.

## Proxy continuity

OK — the same Envoy pods served the whole window. Start→end stats deltas are meaningful.
