---
id: finding/2026-05-02-soak-bpf-loader-program-name-mismatch
type: finding
protocol: []
builtins: []
severity: high
layer: daemon
pattern_tags: [v01-v02-transition, hard-coded-name, soak-incident]
status: fixed
source_file: f/src/bpf_loader.cc
created: 2026-05-02
pkt_path: ''
---

# 2026-05-02-soak-bpf-loader-program-name-mismatch

## Summary

The v0.1 daemon hard-codes `bpf_object__find_program_by_name(obj,
"fw_prog")` but the v0.2 FWL emitter outputs `fwl_prog`. Loading
a v0.2 bundle caused `fw_prog not found in BPF object`, triggering
11 systemd restart cycles before the soak agent intervened.

## Discovery

Soak Incident #1 on deb-02 (2026-05-02T10:34 UTC).

## Fix

`f/src/bpf_loader.cc` — try `fwl_prog` first, fall back to
`fw_prog` for v0.1 bundle compatibility.

## Related

Soak Incident #2 (pin map name mismatch) is the same pattern:
v0.1 PinMaps loop references maps that v0.2 bundles don't carry.
Fixed by skipping maps with `fd < 0`.
