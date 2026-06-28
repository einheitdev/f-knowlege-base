---
id: finding/2026-05-02-soak-emitter-non-ip-arp-drop
type: finding
protocol: [arp]
builtins: []
severity: high
layer: emitter
pattern_tags: [cross-family-default-action, soak-incident]
status: fixed
source_file: fwl/examples/dogfood_v02.fw
created: 2026-05-02
pkt_path: ''
---

# 2026-05-02-soak-emitter-non-ip-arp-drop

## Summary

Non-IP frames (ARP, LLDP, etc.) fall through the v0.2 emitter's
parse prelude with `v4_ok = v6_ok = 0`. Every rule guard
short-circuits false, and control reaches the explicit `default
drop` at the function tail. On a live interface this drops ARP,
breaking L2 resolution and killing SSH access within minutes of
the ARP cache expiring.

## Discovery

Soak Incident #3 on deb-02 (2026-05-02T10:55 UTC). SSH returned
"No route to host" after fd.service attached dogfood_v02.fw.
Recovery required out-of-band `virsh shutdown`.

## Root Cause

The emitter's `_emit_parse_prelude` generated no early-out for
frames that are neither IPv4 (EtherType 0x0800) nor IPv6
(EtherType 0x86DD). The `default drop` at function tail then
applied unconditionally.

## Fix

`f/fwl/fwl/emitter.py` — append `if (!v4_ok && !v6_ok) return
XDP_PASS;` to the parse prelude when the program references any
IP-aware field. Pinned by 4 unit tests in
`tests/unit/test_emitter.py::TestNonIpEarlyOut`.

## Why Prior Gates Missed It

`BPF_PROG_TEST_RUN` evaluates one packet at a time; the corpus
contains TCP/UDP/ICMP/ICMPv6 cases but zero ARP frames. The soak
is the first gate that exercises the program against the full
traffic mix a live NIC receives.
