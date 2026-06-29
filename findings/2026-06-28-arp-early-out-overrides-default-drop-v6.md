---
id: finding/2026-06-28-arp-early-out-overrides-default-drop-v6
type: finding
protocol: [ipv6]
builtins: []
severity: low
layer: emitter
pattern_tags: [cross-family-default-action, arp-early-out]
status: open
source_file: corpus/from_hunt/2026-05-01_v6_internal_hunt/v01_shaped_vs_v6_packet.pkt
created: 2026-06-28
pkt_path: corpus/from_hunt/2026-05-01_v6_internal_hunt/v01_shaped_vs_v6_packet.pkt
---

# 2026-06-28-arp-early-out-overrides-default-drop-v6

## Summary

The corpus case `v01_shaped_vs_v6_packet.pkt` expects DROP but the BPF
oracle returns PASS. A v0.1-shaped program (no IPv6 fields referenced)
evaluated against an IPv6 frame: the v0.2 ARP-protection early-out
(`if (!v4_ok && !v6_ok) return XDP_PASS;`) fires because the program
never activates the v6 parse path, so the frame is treated as non-IP
and passed — overriding the program's `default drop`.

## Discovery

Surfaced during f-04b (VLAN) verification. Proven pre-existing: the
f-04b emitter produces byte-identical C for this VLAN-free program
compared to master, so it is not a VLAN regression. It is a v0.2
tension between the ARP early-out and default-drop semantics.

## Root Cause

The ARP early-out (added to fix soak Incident #3, see
[[2026-05-02-soak-emitter-non-ip-arp-drop]]) returns XDP_PASS for any
frame that activates neither the v4 nor v6 parse path. A v0.1-shaped
program with `default drop` does not reference v6 fields, so a v6 frame
does not set v6_ok, the early-out fires, and the default drop is
bypassed.

## Why It's Deferred

Fixing it touches the ARP early-out, which is load-bearing for the
soak Incident #3 fix. The correct fix needs care: distinguish "non-IP
frame" (ARP, LLDP — should pass) from "IP frame the program chose not
to parse" (should hit default). This belongs in a dedicated change,
not bolted onto an unrelated feature workspace.

## Acceptance for the Fix

- A v0.1-shaped program with `default drop` drops a v6 frame
- ARP/LLDP frames still pass (Incident #3 regression test stays green)
- The corpus case flips to passing
