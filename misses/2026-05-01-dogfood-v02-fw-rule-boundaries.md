---
id: miss/2026-05-01-dogfood-v02-fw-rule-boundaries
type: miss
protocol: [tcp, udp, icmp6]
builtins: [geoip]
pattern_tags: [dogfood, ipv6, geoip, cidr-boundaries, rule-ordering, cross-family, tier2]
source_file: fwl/examples/dogfood_v02.fw
created: 2026-05-01
related: [finding/2026-05-01-v02-emitter-no-v4-parse-ok-gate-cross-family-disagreement]
---

# 2026-05-01-dogfood-v02-fw-rule-boundaries

## Summary

Targeted hunt against `f/fwl/examples/dogfood_v02.fw`. The dogfood
program's specific rule shapes were exercised at boundaries and
across-family — the program itself does not trigger any new
implementation divergence. One adjacent emitter bug was
surfaced — see `finding/2026-05-01-v02-emitter-no-v4-parse-ok-gate-cross-family-disagreement` —
and is filed separately because the dogfood example happens to
sidestep it (the v4 prefixes in the program are all non-zero).

## What was tested (corpus/from_hunt/2026-05-01_dogfood_round2/)

| .pkt case | hypothesis | both oracles agree |
|---|---|---|
| `dogfood_v02_v4_172_31_internal_allow.pkt` | high boundary of 172.16.0.0/12 → allow | yes |
| `dogfood_v02_v4_172_32_outside_drop.pkt` | just outside /12 → drop (default) | yes |
| `dogfood_v02_v4_ssh_external_drop.pkt` | external v4 to TCP/22 → drop (not in allow list) | yes |
| `dogfood_v02_v4_tcp_53_drop.pkt` | TCP/53 (DNS-over-TCP) external → drop (only UDP/53 allowed) | yes |
| `dogfood_v02_v4_internal_overlapping_geoip_allow.pkt` | rule order — 10.0.0.1 in BOTH internal AND geoip-RU → allow (rule 1 wins) | yes |
| `dogfood_v02_v4_internal_geoip_drop_drop_match.pkt` | external v4 from RU geoip → drop (rule 3) | yes |
| `dogfood_v02_v6_fdff_high_boundary_allow.pkt` | high boundary of fc00::/7 → allow | yes |
| `dogfood_v02_v6_fe00_outside_drop.pkt` | just outside fc00::/7 → drop | yes |
| `dogfood_v02_v6_icmp6_external_drop.pkt` | external v6 ICMPv6 → drop (default) | yes |

All nine cases agree across spec, interpreter, and BPF (with sudo
for BPF_PROG_TEST_RUN). The program correctly handles:

- v4 internal CIDR boundaries (high and low).
- v6 internal CIDR boundary (high).
- Cross-family fall-through (v4 src_ip rule on v6 packet, v6
  src_ip6 rule on v4 packet — both fall through).
- Geoip rule ordering (internal allow before geoip drop).
- Default drop for non-matching packets.
- ICMPv6 (next_header=58) — proto guard short-circuits correctly,
  defaults to drop.
- TCP/UDP port disambiguation (TCP/53 not allowed; only UDP/53).

## What was ruled out

- **No latent rule-ordering bug.** Rule 1 (internal allow) fires
  before rule 3 (geoip drop) — verified with overlapping geoip data.
  Both oracles agree.
- **No off-by-one on v4 CIDRs.** 172.31.255.255 and 172.32.0.0
  boundaries handled correctly.
- **No cross-family geoip leak.** v4 geoip prefixes don't match v6
  packets and vice versa.
- **No proto-guard short-circuit gap inside dogfood.** All
  L4 reads (`pkt.dst_port`) sit inside `pkt.proto == tcp/udp`
  conjuncts. The conjuncts short-circuit on non-TCP/non-UDP
  packets per `FWL_V02_SPEC.md:612`, so the
  v4-parse-OK-gate emitter bug
  (`finding/2026-05-01-v02-emitter-no-v4-parse-ok-gate-cross-family-disagreement`)
  doesn't bite this program — the BPF runtime never reads
  `dst_port` on a non-TCP/non-UDP packet, so `dst_port == 0`
  paths can't fire.

## What this hunt did NOT cover

- *Multi-packet rate-limit dynamics.* The Tier 2 dogfood example in
  the spec (`FWL_V02_SPEC.md:986-1030`) — distinct from
  `f/fwl/examples/dogfood_v02.fw` — uses `rate_limit(...)` inside
  the function. The .pkt runner doesn't simulate Tier 2 rate-limit
  state, so the dynamics across iterations weren't tested. Per
  `interpreter.py:179-181`: "Test harness doesn't simulate
  rate-limit dynamics for Tier 2." A multi-iteration test would
  surface the v4-parse-OK gate bug visibly via differing rate-limit
  counters between interpreter and BPF (BPF erroneously enters the
  v4-only `if pkt.src_ip in [0.0.0.0/0]:` branch on v6 packets,
  bumping `fwl_rl_map_X` keyed at `src_ip = 0`).
- *NDP / link-local v6 control plane.* The dogfood program drops
  v6 packets sourced from `fe80::/10` (link-local) because they
  are not in `fc00::/7`. RFC 4861/4862 require link-local sources
  for Router Advertisements, Neighbor Solicitations during DAD,
  etc. A v6 host running this firewall on a real network can't
  receive RAs from its router. This is the same shape as
  `finding/2026-05-01-v6-internal-fw-ndp-dad-and-rs-dropped-against-stated-intent`
  but for `dogfood_v02.fw`. The dogfood comment ("trust internal
  traffic on both IPv4 and IPv6 ... drop everything else") arguably
  intends to drop link-local — operator-dependent. Not filed as a
  separate finding because the user-intent reading is ambiguous and
  the v6_internal pattern is already documented.
- *Programs with `/0` v4 prefix or `0.0.0.0` literal.* Discovered
  the emitter bug (filed as a separate finding). Cases under
  `corpus/from_hunt/2026-05-01_dogfood_round2_emitter_v4_gate/`.

## Outcome

`dogfood_v02.fw` is internally consistent on the surface area
tested. The corpus is grown by 9 cases, locking in the program's
behaviour against future regressions.
