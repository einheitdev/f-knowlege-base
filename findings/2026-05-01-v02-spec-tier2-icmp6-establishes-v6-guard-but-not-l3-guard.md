---
id: finding/2026-05-01-v02-spec-tier2-icmp6-establishes-v6-guard-but-not-l3-guard
type: finding
protocol: [icmp6]
builtins: []
severity: high
layer: spec
pattern_tags: [spec-contradiction, tier2, dominator-rule, ipv6, soundness, statement-position-read, transitivity]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: [finding/2026-05-01-v02-spec-tier2-pkt-proto-no-dominator-rule, finding/2026-05-01-v02-spec-tier2-pkt-proto-icmp-as-v4-guard-vs-cross-family-icmp]
---

# 2026-05-01-v02-spec-tier2-icmp6-establishes-v6-guard-but-not-l3-guard

## Summary

Two adjacent bullets in the Tier 2 dominator-rule paragraph
disagree on whether `pkt.proto == icmp6` is sufficient to dominate
a *statement-position* `pkt.proto` read.

`FWL_V02_SPEC.md:620-622` admits `pkt.proto == icmp6` as a
v6-establishing guard. `FWL_V02_SPEC.md:623` rejects **every**
`pkt.proto == <proto_keyword>` (without exception) as an
L3-establishing guard for a statement-position `pkt.proto` read.

In the same `if pkt.proto == icmp6:` block, the analyser must
therefore admit `addr = pkt.src_ip6` (v6 guard satisfied) and
reject `p = pkt.proto` (L3 guard not satisfied). The two answers
are mutually inconsistent: a true `pkt.proto == icmp6` comparison
proves the v6 parse succeeded, which proves the L3 parse
succeeded, which is exactly what the L3-guard exclusion claims is
not established.

Class B (spec contradiction). Two implementers reading the same
spec will land on different answers for whether the inner
statement is well-formed. The promoter analyser will reject the
inner read; a pragmatic analyser that prefers consistency with the
v6-guard list will accept it. The corpus oracle cannot settle the
question without the spec picking a reading.

## The contradiction

### Reading 1 — `pkt.proto == icmp6` IS a v6-establishing guard

`FWL_V02_SPEC.md:620-622`:

> A read of `pkt.src_ip6`, `pkt.dst_ip6` is valid only at program
> points dominated by a guard that establishes the packet is IPv6.
> Two ways to satisfy that guard:
>   - A condition that reads `pkt.src_ip6` or `pkt.dst_ip6`
>     directly (...).
>   - A condition that reads `pkt.proto == icmp6` (icmp6 only
>     exists on v6 packets).

The justification — "icmp6 only exists on v6 packets" — is sound.
`icmp6` is family-restricted in the proto-enum table at
`FWL_V02_SPEC.md:168` ("`icmp6` | 58 | IPv6 (no ext hdrs) only
when the program activates the v6 parse path"). So a true
`pkt.proto == icmp6` comparison implies (a) `pkt.proto` was
readable (short-circuit semantics) and (b) the family was IPv6
(family-restricted). Both (a) and (b) imply the v6 parse succeeded.

### Reading 2 — `pkt.proto == icmp6` is NOT an L3-establishing guard

`FWL_V02_SPEC.md:623`:

> A read of `pkt.proto` itself is valid only at program points
> dominated by an L3-establishing guard — a condition that reads
> any of `pkt.src_ip`, `pkt.dst_ip`, `pkt.src_ip6`, or
> `pkt.dst_ip6` (...). A `pkt.proto == <proto_keyword>` condition
> does **not** by itself satisfy the guard for a
> *statement-position* `pkt.proto` read — its reading of
> `pkt.proto` is the surface this rule constrains.

Read literally, *every* `pkt.proto == kw` is excluded from the L3
list — including `pkt.proto == icmp6`. The justification (the
condition "is the surface this rule constrains") is the
recursive-avoidance argument: if we admitted `pkt.proto == X` as
its own guard, the rule would be vacuous.

But that argument breaks down for family-restricted keywords. If
`pkt.proto == icmp6` is true, then by Reading 1, the v6 parse
succeeded — which means the L3 parse for the IPv6 family
succeeded — which means `pkt.proto` (the v6 next_header byte at
offset 14+6) was readable. The L3-guard exclusion ignores this
transitivity.

### The two readings disagree on a single program

```python
@xdp(eth0)

def firewall(pkt):
  if pkt.proto == icmp6:
    addr = pkt.src_ip6   # Reading 1: OK (v6 guard established)
    p    = pkt.proto     # Reading 2: ERROR (no L3 guard)
    if addr == ::1 and p == icmp6:
      drop
  allow
```

The two adjacent statement-position reads are governed by two
adjacent bullets in the same dominator paragraph, and the two
bullets give incompatible answers about the same dominating
condition.

| Statement | Bullet | Outcome |
|---|---|---|
| `addr = pkt.src_ip6` | `:620-622` (v6 guard list) | Accepted — `pkt.proto == icmp6` is a v6-establishing guard. |
| `p = pkt.proto` | `:623` (L3 guard list) | Rejected — `pkt.proto == icmp6` is not an L3-establishing guard per the literal rule. |

By any operational reading, both statements are sound on every
packet that reaches them: only v6 packets with `next_header == 58`
get inside, and on such a packet both `pkt.src_ip6` and
`pkt.proto` are readable. The spec rejects the second statement
anyway, on a recursive-avoidance argument that does not apply to
family-restricted keywords.

## Why this matters

1. **Two implementers diverge on the same program.** A literal
   reader of `:623` rejects the program; a reader who notices the
   transitivity (and the precedent set by `:620-622`) accepts it.
   The corpus has no way to lock the right answer until the spec
   picks one.

2. **The error message is operationally misleading.** The spec at
   `:718` says the rejection produces `error: 'pkt.proto' read on
   a path not guarded by an L3-establishing condition`. But on
   the program above, the path *is* L3-guarded — by transitivity
   through the v6-guard rule. The user reads the error and looks
   for a missing L3 read; the only fix is to add a *redundant*
   `pkt.src_ip6 in ::/0` guard around the body, which is a
   semantically identical no-op on the path that actually
   reaches the read.

3. **The asymmetry is invisible to v0.1-only readers.** A v0.1
   user (no v6 surface) sees `pkt.proto == icmp` excluded from the
   v4-guard list (per the `tier2-pkt-proto-icmp-as-v4-guard-vs-cross-family-icmp`
   fix) and naturally concludes "no proto keyword establishes
   either family." A v0.2 reader sees `pkt.proto == icmp6`
   admitted to the v6-guard list and naturally concludes "the
   proto check IS sufficient when the keyword is
   family-restricted." Neither reader sees the inconsistency
   because they do not cross-reference the L3-guard bullet.

## Two possible resolutions

| Resolution | What changes |
|---|---|
| (a) Admit family-restricted proto keywords to the L3 guard list. | Append to `:623`: "Exception: a `pkt.proto == <kw>` whose `<kw>` is family-restricted (`icmp6` is the only such keyword in v0.2) **does** satisfy the L3 guard for a statement-position `pkt.proto` read, by transitivity through the v6-establishing guard rule." This restores internal consistency and makes the spec match operational soundness. |
| (b) Drop `pkt.proto == icmp6` from the v6-guard list. | Symmetric resolution: tighten the v6-guard list at `:620-622` to admit only direct `pkt.src_ip6`/`pkt.dst_ip6` reads. Costs the spec the operationally-useful idiom `if pkt.proto == icmp6: drop pkt.src_ip6 ...`. |

(a) is the cleaner fix because it preserves the v6-guard rule's
operational utility and fixes the inconsistency. The change is one
sentence.

## Suggested corpus item

A `.pkt` test that exercises both branches of the disagreement.
The program compiles iff (a) is the chosen resolution; it fails to
compile iff (b) is. The two readings cannot agree on the
`expected.compiles` value, so the case is a discriminator.

```yaml
program: |
  @xdp(eth0)

  def firewall(pkt):
    if pkt.proto == icmp6:
      addr = pkt.src_ip6
      p    = pkt.proto
      if addr == ::1 and p == icmp6:
        drop
    allow

# Reading 1 + transitivity:  expected.compiles: true
# Reading 2 (literal :623):  expected.compiles: false
expected:
  compiles: true        # picked once the spec resolves
  compile_error_pattern: ""
```

A second, narrower discriminator using only the proto read:

```yaml
program: |
  @xdp(eth0)
  def firewall(pkt):
    if pkt.proto == icmp6:
      p = pkt.proto
    allow

# Reading 1 + transitivity: expected.compiles: true
# Reading 2 (literal):      expected.compiles: false
```

## Class

Class B (spec contradiction; spec layer). Two adjacent bullets in
the same paragraph give incompatible answers about whether
`pkt.proto == icmp6` dominates a statement-position `pkt.proto`
read. Will promote to Class A in the corpus the moment the
analyser is implemented and a pragmatic implementer picks one
reading while a strict implementer picks the other.
