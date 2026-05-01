---
id: finding/2026-05-01-v02-spec-rate-limit-per-src-ip-with-v6-only-predicate-dead-rule
type: finding
protocol: []
builtins: [rate_limit]
severity: medium
layer: spec
pattern_tags: [spec-ambiguity, ipv6, rate-limit, tier1, dead-rule, cross-family, error-messages]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: [finding/2026-05-01-v02-spec-strict-superset-vs-cross-family-pkt-proto]
---

# 2026-05-01-v02-spec-rate-limit-per-src-ip-with-v6-only-predicate-dead-rule

## Summary

The v0.2 spec admits a Tier 1 rule whose predicate references only
v6-typed pkt fields and whose `limited by rate_limit(...)` modifier
uses a v4-only `per=` field (`src_ip`, `dst_ip`). Per the spec's own
runtime rule (`FWL_V02_SPEC.md:181-192`), every such rule is
provably-dead at compile time: the predicate fails on v4 frames (no
v6 read on v4) and the modifier fails on v6 frames (the `per=` field
is unreadable). The rule never fires on any packet.

The spec neither rejects the program nor warns. Three downstream
problems:

1. **Silent dead rule.** Operators write `drop if pkt.src_ip6 == ::1
   limited by rate_limit(10, per=src_ip)` and ship it; the rule does
   nothing on any input. v0.2 has no diagnostic for this.

2. **Interpreter / BPF drift.** Implementations may disagree on
   whether the rule "always falls through" (the spec's expected
   behaviour) or "always fires" (taking the predicate at face value
   and ignoring the per-field unreadability) on v6 packets. The spec
   text at `:188-192` is the only authority and it lives one section
   away from the rule grammar; an implementor could miss it.

3. **`.pkt` corpus has no path to test the case.** A test asserting
   "this rule is dead" cannot use `expected.compiles: false` (the
   spec doesn't reject) and cannot reliably use a single concrete
   packet (the rule's death-by-fall-through is true for every
   packet).

Class B (spec gap; spec layer). The rule should either be rejected at
compile time or the spec should explicitly endorse the dead-rule
shape (with a warning) and add a corpus test that locks the
fall-through behaviour for both families.

## The two spec rules that combine

### Predicate gate — v6 fields don't match v4 packets

`FWL_V02_SPEC.md:170-175`:

> A rule whose condition references `pkt.src_ip6` or `pkt.dst_ip6`
> does not match IPv4 packets (the parse for IPv6 fields fails on
> EtherType `0x0800`). Symmetrically, a rule whose condition
> references `pkt.src_ip` or `pkt.dst_ip` does not match IPv6
> packets. There is no compile error for accessing v6 fields
> without an explicit guard — the parse fall-through is the guard.

So a rule like `drop if pkt.src_ip6 == ::1 limited by rate_limit(10,
per=src_ip)` matches no v4 packets — its predicate falls through on
EtherType `0x0800`.

### Modifier gate — `per=src_ip` doesn't bucket v6

`FWL_V02_SPEC.md:181-192`:

> v0.2 keeps the same field list — `per=src_ip` continues to bucket
> on the IPv4 source address only. **Buckets are not shared between
> v4 and v6**: an IPv6 packet has no IPv4 source, and a rule with
> `per=src_ip` does not match IPv6 packets (the field is unreadable,
> so the per-bucket key is undefined and the rule falls through).

So the same rule does not match v6 packets either — the modifier's
per-field can't read on EtherType `0x86DD`.

### The rule's matching set is empty

A rule that matches **no** v4 packet (predicate gate) and **no** v6
packet (modifier gate) matches nothing on any frame the program will
see. Note that EtherType filtering is exhaustive for v0.2 (the spec
treats every non-IPv4-non-IPv6 EtherType as fall-through anyway).

Symmetric forms are equally dead:

| Predicate | Modifier `per=` | Why dead |
|---|---|---|
| Any field touching `pkt.src_ip6` or `pkt.dst_ip6` | `src_ip` or `dst_ip` | predicate excludes v4; modifier excludes v6 |
| Any field touching `pkt.src_ip` or `pkt.dst_ip` | (any) | predicate excludes v6; modifier with `per=src_ip6`/`per=dst_ip6` is already a compile error per `:240`, but no symmetric case exists yet for v4 modifiers — those just work on v4 |

The first row is the new dead-rule shape v0.2 introduced (it cannot
exist in v0.1 because v0.1 has no v6 fields).

## Concrete program the spec gives no guidance on

```python
@xdp(eth0)

# Operator's intent: rate-limit IPv6 sources hitting ::1 to 10/sec.
drop if pkt.src_ip6 == ::1 limited by rate_limit(10, per=src_ip)

default allow
```

For *every* packet:
- v4: predicate fails (no v6 field on v4) → rule does not match → fall through.
- v6: predicate matches (suppose), modifier's `per=src_ip` fails to bucket on v6 → rule falls through (`:188-192`).

Operator's expectation: rate-limit v6 sources to ::1.
Actual behaviour per spec: rule does nothing.

The spec has no compile-error row for this combination. A v0.2-conforming compiler must either:

- (a) Reject the program with a new error (which the spec doesn't define).
- (b) Accept it and produce a program whose v6 packets to ::1 silently exceed any rate (because the rule is dead).
- (c) Accept it with a warning (the spec doesn't define one either).

Three implementations will land on three different answers, and
the operator has no spec text to point at.

## Why this matters

Three angles, in order of severity:

1. **Operator footgun.** The natural intent — rate-limit IPv6 traffic
   from a specific destination — has no working v0.2 spelling. The
   spec defers `per=src_ip6`/`per=src_addr` to v0.3 (`:1014-1015`).
   A v0.2 user reaching for the obvious rule writes the dead-rule
   form, ships it, and learns the hard way that no rate limit fired.

2. **`.pkt` regression authority.** `hone regress` runs the corpus
   against three oracles. For a dead rule, both oracles must agree on
   "fall through to default" for every packet — but the spec's silence
   means the analyser may reject one program (option a), accept and
   warn another (option c), or accept silently (option b). The corpus
   cannot have a single test that locks the behaviour without the
   spec picking a reading.

3. **The strict-superset claim is unaffected** (this is *not* a v0.1
   regression; v0.1 has no v6 fields), but the spec's "v0.2 is a
   strict superset" framing implicitly raises the bar: every new
   construct should be diagnosable when used incoherently. v0.1's
   `per=src_ip` was incoherent only when combined with truncated
   packets, which v0.1 already handles (rule falls through). v0.2
   adds a *programmatic* incoherent combination — predicate-family
   vs modifier-family — that is statically detectable.

## Three possible resolutions

| Resolution | Effect |
|---|---|
| (a) Compile error. | Add an analyser pass that scans every Tier 1 rule with a `limited by rate_limit(...)` modifier; if the rule's predicate references only v6 fields and the modifier's `per=` is `src_ip` or `dst_ip` (or vice versa), emit `error: rate_limit per=<field> cannot bucket the rule's matched family — the predicate matches only <ipv6> traffic but per=<field> is <ipv4>-only. Use a v4 predicate or wait for v0.3's per=src_ip6.`. |
| (b) Compile warning. | Same detection, weaker diagnostic: `warning: rule predicate matches only <ipv6> traffic but rate_limit per=<field> is <ipv4>-only; the rule will never fire. Drop the modifier or restructure.`. The user's program still ships; intent vs reality is logged. |
| (c) Status quo + corpus lock. | Accept the program. Add a corpus case that fires v6 packets matching the predicate and asserts both oracles produce `XDP_PASS` (the rule fell through to `default allow`). Document at `:188-192` that this combination is silently a no-op and operators must avoid it. |

(a) is the cleanest user-facing answer; (b) is the lowest-risk
analyser change; (c) is the smallest spec change but leaves the
footgun in place.

## Proposed fix (option a — error)

Add to `FWL_V02_SPEC.md:229-241` (Compile errors table for IPv6
Fields):

| Condition | Error |
|---|---|
| Tier 1 rule whose predicate references only v6 pkt fields with `limited by rate_limit(..., per=src_ip)` or `per=dst_ip` | `error: rate_limit per=<field> cannot bucket this rule — the predicate matches only IPv6 traffic but per=<field> reads an IPv4-only address. Restructure the rule, or wait for v0.3's per=src_ip6.` |

And add an Edge case under "IPv6 Fields":

> *Cross-family rate-limit incoherence.* A rule whose predicate is
> v6-only and whose `per=` is v4-only (or vice versa) matches no
> packet on any frame. The compiler rejects this combination at
> analysis time per the Compile errors table. The intended forms
> in v0.2 are: v4 predicate with v4 `per=`, or v6 predicate with
> v4 `per=` only when the operator deliberately wants no rate
> limit (write the rule without the modifier instead).

The implementation already classifies each predicate's address
family (the dominator-rule analyser walks the AST). Adding a check
when emitting a Tier 1 rule with a `limited by` modifier is one
extra pass over the same data.

## Class

Class B — spec gap. The two referenced paragraphs (`:170-175` and
`:181-192`) define the runtime behaviour for each family
independently, but their composition produces a provably-dead rule
that the spec neither rejects nor warns about. Spec-layer.
