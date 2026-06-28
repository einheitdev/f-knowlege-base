---
id: finding/2026-05-02-soak-hone-hunt-permission-denied
type: finding
protocol: []
builtins: []
severity: medium
layer: harness
pattern_tags: [soak-infrastructure, deployment-config]
status: fixed
source_file: f-hone/deploy/systemd/hone-soak.service
created: 2026-05-02
pkt_path: ''
---

# 2026-05-02-soak-hone-hunt-permission-denied

## Summary

`hone-soak.service` ran as user `fd` (uid 999, `/sbin/nologin`).
The `claude` CLI lives in `worker`'s home directory. Every `hone
hunt` invocation across 96 soak ticks failed with
`PermissionError: [Errno 13] Permission denied:
'/home/fd/.npm-global/bin/claude'`. The soak never spawned an
LLM-driven adversarial round — 96 hunt ticks of capacity lost.

## Discovery

Soak Finding #2. Observed in soak logs across the full 53h window.

## Fix

Changed `User=fd` / `Group=fd` to `User=worker` / `Group=worker`
in both `deploy/systemd/hone-soak.service` and
`deploy/staging/hone-soak.service`. Also set `ProtectHome=false`
so the worker user can reach `claude` in its home directory.
