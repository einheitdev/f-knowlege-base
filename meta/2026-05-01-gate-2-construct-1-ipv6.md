---
type: meta
title: Phase 2 Gate 2 — Construct 1 (IPv6 fields) sign-off
date: 2026-05-01
agent: f-phase2 implementation agent
related_findings: 3  # all hunt findings status:fixed
---

# Gate 2 sign-off — Construct 1 (IPv6 fields)

## Outcome

`pkt.src_ip6` / `pkt.dst_ip6`, IPv6 literals (RFC 5952 canonical with §5 IPv4-mapped carve-out), IPv6 CIDR (`/0..128`), IPv6 CIDR/literal lists, `icmp6` proto keyword — all shipped.

## Per-construct sign-off checklist (CONSTRUCTS.md)

- [x] **AST + grammar + analyzer + interpreter + emitter** all updated. ~400 lines of new code in `f/fwl/fwl/`.
- [x] **Per-construct corpus** under `f/fwl/tests/corpus/08_ipv6_fields/` — 22 cases (8 pass, 4 fail, 6 edge, 4 interaction). All 22 pass under `hone regress`.
- [x] **`hone regress`** green on construct corpus AND f/fwl/tests/corpus whole-tree (85/85). Whole-base kb corpus 294/302 — 8 failures are all pre-existing v0.1 loader-bug findings filed during the spec-review hunts; none are Construct 1 regressions.
- [x] **`hone fuzz boundary_probing`**: 0 divergent / 42 candidates.
- [x] **`hone fuzz oracle_divergence`** seeds 0/1/2/3 (count=1000 each): **0 divergent / 4042 candidates**.
- [x] **`hone hunt --target ../f/fwl/examples/v6_internal.fw --max-turns 80`**: 3 findings (1 high, 1 medium, 1 low), **all status: fixed**.
- [x] **`hone mutate` + `hone transfer`** for every confirmed finding: 0 new divergences.
- [x] **`hone tune`**: weights healthy (no strategy below 5% floor, none hoarding >60%): boundary_probing=0.243, oracle_divergence=0.243, hunt=0.220, mutate=0.147, transfer=0.147.
- [x] **`hone diff-impact` / construct map**: `compiler_construct_map.yaml` updated to add `ipv6` tag to `interpreter.py`, `emitter.py`, `pkt.py`; `FWL_V02_SPEC.md` and `PKT_V02_SPEC.md` added with v0.2 construct tags.

## Hunt findings (all fixed in this gate)

1. **`finding/2026-05-01-v02-interpreter-missing-v6-surface-activation-gate`** (high; layer: interpreter). The AST interpreter read v0.1-style fields (`pkt.proto`, `pkt.src_port`, `pkt.dst_port`, `pkt.tcp.syn`, `pkt.tcp.ack`) directly from a v6-builder packet's decoded dict regardless of whether the program activated the v6 parse path. Cross-oracle drift: BPF emitter correctly omitted v6 parse for v0.1-shaped programs (returns DROP), interpreter returned PASS. Fix: added `_gate_v6_packet_for_v01_program` to `interpreter.evaluate` mirroring `emitter._is_v6_active`.

2. **`finding/2026-05-01-v6-internal-fw-ndp-dad-and-rs-dropped-against-stated-intent`** (medium; layer: user-rule). The `v6_internal.fw` rule `allow if pkt.proto == icmp6 and pkt.src_ip6 in fe80::/10` silently dropped Neighbor Solicitation (DAD per RFC 4862 §5.4.2) and the initial Router Solicitation (RFC 4861 §6.3.7) packets, both of which use `::` as the source. Fix: widened the rule to `pkt.src_ip6 in [fe80::/10, ::/128]`.

3. **`finding/2026-05-01-pkt-loader-accepts-noncanonical-ipv6-in-builder-fields`** (low; layer: loader). `pkt._ipv6_to_bytes` accepted any RFC 4291 valid IPv6 string (uppercase hex, expanded zero hextets, `::ffff:0102:0304`); PKT_V02_SPEC.md mandates RFC 5952 canonical only. Fix: added `_canonical_ipv6_for_builder` mirroring `parser._canonical_ipv6` and the canonical-form check in `_ipv6_to_bytes`. The PoC `.pkt` was updated to assert `expected.loads: false` as anti-regression.

## Implementation summary

| Layer | File(s) | Change |
|---|---|---|
| Grammar | `fwl/grammar.lark` | `IPV6`, `IPV6_CIDR`, `IP6_FIELD` tokens; `icmp6` added to `PROTO_KEYWORD`; `ipv6_cidr`/`ipv6_cidr_list` productions; trailing-`:` regex avoidance for IPv6-followed-by-Tier-2-`if`-suffix. |
| AST | `fwl/ast.py` | `Ipv6Literal`, `Ipv6CidrLiteral`, `Ipv6CidrListLiteral`, `Proto.ICMP6`, `IP6_FIELDS`. |
| Parser | `fwl/parser.py` | RFC 5952 canonicality check (rules 1–4, including §5 IPv4-mapped); CIDR /0..128 validation. |
| Analyser | `fwl/analyzer.py` | Cross-family type rules; ordered-comparison rejection; `IP6_FIELDS` in proto-guard table; `ICMP6` in `_ALL_PROTOS`. |
| Interpreter | `fwl/interpreter.py` | 128-bit address compare via `ipaddress.IPv6Address`; v6 CIDR matching via masked compare; v6-surface activation gate per PKT_V02_SPEC. |
| .pkt builder | `fwl/pkt.py` | `tcp6`/`udp6`/`icmp6` builders; IPv6 frame layout (40-byte fixed header, EtherType `0x86DD`); RFC 5952 canonicality check on builder field values. |
| Emitter | `fwl/emitter.py` | Dual-stack EtherType branch; v6 parse prelude; 128-bit splits as 64-bit hi/lo; v6 CIDR via two 64-bit masked compares; `v6_ok` gate (prevents v4 packets from spuriously matching `pkt.src_ip6 == ::`). |
| Examples | `fwl/examples/v6_internal.fw` | New v6-aware internal-network dogfood. |
| Corpus | `fwl/tests/corpus/08_ipv6_fields/` | 22 `.pkt` cases. |

## What's next

Construct 2 (`geoip()` built-in) is the next phase per `planning/PHASE_2_PLAN.md`. The Construct 1 work establishes the corpus-machinery and builder additions that Constructs 2 and 3 reuse — IPv6 frames, the dual-stack prelude, the 128-bit comparison machinery — so the per-construct workflow continues with `geoip()` next.
