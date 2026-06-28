---
id: finding/2026-05-01-v02-emitter-no-v4-parse-ok-gate-cross-family-disagreement
type: finding
protocol: [tcp, udp, icmp6]
builtins: []
severity: high
layer: emitter
pattern_tags: [cross-family, oracle-disagreement, ipv6, v4-parse-gate, tier2, dogfood, default-zero, asymmetry]
status: fixed
source_file: fwl/fwl/emitter.py
created: 2026-05-01
fixed: 2026-05-01
pkt_path: corpus/from_hunt/2026-05-01_dogfood_round2_emitter_v4_gate/
related: [finding/2026-05-01-v02-interpreter-missing-v6-surface-activation-gate]
---

## Fix applied

Added two new BPF prelude flags to `fwl/fwl/emitter.py`:

- `__u8 v4_ok = 0;` — set to 1 inside the IPv4 EtherType branch
  after the fixed-header bounds check. Gates every v4-field
  comparison (`_emit_ip_compare`, `_emit_proto_compare`).
- `__u8 l4_ok = 0;` — set to 1 inside each TCP/UDP guard block
  after the L4 bounds check (both v4 and v6 branches). Gates every
  port comparison and `pkt.tcp.syn`/`pkt.tcp.ack` bool read.

`_emit_proto_compare` now also gates on `(v4_ok || v6_ok)` so a
non-IP frame can't match `pkt.proto != tcp`. The `v6_ok` flag is
now always declared (was previously gated on v6_active) so the
OR expression is well-formed in v0.1-shaped programs (where v6_ok
stays 0 and the OR degrades to v4_ok).

Pinned regressions:
- `fwl/tests/corpus/10_tier2_functions/edge_v4_zero_cidr_v6_packet.pkt`
- `fwl/tests/corpus/10_tier2_functions/edge_src_ip_eq_zero_v6_packet.pkt`

Verified: 457/457 pytest, 133/133 corpus regress under sudo
with BPF live, no regressions.


# 2026-05-01-v02-emitter-no-v4-parse-ok-gate-cross-family-disagreement

## Summary

The BPF emitter (`fwl/fwl/emitter.py`) gates IPv6 field comparisons
on a `v6_ok` flag that is set to 1 only after the IPv6 fixed-header
bounds check succeeds (see `_emit_v6_branch:502`,
`_emit_ip6_compare:654`). The emitter has **no equivalent
`v4_ok` flag for IPv4 fields**: `src_ip`, `dst_ip`, `src_port`,
`dst_port`, `tcp_syn`, `tcp_ack`, and `proto` are declared as
zero-initialized C variables (`_emit_parse_prelude:315-337`) and
overwritten only when the matching parse step succeeds. Comparisons
emitted by `_emit_ip_compare`, `_emit_port_compare`, and
`_emit_proto_compare` then test those C variables raw — without
checking whether the parse step that should have populated them
actually ran.

For an IPv6 packet processed by a **v6-active program** (the case
that matters: `pkt.src_ip6 in fc00::/7` activates the v6 parse path,
so the program reaches the v4-field rule below), the v4 parse branch
is skipped entirely (different EtherType). `src_ip` stays at 0.

Now any v4-field comparison whose *true value when src_ip is 0* is
**true** spuriously fires on the v6 packet. Symmetric issue for
`dst_port` etc. on packets whose protocol the BPF parser doesn't
read L4 from (ICMPv6, extension headers).

The AST interpreter does the right thing: a v6 builder packet has no
`src_ip`/`dst_ip` keys in its decoded dict (per `PKT_V02_SPEC.md
:120`), so the comparison reads `None` and falls through. The two
oracles disagree on these v4-field comparisons applied to v6 packets.

## Hypothesis

A v6-active Tier 2 program reaches a v4-field comparison on a v6
packet (because the v6 packet doesn't match the v6-only earlier
guards). The interpreter falls through; the BPF emits a comparison
against the zero-initialized `src_ip`. Whenever the comparison's
arithmetic equates "0" with "match", BPF disagrees with interpreter.

The four canonical patterns:

1. `pkt.src_ip == 0.0.0.0` — BPF: `(src_ip == 0)` = true; interpreter: false.
2. `pkt.src_ip != X.X.X.X` (X non-zero) — BPF: `(src_ip != X)` = true; interpreter: false.
3. `pkt.src_ip in [0.0.0.0/0]` — emitter shortcircuits `_emit_cidr_match` to `1` (always true) for `/0`; interpreter: false.
4. `pkt.dst_port == 0` (no proto guard, ICMPv6 packet) — BPF: dst_port stays 0 (TCP/UDP block skipped on next_header=58), `(dst_port == 0)` = true; interpreter: dst_port absent → false.

(Variant 3 is the most concerning: the spec's *recommended* v4-
establishing guard inside Tier 2 is `if pkt.src_ip in [0.0.0.0/0]:`
— see `FWL_V02_SPEC.md:1007` and example 5 in the Tier 2 section.
Programs following the spec's own pattern are bitten on every v6
packet that reaches the guard.)

## Evidence

`corpus/from_hunt/2026-05-01_dogfood_round2_emitter_v4_gate/` —
four `.pkt` cases all using a v6-active two-rule program (the v6
guard lets v6 packets fall through to the v4-field rule):

```yaml
source_fw: |
  @xdp(eth0)
  def firewall(pkt):
    if pkt.src_ip6 in fc00::/7:
      drop
    if pkt.src_ip in [0.0.0.0/0]:   # variant 3
      drop
    allow

test_packet:
  builder: tcp6(src_ip="2001:db8::dead", dst_ip="2001:db8::cafe", dst_port=443)

expected:
  bpf_action: allow
```

Run output (sudo, all four):

```
FAIL  probe_dst_port_v6_no_proto_guard.pkt
      spec         pass
      interpreter  pass
      bpf          fail  -- expected XDP_PASS, got XDP_DROP
FAIL  probe_src_ip_eq_zero_v6_packet.pkt
      spec         pass
      interpreter  pass
      bpf          fail  -- expected XDP_PASS, got XDP_DROP
FAIL  probe_src_ip_neq_eight_v6_packet.pkt
      spec         pass
      interpreter  pass
      bpf          fail  -- expected XDP_PASS, got XDP_DROP
FAIL  probe_v4_zero_cidr_v6_packet.pkt
      spec         pass
      interpreter  pass
      bpf          fail  -- expected XDP_PASS, got XDP_DROP

0/4 cases passed
```

Inspecting the emitted C for variant 3 (`pkt.src_ip in [0.0.0.0/0]`):

```c
__u32 src_ip = 0;
__u8 v6_ok = 0;
...
if (eth->h_proto == bpf_htons(ETH_P_IP)) {
  // src_ip = bpf_ntohl(ip->saddr);   <-- not reached on v6
} else if (eth->h_proto == bpf_htons(ETH_P_IPV6)) {
  v6_ok = 1;
  // src_ip6_hi/lo populated; src_ip stays 0
}
...
if ((v6_ok && ((src_ip6_hi & 0xFE00...) == 0xFC00...))) {
  return XDP_DROP;
}
if ((1)) {                             // <-- /0 emitted as constant 1
  return XDP_DROP;
}
return XDP_PASS;
```

The `(1)` is the smoking gun: `_emit_cidr_match:611-612` returns the
literal C `"1"` for `cidr.bits == 0`. There is no `v4_ok && (1)` gate
analogous to the v6 path. On any v6 packet that reaches the rule,
the emitter unconditionally drops.

## Root cause

`_emit_parse_prelude` in `fwl/fwl/emitter.py:307-422` declares C
variables `__u32 src_ip = 0` etc. and populates them only inside
`if (eth->h_proto == bpf_htons(ETH_P_IP)) { ... }`. There is no flag
recording whether that branch ran.

`_emit_ip_compare` (line 575-585) and `_emit_cidr_match` (line
609-616) emit raw comparisons against the C variable. They have no
gate on "did the v4 parse succeed?".

Compare to `_emit_ip6_compare` at line 630-659, which prepends
`(v6_ok && ...)` to every v6 field comparison precisely to handle
the symmetric case (a v4 packet on a program reading v6 fields).
The asymmetry is documented in the comment at lines 320-323:

```python
# v6_ok gates every IPv6 comparison so that v4-packet zeros don't
# spuriously match `pkt.src_ip6 == ::` or `pkt.src_ip6 != <non-zero>`.
# Set to 1 only inside the v6 branch after the 40-byte fixed-header
# bounds check succeeds.
```

The same hazard exists symmetrically for v4 fields on v6 packets,
but the emitter does not address it.

A separate L4-parse-OK problem applies to `dst_port`/`src_port`/
`tcp_syn`/`tcp_ack`: these are populated only inside `if (proto ==
IPPROTO_TCP) { ... }` / `if (proto == IPPROTO_UDP) { ... }` blocks.
Programs that read these fields without a proto guard (the spec's
Tier 2 dominator rule normally requires the guard, but Tier 1 has
no analogous check, and the analyzer's dominator rule does not
reach short-circuit-protected reads per `FWL_V02_SPEC.md:612`) hit
the same "value 0 looks like a match" hazard.

## Why this matters for dogfood_v02.fw

The actual `f/fwl/examples/dogfood_v02.fw` doesn't textually use
`/0`, `0.0.0.0`, or `dst_port == 0` patterns. Its v4 prefixes
(`10.0.0.0/8`, `192.168.0.0/16`, `172.16.0.0/12`) all have non-zero
`prefix & mask`, so on a v6 packet `(0 & mask) == prefix` evaluates
false in BPF — same as interpreter. **The dogfood example happens to
dodge the bug**, but only because no operator-supplied prefix is
0-prefixed.

The spec's *Tier 2 dogfood example* (`FWL_V02_SPEC.md:986-1030`) is
**not** safe — line 1007 has `if pkt.src_ip in [0.0.0.0/0]:` as the
v4-establishing guard. Every v6 TCP/22 SYN reaching that guard
takes the BPF then-branch (executing the v4-only rate-limit and
counter logic) but the interpreter then-branch is not taken. The
two oracles silently disagree on every v6 SSH packet under the
spec's own dogfood — and the soak runs the spec's dogfood, not the
truncated `dogfood_v02.fw` in the examples directory.

A user who copy-pastes the spec's example, follows the spec's
v4-establishing-guard recommendation, and runs `.pkt` tests in CI
without root sees `bpf=skip` and a green corpus. The bug surfaces
only when BPF runs (production daemon, root `fwl test`, or the
operator's deploy soak).

## Surface area

Every program that combines:
- v6 parse activation (any v6 surface mentioned, anywhere), AND
- a comparison against a v4 field (`pkt.src_ip`, `pkt.dst_ip`)
  whose right-hand side is `0.0.0.0`, an IPv4 CIDR with prefix
  starting at 0 (e.g., `0.0.0.0/0`, `0.0.0.0/8`), or `!= <non-zero>`.

OR

- a comparison against an L4 field (`pkt.dst_port`, `pkt.src_port`,
  `pkt.tcp.syn`, `pkt.tcp.ack`) that is **not** dominated by a
  proto guard, and whose right-hand side is the field's
  zero-initialized value (`== 0`, `!= <non-zero>`, `in [0]`).

Tier 1 doesn't have a dominator rule, so v0.1-shaped programs that
write `if pkt.dst_port == 0:` in a rule already trip variant 4 on
ICMPv6/extension-header packets — but only in v6-active programs
(v0.1-shaped programs already fall through every rule on v6 frames
per `_gate_v6_packet_for_v01_program`).

## Recommended fix

Mirror the v6 pattern. In `_emit_parse_prelude`:

1. Declare `__u8 v4_ok = 0;` whenever the program touches v4 fields.
2. Set `v4_ok = 1;` inside the IPv4 branch after `(void *)(ip + 1)
   <= data_end` succeeds (alongside `proto = ip->protocol;`).
3. Optionally declare `__u8 l4_tcp_ok = 0;` / `__u8 l4_udp_ok = 0;`
   and set them inside the TCP/UDP guard blocks for `dst_port`/etc.

In `_emit_ip_compare` / `_emit_port_compare` / `_emit_proto_compare`:
prepend `(v4_ok && ...)` (and the L4 gate where relevant) to every
emitted comparison, matching the v6 path.

`_emit_cidr_match` for `bits == 0` should not return the literal
`"1"` — it should return `"v4_ok"` (or be wrapped by the caller
adding the gate). Either fix removes the spurious match on v6
packets.

The interpreter is already correct; no interpreter change is needed.

## Class

Class A — interpreter and BPF runtime disagree on the same packet.
Found while hunting on `dogfood_v02.fw`; the actual file dodges the
bug but the spec's Tier 2 dogfood example (which the Phase 2 ≥48h
soak runs) is bitten on every v6 SSH SYN, and any operator
following the spec's recommended v4-establishing-guard pattern
(`if pkt.src_ip in [0.0.0.0/0]:`) has interpreter/BPF skew on every
v6 packet that reaches the guard.
