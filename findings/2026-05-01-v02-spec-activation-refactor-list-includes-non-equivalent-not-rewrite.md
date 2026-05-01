---
id: finding/2026-05-01-v02-spec-activation-refactor-list-includes-non-equivalent-not-rewrite
type: finding
protocol: [tcp, udp, icmp, icmp6]
builtins: []
severity: medium
layer: spec
pattern_tags: [spec-contradiction, ipv6, activation-rule, refactor-invariance, negation, pkt-proto]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: [finding/2026-05-01-v02-spec-v6-activation-rule-only-covers-equality-not-inequality-or-in-or-locals, finding/2026-05-01-v02-spec-proto-equality-table-vs-opaque-byte-equality]
---

# 2026-05-01-v02-spec-activation-refactor-list-includes-non-equivalent-not-rewrite

## Summary

The Compilation section's IPv6-activation paragraph at
`FWL_V02_SPEC.md:936` claims:

> Refactors that preserve the program's semantics (e.g. hoisting
> `pkt.proto` into a Tier 2 local, **swapping `==` for `!=` with
> negation**, expanding a list-membership `in` into an `or`-chain)
> preserve activation.

The middle item — "swapping `==` for `!=` with negation" — is **not**
a semantics-preserving refactor in v0.2. The proto-equality
*Equality semantics* paragraph at `FWL_V02_SPEC.md:167` says so
explicitly:

> `pkt.proto != <kw>` is the negation: true when the read succeeded
> and the byte differs, *false* when the read failed (so `not
> (pkt.proto == tcp)` is not the same as `pkt.proto != tcp` on a
> non-IP frame; the negation rule still requires the read to
> succeed).

The two paragraphs disagree directly. `:936` lists the
`==`/`not(...)`/`!=` rewrite as semantics-preserving while citing
it as evidence that activation is "semantic, not textual"; `:167`
lists exactly that rewrite as **non-equivalent** (with a worked
example: non-IP frame).

Class B (spec internal contradiction). Both paragraphs are
load-bearing — `:167` defines runtime behaviour, `:936` defines
analyser behaviour — and they cannot both be right.

## The contradiction in detail

### Reading 1 — `:167` non-equivalence (runtime semantics)

For a non-IP frame (ARP, LLDP, MPLS, raw 802.1Q with non-IP inner,
truncated v4 frame at the IP-header read):

| Expression | v0.2 truth value |
|---|---|
| `pkt.proto == tcp` | false (read failed) |
| `pkt.proto != tcp` | false (`!=` requires the read to have succeeded) |
| `not (pkt.proto == tcp)` | true (negation of false) |
| `not (pkt.proto != tcp)` | true (negation of false) |

So `pkt.proto == tcp` and `not (pkt.proto != tcp)` differ on every
non-IP frame. Same for `pkt.proto != tcp` vs `not (pkt.proto ==
tcp)`. In both directions, the rewrite that `:936` calls
"semantics-preserving" actually flips the verdict on non-IP frames.

### Reading 2 — `:936` claims the rewrite is semantics-preserving

The activation-rule paragraph lists three example refactors as
demonstrations of "semantic, not textual" activation:

1. Hoisting `pkt.proto` into a Tier 2 local — verified in `:169`
   (Refactor invariance) and the related fixed finding
   `finding/2026-05-01-v02-spec-proto-equality-table-vs-opaque-byte-equality`.
2. **Swapping `==` for `!=` with negation** — disagrees with `:167`.
3. Expanding a list-membership `in` into an `or`-chain — `pkt.proto
   in [tcp, udp]` ↔ `pkt.proto == tcp or pkt.proto == udp`. These
   are logically equivalent on IP frames; on non-IP they both
   evaluate false (each `==` short-circuits to false on read
   failure). Equivalent.

Item 2 is the odd one out. The cite of three rewrites was meant to
tighten "activation tracks semantics", but item 2 is exactly a
refactor that changes semantics on non-IP frames.

## Concrete `.pkt` evidence

```yaml
# corpus/from_hunt/2026-05-01_v02_spec/proto_neq_vs_not_eq_non_ip.pkt
name: "pkt.proto != tcp vs not (pkt.proto == tcp) on a non-IP (ARP) frame"
# Hypothesis per FWL_V02_SPEC.md:167:
#   pkt.proto != tcp on ARP -> false (read failed; != requires success)
#   not (pkt.proto == tcp) on ARP -> true (== false; not(false) = true)
# So the two rules below behave DIFFERENTLY on an ARP frame.
source_fw_a: |
  @xdp(eth0)
  drop if pkt.proto != tcp
  default allow

source_fw_b: |
  @xdp(eth0)
  drop if not (pkt.proto == tcp)
  default allow

test_packet:
  builder: arp()       # EtherType 0x0806 — non-IP

expected_a:
  compiles: true
  bpf_action: allow    # `!= tcp` is false on ARP, fall-through to default
expected_b:
  compiles: true
  bpf_action: drop     # `not(== tcp)` is true on ARP, rule fires
```

The two programs are textually a `==`-↔-`!=`-with-negation rewrite
of each other; they produce *different* XDP verdicts on the same
ARP packet. So they are not semantics-equivalent — yet `:936` lists
that exact rewrite as "semantics-preserving."

## Why this matters

Three concrete failure modes:

1. **Implementer surface shrinks if `:936` is taken literally.** An
   analyser that treats the listed rewrites as "definitionally
   semantics-preserving" can:
   - Apply algebraic simplification (`not(==)` → `!=`) before
     emission, breaking the runtime semantics `:167` defines.
   - Use the simplified form to make activation decisions, even
     when the user-written form has different semantics on non-IP
     frames.
   Either choice silently regresses programs whose author wrote
   `not (pkt.proto == tcp)` deliberately to match non-IP frames.

2. **`.pkt` corpus authors lose their mental model.** A corpus
   author who wants to assert "this `.fw` and that `.fw` are
   equivalent" needs a clear list of which rewrites are equivalent.
   `:936`'s list says one thing, `:167` says another. The corpus
   either over-claims equivalence (matching `:936`) or
   under-claims it (matching `:167`); both states make the corpus
   a bad oracle.

3. **The companion fixed finding's invariant
   (`pkt.proto == kw` ↔ `p = pkt.proto; if p == kw:`) does NOT
   extend to the negated form.** The hoist invariance only holds
   when the comparison's L3-establishing guard dominates the
   read (per `:169`). If a user reads `:936` as endorsing the full
   `==`-↔-`!=`-with-negation refactor and applies it to the
   hoisted form (`p = pkt.proto; if p != kw:` ↔ `p = pkt.proto; if
   not (p == kw):`), they get yet another non-equivalence on non-IP
   frames. The spec's hoist guarantees do not cover this rewrite.

## Three possible resolutions

| Resolution | Effect |
|---|---|
| (a) Drop the `==`/`!=`-with-negation example from `:936`. | Smallest fix. The remaining two examples (hoist, list-`in`-to-`or`-chain) are correct. Sentence becomes: "Refactors that preserve the program's semantics (e.g. hoisting `pkt.proto` into a Tier 2 local, expanding a list-membership `in` into an `or`-chain) preserve activation." |
| (b) Replace the `==`/`!=` example with a different correct one. | Pick a refactor that *is* semantics-preserving — e.g. constant-folding a redundant parenthesis, or `cond_a or cond_a` → `cond_a`. Marginally informative; not necessary. |
| (c) Edit `:167` instead, to make `pkt.proto != kw` and `not (pkt.proto == kw)` equivalent. | Forces the runtime to define `!=` so it is true on read-failure (i.e. propagate the negation through the read-failure semantics). This is a substantive runtime change with downstream effects on the entire `pkt.proto != kw` corpus surface. Bad trade-off. |

(a) is the only option that doesn't perturb runtime behaviour or
spec readability. The `==`/`!=`-with-negation rewrite is genuinely
not semantics-preserving in v0.2; the activation paragraph's claim
otherwise is just a wording slip.

## Proposed fix (option a)

Replace `FWL_V02_SPEC.md:936` with:

> The activation check is *semantic*, not textual: the analyser
> runs after parsing and type inference, when the type of every
> operand is known. Refactors that preserve the program's semantics
> (e.g. hoisting `pkt.proto` into a Tier 2 local — see the Refactor
> invariance paragraph in the IPv6 Fields section — or expanding a
> list-membership `in` into an `or`-chain) preserve activation. Note
> that swapping `==` for `!=` with an outer `not` is **not**
> semantics-preserving in v0.2 (`pkt.proto != kw` and `not
> (pkt.proto == kw)` differ on non-IP frames per `:167`); applying
> that rewrite changes both the runtime behaviour and the
> activation-relevant surface.

That keeps the "activation is semantic" guarantee, names the two
correct example rewrites, and explicitly disclaims the third.

The companion `.pkt` corpus item above goes into
`corpus/from_hunt/2026-05-01_v02_spec/` as the regression lock for
the runtime non-equivalence.

## Class

Class B at the spec layer — two paragraphs in the same document
disagree on whether the same textual rewrite preserves program
semantics. The contradiction is one sentence wide; the fix is one
sentence wide.
