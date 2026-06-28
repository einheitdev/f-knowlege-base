---
id: finding/2026-05-01-v6-internal-fw-ndp-dad-and-rs-dropped-against-stated-intent
type: finding
protocol: [icmp6]
builtins: []
severity: medium
layer: user-rule
pattern_tags: [ipv6, ndp, link-local, user-intent-mismatch, control-plane, rfc-4861, rfc-4862]
status: fixed
source_file: fwl/examples/v6_internal.fw
created: 2026-05-01
pkt_path: corpus/from_hunt/2026-05-01_v6_internal_hunt/v6_internal_dad_drops.pkt
related: []
---

# 2026-05-01-v6-internal-fw-ndp-dad-and-rs-dropped-against-stated-intent

## Summary

`fwl/examples/v6_internal.fw` rule 4
(`allow if pkt.proto == icmp6 and pkt.src_ip6 in fe80::/10`) gates
the IPv6 control-plane allow on a link-local (`fe80::/10`) source
address. The header comment states the intent of this rule is:

> # IPv6 neighbour discovery / link-local control plane stays open

But two essential NDP message types **must** use the unspecified
address (`::`) as their IPv6 source per the RFCs, and `::` is not
in `fe80::/10`:

1. **Duplicate Address Detection (DAD).** RFC 4862 §5.4.2 mandates
   that the Neighbor Solicitation a host sends while tentatively
   claiming an address use `::` as the source. Without DAD, two
   hosts on the same link can independently configure the same
   address.
2. **Router Solicitation at startup.** RFC 4861 §6.3.7 allows (and
   in practice almost all stacks use) `::` as the source for an
   initial Router Solicitation a host sends before it has a
   link-local address yet (interface coming up, before DAD has
   completed for the link-local).

Both packet types reach `v6_internal.fw` matching `pkt.proto ==
icmp6` but fail `pkt.src_ip6 in fe80::/10` because `::` has top-7
bits = 0, which is not in `fe80::/10` (top-10 bits =
`1111111010`). The packet falls through every rule and hits
`default drop`.

## Hypothesis

The rule's natural-language description (the inline comment) reads
as "allow IPv6 NDP / link-local control-plane traffic." The
implementation reads as "allow IPv6 ICMPv6 packets sourced from
link-local." These are not equivalent — RFC-required NDP packets
sometimes use `::` instead of a link-local source, and those are
exactly the messages a fresh-bringing-up node sends to discover its
neighbours and routers.

The result is a working firewall with one operational hazard: any
host coming up on the link gated by this XDP firewall has its DAD
broadcast-receipt path silently broken. If two hosts behind the
firewall claim the same address, the firewall hides that conflict
from each of them.

## Evidence

`corpus/from_hunt/2026-05-01_v6_internal_hunt/v6_internal_dad_drops.pkt`:

```yaml
source_fw: |
  @xdp(eth0)
  allow if pkt.src_ip in [10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16]
  allow if pkt.src_ip6 in fc00::/7
  allow if pkt.proto == tcp and pkt.dst_port in [80, 443, 22]
  allow if pkt.proto == udp and pkt.dst_port == 53
  allow if pkt.proto == icmp6 and pkt.src_ip6 in fe80::/10
  log if pkt.proto == tcp and pkt.tcp.syn and not pkt.tcp.ack
  default drop

test_packet:
  builder: icmp6(src_ip="::", dst_ip="ff02::1:ff00:1")

expected:
  bpf_action: drop
```

Run output:

```
PASS  v6_internal_dad_drops.pkt  (...)
      spec         pass
      interpreter  pass
      bpf          skip  [...clang compile passed]
```

Both oracles agree the packet drops — the firewall is internally
consistent. The bug is a Tier C user-rule bug: the
natural-language *intent* of the rule (NDP control plane stays
open) does not match what the rule actually does (only sources in
`fe80::/10`).

## Surface area

| NDP message | Source per RFC | Currently allowed? |
|---|---|---|
| Neighbor Solicitation (normal) | link-local of sender | ✓ allowed |
| Neighbor Solicitation (DAD) | `::` (RFC 4862 §5.4.2) | ✗ dropped |
| Router Solicitation (post-LL) | link-local of sender | ✓ allowed |
| Router Solicitation (pre-LL) | `::` (RFC 4861 §6.3.7) | ✗ dropped |
| Neighbor Advertisement | link-local | ✓ allowed |
| Router Advertisement | link-local of router | ✓ allowed |
| MLDv2 (Multicast Listener) | link-local | ✓ allowed |

So two of the seven NDP message types — both used during interface
bring-up — are silently dropped.

## Recommended fix

The rule needs to also allow `::` as a source. Two equally clean
options:

```python
# A) explicit-list form
allow if pkt.proto == icmp6 and pkt.src_ip6 in [fe80::/10, ::/128]

# B) "any link-local control plane" form, accepting ULA as well
allow if pkt.proto == icmp6
```

(Option B drops the source filter entirely. ICMPv6 control traffic
is generally safe to allow at this layer because it has its own
Hop-Limit-255 receive check at the kernel, but option A is the
minimal change matching the comment's intent.)

The fix is in `fwl/examples/v6_internal.fw`. The corpus case
`v6_internal_dad_drops.pkt` will flip to `bpf_action: allow` once
the source filter is widened.

## Why this slipped through

The corpus under `f/fwl/tests/corpus/08_ipv6_fields/` exercises
generic IPv6 CIDR/literal matching, not the specific RFC-mandated
NDP message corner cases. Phase 2 verification asks "does the
program do what it says" — the comment says "control plane" but
the implementation accepts only one kind of control-plane source.
A targeted DAD/RS test would have caught it in dogfooding, but the
existing tests don't probe NDP semantics.
