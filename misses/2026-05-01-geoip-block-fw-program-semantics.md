---
id: miss/2026-05-01-geoip-block-fw-program-semantics
type: miss
protocol: [tcp, udp, icmp, icmp6]
builtins: [geoip]
severity: n/a
layer: user-rule
pattern_tags: [geoip, cross-family, v6-activation, rule-ordering, allowlist, blocklist]
status: ruled-out
source_file: fwl/examples/geoip_block.fw
created: 2026-05-01
---

# 2026-05-01-geoip-block-fw-program-semantics

## Hypotheses checked

### H1 — Cross-family `pkt.proto == tcp/udp` semantic on v6 frames

The program's top comment claims:

> Activates the v6 parse path via the second drop rule, so
> `pkt.proto == tcp/udp` here matches both v4 and v6 frames per
> the proto-enum table in FWL_V02_SPEC.md.

I sent a `tcp6` (v6 TCP) packet from a non-geoip-listed source to
dst_port 80 and verified both oracles reach the rule
`allow if pkt.proto == tcp and pkt.dst_port in [80, 443]`. The
v6 parse activation works as the spec describes: rule 2's
`pkt.src_ip6` reference flips `_is_v6_active` (emitter) and
`_program_touches_v6_surface` (interpreter), the v6 prelude
populates `proto`/`dst_port` from the IPv6 fixed header, and
both oracles agree on PASS.

UDP (port 53) symmetrically: `udp6(dst_port=53)` from a neutral
source allows in both oracles.

### H2 — Interpreter / emitter v6-activation predicate parity

`fwl/fwl/interpreter.py:_condition_touches_v6` and
`fwl/fwl/emitter.py:_is_v6_active` are textually parallel: both
detect `IP6_FIELDS`, `Ipv6Literal`/`Ipv6CidrLiteral`/
`Ipv6CidrListLiteral` operands, the `ICMP6` proto keyword, and
mixed `ListLiteral` items containing v6 literals. Neither walks
into a `geoip(...)` call — but the analyzer binds each
`GeoIp.family` from the LHS field type, so a `pkt.src_ip6 in
geoip(...)` is already covered by the `IP6_FIELDS` branch on
that same comparison. No drift between the two predicates.

### H3 — `default drop` covers the unallowed-and-unguarded path

A v4 packet from a country in neither blocklist nor allowlist
to a port outside `[80, 443, 53]` should reach `default drop`.
Both oracles agree. No bug.

### H4 — Allowlist semantic risk (US/DE source can do anything)

The allow rules `allow if pkt.src_ip in geoip(US, DE)` and the
v6 sibling place no port/proto restriction on US/DE traffic, so
in principle an SSH brute-force attempt from a US source is
allowed. The program's top comment says `small allowlist (US,
DE)` and treats them as fully trusted; this is **by design**,
not a bug. If a future revision wants to restrict US/DE traffic
to specific services, the rules would need a port guard — but
the current behaviour matches the comment.

### H5 — Cross-family `pkt.src_ip == 0.0.0.0` on v6 frames

The BPF emitter initialises `src_ip = 0` and updates it only
inside the IPv4 EtherType branch. A v6 packet leaves `src_ip` at
0, so a hypothetical rule `if pkt.src_ip == 0.0.0.0` would match
in BPF (cross-family false positive) while the interpreter
correctly returns false (the v6 builder dict has no `src_ip`
key, and even if the v01-shape gate isn't applied, `actual is
None → return False`). However, `geoip_block.fw` does not write
such a rule; its only v4 `src_ip` reads are `in geoip(...)`, and
real geoip data does not contain `0.0.0.0/0`-style prefixes for
RU/CN/KP/US/DE, so the cross-family false-positive does not
surface here. Filed as a separate hypothesis under
`/tmp/hunt-pkts/` for the next hunt — the bug is real but it's a
property of the emitter prelude, not of `geoip_block.fw`.

## Why nothing here is a bug for this program

Each hypothesis either:

- behaves as the spec describes (H1, H2, H3),
- matches the program's top-comment intent (H4), or
- is structurally unreachable from `geoip_block.fw`'s rules (H5).

The one real Class-A divergence I found while hunting on this
program is documented separately in
`finding/2026-05-01-bpf-runner-skips-geoip-map-population` — the
test harness (not this program) doesn't populate the BPF geoip
LPM trie maps from the `.pkt` `geoip_data` block, so any
`geoip(...)` rule whose lookup result drives the action shows
interpreter-vs-BPF disagreement. That's a runner bug surfacing
on every geoip program, not one specific to `geoip_block.fw`.

## What this hunt did NOT cover

- Behaviour with non-default ICMPv6 builder fields. The v0.2
  builder fixes ICMPv6 Echo Request (type 128, code 0) and the
  spec defers parameterisation to v0.3.
- IPv6 extension headers. Default `tcp6` builders never produce
  Hop-by-Hop / Fragment headers; the spec marks the
  ext-hdr-unreadable-L4 case as a hand-crafted-bytes corner that
  needs raw-byte support (deferred to v0.3 of `.pkt`).
- The cross-family `pkt.src_ip == 0.0.0.0` pseudobug (H5) on
  programs that *do* write such a rule. Worth a separate hunt
  on a synthetic test program; out of scope for `geoip_block.fw`
  because the rule shape doesn't appear here.
- Behaviour with very large geoip prefix counts approaching the
  65 536-entry budget. The spec lists this as a bundle-time
  validation, not a runtime concern.
- The daemon-side LPM trie population path
  (`fd`/`bundle attach`). Out of scope for `.pkt` testing; the
  hunt under `findings/.../bpf-runner-skips-geoip-map-population`
  is specifically the test harness's gap.
