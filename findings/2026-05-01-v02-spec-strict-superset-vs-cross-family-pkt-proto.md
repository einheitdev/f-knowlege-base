---
id: finding/2026-05-01-v02-spec-strict-superset-vs-cross-family-pkt-proto
type: finding
protocol: [tcp, udp]
builtins: []
severity: high
layer: spec
pattern_tags: [spec-contradiction, ipv6, strict-superset, cross-family, pkt-proto, regression-hazard]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: [finding/2026-05-01-v02-spec-tier2-pkt-proto-no-dominator-rule]
---

# 2026-05-01-v02-spec-strict-superset-vs-cross-family-pkt-proto

## Summary

The v0.2 spec makes two promises that cannot both be true:

1. **Strict-superset / identical semantics.** `FWL_V02_SPEC.md:9-12`:
   > v0.2 is a strict superset of [v0.1](FWL_V01_SPEC.md): every v0.1
   > program is a valid v0.2 program with identical semantics.

2. **Cross-family `pkt.proto`.** `FWL_V02_SPEC.md:161-167` proto enum
   table:
   > | `tcp` | IPv4 with `proto == 6`, or IPv6 with `next_header == 6`
   >   (no ext hdrs) | 6 |
   > | `udp` | IPv4 with `proto == 17`, or IPv6 with `next_header == 17`
   >   (no ext hdrs) | 17 |

A purely-v0.1 program — e.g. `drop if pkt.proto == tcp\ndefault allow`
— behaves differently on a v6 TCP packet under v0.1 versus v0.2:

| Version | EtherType `0x86DD` (v6) TCP packet | Outcome |
|---|---|---|
| v0.1 | EtherType check fails; rule does not match (`FWL_V01_SPEC.md:194-198`); fall-through to `default allow` | `XDP_PASS` |
| v0.2 (per the proto-enum table) | `pkt.proto == tcp` matches because v0.2 reads next_header on v6 packets | `XDP_DROP` |

Same source, different action for the same packet. The strict-superset
claim is false for any v0.1 program that mentions `pkt.proto == tcp`,
`pkt.proto == udp`, `pkt.src_port`, `pkt.dst_port`, `pkt.tcp.syn`, or
`pkt.tcp.ack` — i.e. essentially every non-trivial v0.1 program.

Class A (interpreter and BPF will report different verdicts depending
on which of the two paragraphs they implemented against, and v0.1
corpus cases with v6 packet inputs may now flip verdict). Filed as
spec-layer because the root cause is the contradiction between the
two paragraphs.

## The two readings the spec invites

### Reading A — v6 parse always active, cross-family `pkt.proto`

`pkt.proto`'s semantics is the proto-enum table at `:161-167`. The
emitter unconditionally emits the v6 fixed-header parse; on a v6 TCP
packet, `pkt.proto == tcp` evaluates true.

Under this reading every v0.1 program with a `pkt.proto == tcp|udp`
clause changes behaviour for v6 packets. The strict-superset claim at
`:9-12` is broken.

### Reading B — v6 parse activates only when v6 fields are touched

`FWL_V02_SPEC.md:907-912` (Compilation section):

> For a program touching IPv6 fields:
>
> - The analyser activates the IPv6 parse path in the emitter.

A v0.1 program touches no v6 fields, so the v6 parse path is not
active. On a v6 packet, the v0.1-style EtherType check fails and the
rule falls through, identical to v0.1 behaviour.

But under this reading the proto-enum table at `:161-167` is **wrong
for v0.1 programs**: `pkt.proto == tcp` matches v4 only, not v4+v6.
The same source line has different semantics depending on whether the
rest of the program touches v6 fields — and the spec never says so
explicitly. A reader who lifts the proto-enum table verbatim into a
v0.1 program assumes cross-family matching that the program will not
deliver.

### Reading C — `pkt.proto` is family-conditional only inside v6-touching programs

A hybrid: the proto-enum table is the v0.2 semantics *for programs
that touch v6 fields*; for v0.1-only programs, the v0.1 semantics
apply. The spec does not say this anywhere, but it is the only reading
that satisfies both `:9-12` and `:907-912`.

Under this reading there are now *two* answers to "what does
`pkt.proto == tcp` match" depending on context, and the spec does not
spell out the trigger. A program that adds a single `pkt.src_ip6 == ::1`
condition silently changes the semantics of every other
`pkt.proto == tcp` clause in the same file.

The three readings produce three different oracle results on the same
v0.1 source running against a v6 TCP packet. The v0.1 corpus already
contains programs of this shape; running them through the v0.2
oracle is undefined.

## Concrete example

A v0.1 program that the v0.1 corpus already exercises:

```python
@xdp(eth0)

drop if pkt.proto == tcp and pkt.dst_port == 22
default allow
```

A `.pkt` test that fires an IPv6 TCP packet to port 22:

| Implementer reading | Verdict |
|---|---|
| A (cross-family always) | `XDP_DROP` |
| B (v6 parse off for v0.1 programs) | `XDP_PASS` (default allow) |
| C (context-dependent) | depends on whether *any* line in the file touches a v6 field |

The v0.1 corpus that hone uses as its regression oracle was written
against v0.1 semantics — i.e. reading B's outcome (`XDP_PASS`). If the
v0.2 implementation lands on reading A, the entire v0.1 corpus
silently regresses on v6-packet inputs. The spec gives no rule the
hone agent can cite to call this a bug versus a feature.

The v0.1 corpus does not currently contain v6 packets (v0.1 had no
v6), but the moment the v0.2 corpus adds one — and `PKT_V02_SPEC.md`
explicitly adds v6 builders for this purpose — the contradiction
becomes a verifier-side disagreement rather than a hypothetical.

## Why this matters

The strict-superset clause is one of v0.2's load-bearing
guarantees — it is the basis for the methodology rule "the v0.1
corpus is the regression oracle for every Phase 2 PR"
(`planning/PHASE_2_PLAN.md`). If v0.1 programs change behaviour on
v6 packets under v0.2, the regression oracle no longer measures what
it claims to.

## Proposed fix

Pick one of the readings and make it explicit. The simplest is
**Reading B**, which preserves the strict-superset claim and is
already half-stated at `:907-912`. The fix:

1. Edit `FWL_V02_SPEC.md:161-167` to make the cross-family rows
   conditional on the v6 parse being active:

   > | Keyword | Matches when packet is | IPPROTO |
   > |---|---|---|
   > | `tcp` | IPv4 with `proto == 6` always; IPv6 with
   >   `next_header == 6` (no ext hdrs) **only when the program
   >   activates the v6 parse path — see Compilation** | 6 |

   And similarly for `udp`. `icmp` stays IPv4-only and `icmp6` stays
   IPv6-only as written.

2. Edit `FWL_V02_SPEC.md:907-912` to spell out the *negative* case:

   > A program that touches **no** v6 fields (`pkt.src_ip6`,
   > `pkt.dst_ip6`, `pkt.proto == icmp6`, or any `geoip(...)`
   > comparison whose LHS is `ipv6`-typed) does **not** activate the
   > v6 parse path. For such programs, `pkt.proto == tcp|udp`
   > matches IPv4 only, exactly as in v0.1. This is what preserves
   > the v0.1 strict-superset guarantee.

3. Add an Edge case under "IPv6 Fields":

   > *v0.1-shaped program receives a v6 packet.* Because the program
   > does not touch any v6 field, the v6 parse path is inactive. The
   > v6 packet's EtherType (`0x86DD`) fails the v4 parse and the
   > rule falls through, exactly as in v0.1.

The alternative — Reading A — simplifies the emitter (one parse path,
unconditional) at the cost of breaking the strict-superset clause. If
that is the intent, then `:9-12` must be weakened to "every v0.1
program is a valid v0.2 program; semantics are identical for IPv4
packets" and the v0.1 corpus must add v6 packets to its regression
suite or be tagged as v4-only.

## Class

Class A (downstream cross-oracle disagreement on v6 packets) with a
spec-layer root cause. The interpreter and emitter cannot land on the
same answer until the spec picks a reading.
