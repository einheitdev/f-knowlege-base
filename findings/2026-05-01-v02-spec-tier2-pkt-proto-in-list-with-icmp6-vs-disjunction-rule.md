---
id: finding/2026-05-01-v02-spec-tier2-pkt-proto-in-list-with-icmp6-vs-disjunction-rule
type: finding
protocol: [tcp, udp, icmp, icmp6]
builtins: []
severity: high
layer: spec
pattern_tags: [spec-contradiction, tier2, dominator, ipv6-guard, in-list, disjunction, family-restricted]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: [finding/2026-05-01-v02-spec-tier2-dominator-rule-negation-and-else-not-addressed, finding/2026-05-01-v02-spec-v6-activation-rule-only-covers-equality-not-inequality-or-in-or-locals, finding/2026-05-01-v02-spec-proto-in-membership-contradicts-type-rule-and-error-table]
---

# 2026-05-01-v02-spec-tier2-pkt-proto-in-list-with-icmp6-vs-disjunction-rule

## Summary

The Tier 2 dominator rule's L3-guard clause at
`FWL_V02_SPEC.md:622-624` lists two ways an `icmp6`-bearing
condition establishes the v6-family guard:

> A condition that uses a *family-restricted* proto keyword on a
> `pkt.proto` comparison — `pkt.proto == icmp6` or
> `pkt.proto in [icmp6, ...]` (icmp6 is the only family-restricted
> keyword in v0.2; it matches IPv6 only). Reaching such a
> comparison's then-branch implies (a) the `pkt.proto` read
> succeeded — i.e. the L3 parse for the inferred family succeeded
> — and (b) the family is the keyword's family.

The form `pkt.proto in [icmp6, ...]` is ambiguous on what fills
the `...`. Two readings are spec-conformant:

1. **Strict reading.** `[icmp6, ...]` is shorthand for "an
   in-list whose every element is family-restricted to v6". Since
   `icmp6` is the only such keyword, the only valid list is
   `[icmp6]` — semantically identical to the `==` form via v0.1's
   `in [N] ≡ == N` rule. The "..." denotes only repetition (which
   v0.2 dedupes silently per the geoip-style rule).

2. **Loose reading.** `[icmp6, ...]` is shorthand for "any
   in-list that contains `icmp6`". Lists like `[icmp6, tcp]`,
   `[icmp6, udp]`, `[tcp, icmp6, udp]` all match the form and
   establish the v6-family guard.

The disjunction rule at `:630` makes the loose reading
contradictory:

> A disjunction `cond_a or cond_b` establishes a guard for the
> then-branch only when **every** disjunct independently
> establishes the guard (the v0.1 union-of-branches rule).

By v0.1's `in [N1, N2] ≡ ==N1 or ==N2` rule (`FWL_V01_SPEC.md`
operators table), `pkt.proto in [icmp6, tcp]` is a disjunction:
`pkt.proto == icmp6 or pkt.proto == tcp`. The `tcp` arm does
**not** establish the v6 family (per the proto-enum table at
`:160-165`, `tcp` is allowed on both v4 and v6 in v6-active
programs). Per the disjunction rule, the in-list does NOT
establish v6.

So:
- Loose reading of `:622-624` ⇒ `pkt.proto in [icmp6, tcp]`
  establishes v6.
- Disjunction rule at `:630` ⇒ `pkt.proto in [icmp6, tcp]` does
  NOT establish v6.

The same source satisfies one rule and violates the other. The
spec does not say which rule wins.

Class B (spec internal contradiction) with a downstream Class A
surface: the dominator analyser will diverge across implementations
on every Tier 2 program that reads a v6-only field inside a body
guarded by `pkt.proto in [icmp6, <v4-or-cross-family-keyword>]`.

## The two clauses

### `:622-624` — family-restricted-keyword guard

> A condition that uses a *family-restricted* proto keyword on a
> `pkt.proto` comparison — `pkt.proto == icmp6` or
> `pkt.proto in [icmp6, ...]` (icmp6 is the only family-restricted
> keyword in v0.2; it matches IPv6 only). Reaching such a
> comparison's then-branch implies (a) the `pkt.proto` read
> succeeded — i.e. the L3 parse for the inferred family succeeded
> — and (b) the family is the keyword's family.

The text is positive ("such a comparison's then-branch implies …
the family is the keyword's family") and the `[icmp6, ...]`
notation is undefined.

### `:630` — disjunction rule (polarity- and union-aware)

> A disjunction `cond_a or cond_b` establishes a guard for the
> then-branch only when **every** disjunct independently
> establishes the guard (the v0.1 union-of-branches rule).
> `pkt.src_ip6 == ::1 or pkt.proto == tcp` does not establish the
> v6 guard, because the right disjunct can be true on a v4 TCP
> packet without the v6 parse succeeding. `pkt.proto == tcp or
> pkt.proto == udp` does establish the L4-readable guard, because
> both disjuncts independently provide one.

The `or` form is explicit. By v0.1's `in [...]`-as-disjunction
rule, `pkt.proto in [icmp6, tcp]` reduces to `pkt.proto == icmp6
or pkt.proto == tcp`, which the disjunction rule judges
non-establishing for v6.

### v0.1 affirms the `in`-as-disjunction rewrite

`FWL_V01_SPEC.md` Operators section establishes that `pkt.<field>
in [v1, v2, v3]` is the same predicate as `pkt.<field> == v1 or
pkt.<field> == v2 or pkt.<field> == v3`. v0.2 inherits this rule
verbatim (the `in` operator is unchanged per the surface deltas
table at `:42`).

So the two readings of `[icmp6, ...]` translate, after the v0.1
rewrite, to:

| Source | Disjunction form | `:622` says | `:630` says |
|---|---|---|---|
| `pkt.proto == icmp6` | `pkt.proto == icmp6` | establishes v6 | establishes v6 |
| `pkt.proto in [icmp6]` | `pkt.proto == icmp6` | establishes v6 | establishes v6 |
| `pkt.proto in [icmp6, icmp6]` | `pkt.proto == icmp6` (after dedup) | establishes v6 | establishes v6 |
| `pkt.proto in [icmp6, tcp]` | `pkt.proto == icmp6 or pkt.proto == tcp` | **establishes v6** (loose reading of `:622`) | **does not establish v6** (`:630` rule) |
| `pkt.proto in [icmp6, udp, tcp]` | `... or == udp or == tcp` | **establishes v6** (loose) | **does not establish v6** (`:630`) |

The first three rows are unambiguous. The last two depend on
which rule the analyser consults.

## Concrete program the spec is contradictory on

```python
@xdp(eth0)
def firewall(pkt):
  if pkt.proto in [icmp6, tcp]:
    addr = pkt.src_ip6        # statement-position v6 read
    if addr == ::1:
      drop
  allow
```

The in-list contains `icmp6` (a v6-only keyword) and `tcp` (a
cross-family keyword). On reaching the body:

- *Loose `:622` reading:* the body is dominated by a v6-establishing
  guard. `addr = pkt.src_ip6` is well-formed. The program compiles.
- *Strict `:630` (disjunction) reading:* the body is dominated by an
  L3-readable guard (both disjuncts give that) but NOT by a
  v6-family guard (the `tcp` disjunct does not give v6). `addr =
  pkt.src_ip6` is a bare-field statement-position read on a path
  not guarded by an IPv6-establishing condition (`:735`). The
  program is rejected with `error: 'pkt.src_ip6' read on a path
  not guarded by an IPv6-establishing condition`.

The same source compiles on one reasonable analyser and is
rejected on another.

The runtime behaviour under the loose reading is also bad: on a
v4 TCP packet, the body executes; `pkt.src_ip6` reads garbage
bytes from the v4 IP header / Ethernet payload at the v6-source-
address offset. This is exactly the failure mode the dominator
rule was introduced to prevent.

## Why this matters

Three angles:

1. **The spec has two rules that disagree on the same source.**
   Internal contradiction. An implementor cannot satisfy both;
   they have to pick.

2. **The contradiction lands on the dominator analysis itself.**
   The whole point of formalizing the dominator rule (per the
   companion `…-tier2-dominator-rule-negation-and-else-not-
   addressed` finding's resolution) was to make the analyser's
   accept/reject set crisp and polarity-aware. The
   `pkt.proto in [icmp6, ...]` clause re-introduces an underspecified
   surface that defeats the polarity-aware machinery for the
   one keyword the v6 guard most depends on.

3. **The `.pkt` corpus has no path to lock either reading.**
   `compiles: true` matches the loose reading; `compiles: false`
   with the dominator-error pattern matches the strict reading.
   Hone's three-oracle check (interpreter, BPF runtime, spec
   review) cannot pick a sentinel without the spec resolving.

The same hole replicates for any future v6-only proto keyword
v0.3 may add (e.g. a hypothetical `v6frag` for the Fragment
extension header). The wording style in `:622-624` will have the
same ambiguity for the same in-list-mixing reason.

## Three resolutions

| Resolution | Effect |
|---|---|
| (a) Strict reading, made explicit. | Edit `:622-624` to remove the `[icmp6, ...]` shorthand and replace it with a precise rule: "A condition `pkt.proto == icmp6` (or its rewrite-equivalent `pkt.proto in [icmp6]`, modulo silent dedup) is the only `pkt.proto` form that establishes the v6 family guard. An in-list that contains `icmp6` together with any cross-family proto keyword does NOT establish the v6 family guard, by the disjunction rule at `:630`." Programs writing `if pkt.proto in [icmp6, tcp]:` followed by a bare v6 read are rejected. |
| (b) Loose reading, with a runtime patch. | Keep the loose `[icmp6, ...]` clause but require the emitter to insert an EtherType check before the bare v6 read inside the body. The body becomes "if EtherType is v6: read; else: skip the read and treat any subsequent v6-typed comparison as false." This is a runtime fall-through, not a compile-time guard, and it changes the semantics of the dominator rule from "compile-time correctness" to "runtime safety net". Operationally messier. |
| (c) Reject the in-list form entirely when it mixes families. | New compile-error row: "An `in`-list containing `icmp6` together with any cross-family proto keyword (`tcp`, `udp`, `icmp`) is rejected; rewrite as `pkt.proto == icmp6 or pkt.proto == <other>` to make the disjunction explicit, or use a single-element list." This forces the user to spell out the disjunction so the disjunction rule applies unambiguously. |

(a) is the cleanest resolution and aligns with the disjunction
rule already in the spec. (c) is the most explicit user-facing
fix — it disallows the ambiguous form entirely. (b) is the
operator-friendliest but changes the dominator rule's character.

## Proposed fix (option a)

Replace `FWL_V02_SPEC.md:622-624` with:

> A condition that uses a *family-restricted* proto keyword on a
> `pkt.proto` comparison — `pkt.proto == icmp6` (icmp6 is the
> only family-restricted keyword in v0.2; it matches IPv6 only).
> Reaching such a comparison's then-branch implies (a) the
> `pkt.proto` read succeeded — i.e. the L3 parse for the inferred
> family succeeded — and (b) the family is the keyword's family.
>
> The same guard establishment applies to the `in`-list form
> `pkt.proto in [icmp6]` (single-element list, semantically
> equivalent to `==` per v0.1's `in [N] ≡ == N` rule). An
> `in`-list of two or more proto keywords is governed by the
> disjunction rule at `:630`: every keyword in the list must
> independently establish the guard. So `pkt.proto in [icmp6,
> icmp6]` (after dedup) establishes v6; `pkt.proto in [icmp6,
> tcp]` does NOT establish v6 (the `tcp` arm doesn't); `pkt.proto
> in [icmp6, udp]` does NOT establish v6.

Add a corresponding Edge case under "IPv6 Fields" or under the
Tier 2 Edge-cases section:

> *In-list mixing icmp6 with cross-family keywords.* `pkt.proto
> in [icmp6, tcp]` matches v6 ICMPv6 frames *and* v4-or-v6 TCP
> frames. Reaching the then-branch does not establish the v6
> family (per the disjunction rule). A bare v6 read inside the
> body is rejected: `error: 'pkt.src_ip6' read on a path not
> guarded by an IPv6-establishing condition`. To get both v6
> matching and a v6 guard, write the conditions separately:
>
> ```python
> if pkt.proto == icmp6:
>   addr = pkt.src_ip6     # OK: dominator established
>   ...
> if pkt.proto == tcp:
>   port = pkt.dst_port    # OK: L4 dominator established
>   ...
> ```

## Suggested corpus item

```yaml
# corpus/from_hunt/tier2-icmp6-in-list-with-tcp-no-v6-guard.pkt
program: |
  @xdp(eth0)
  def firewall(pkt):
    if pkt.proto in [icmp6, tcp]:
      addr = pkt.src_ip6
      if addr == ::1:
        drop
    allow

description: |
  Loose reading of :622-624: compiles (in-list contains icmp6, so
  the body is "v6-guarded"); on a v4 TCP packet, the bare
  `pkt.src_ip6` read produces garbage bytes.
  Strict reading per :630 disjunction rule: compile error
  ("pkt.src_ip6 read on a path not guarded by an IPv6-establishing
  condition"), because the `tcp` arm does not establish v6.

expected:
  compiles: false
  compile_error_pattern: "pkt.src_ip6.*not guarded by an IPv6-establishing condition"
```

A second discriminator that exercises the same hole at runtime
(in case the analyser picks the loose reading and lets the
program through):

```yaml
program: |
  @xdp(eth0)
  def firewall(pkt):
    if pkt.proto in [icmp6, tcp]:
      addr = pkt.src_ip6
      if addr == ::1:
        drop
    allow

# Reading 1 (loose, accepts at compile time): on a v4 TCP packet
# the body executes; pkt.src_ip6 reads garbage; the addr == ::1
# comparison is undefined; the interpreter and BPF emitter will
# pick different sentinels.
packets:
  - tcp(src_ip="10.0.0.1", dst_ip="10.0.0.2", dport=80)

# The interpreter likely returns false for the addr-against-::1
# comparison (no v6 read on a v4 frame) ⇒ XDP_PASS.
# The BPF emitter under the loose reading emits a 16-byte read
# from the v6-source-address offset (which lands inside the v4
# IP header) ⇒ undefined byte pattern ⇒ comparison undefined
# ⇒ may XDP_DROP or XDP_PASS depending on what the bytes happen
# to be.
expected:
  - {action: pass}
```

## Class

Class B (spec internal contradiction between `:622-624` and
`:630`) with a downstream Class A surface. Two analysers landing
on different readings of `[icmp6, ...]` produce different
accept/reject sets and, on the accept path, different runtime
behaviour on cross-family-mixed in-lists.
