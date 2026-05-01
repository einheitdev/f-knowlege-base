---
id: finding/2026-05-01-v02-spec-v6-activation-rule-only-covers-equality-not-inequality-or-in-or-locals
type: finding
protocol: [icmp6, tcp, udp]
builtins: [geoip]
severity: high
layer: spec
pattern_tags: [spec-incompleteness, ipv6, activation-rule, cross-family, pkt-proto, tier2, locals, regression-hazard]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: [finding/2026-05-01-v02-spec-strict-superset-vs-cross-family-pkt-proto, finding/2026-05-01-v02-spec-proto-equality-table-vs-opaque-byte-equality]
---

# 2026-05-01-v02-spec-v6-activation-rule-only-covers-equality-not-inequality-or-in-or-locals

## Summary

The v0.2 strict-superset claim is preserved by the v6 parse-path
*activation* rule at `FWL_V02_SPEC.md:912`, which lists exactly
four surfaces that activate the v6 path:

> For a program **touching IPv6 fields** (`pkt.src_ip6`,
> `pkt.dst_ip6`, `pkt.proto == icmp6`, or a `geoip(...)` whose LHS
> has type `ipv6`):

The rule mentions only `pkt.proto == icmp6` (equality with the
`icmp6` proto keyword). It does not cover three other v0.2
surfaces that also reference the `icmp6` keyword or that
otherwise reach v6 frames:

1. **Inequality.** `pkt.proto != icmp6` — the negation form of
   the same comparison.
2. **`in` membership over a proto-keyword list.** The grammar at
   `:1175` admits `list = "[" operand { "," operand } "]"` and
   `operand = … | proto_keyword` (`:1165`), so
   `pkt.proto in [tcp, icmp6]` is a grammar-legal v0.2 form. The
   `icmp6` keyword appears, but not via `==`.
3. **Tier 2 hoist.** `p = pkt.proto; if p == icmp6: …` (or the
   inequality form). The local `p` is `proto`-typed and the
   keyword `icmp6` appears in the comparison; per
   `finding/…-proto-equality-table-vs-opaque-byte-equality` (fixed
   to byte-equality, option a), this is supposed to be exactly
   equivalent to the inline `pkt.proto == icmp6` form. But the
   activation rule on `:912` keys off the *textual* surface, not
   the proto-keyword's appearance in any comparison position.

For each of (1)–(3), the activation rule as written does **not**
fire — the program's only mention of a v6 surface is the proto
keyword `icmp6` in a non-`==` position, none of which the rule
enumerates. The v6 parse path is therefore not generated. Yet the
program clearly *intends* v6 matching: the user wrote `icmp6`
because they want to act on ICMPv6 packets.

Class B (spec incompleteness with downstream cross-oracle drift —
behaves as Class A under any implementation that follows the
activation rule literally).

## The four surfaces, side by side

```python
@xdp(eth0)

# Case 1 — equality, currently activates v6 path per :912.
drop if pkt.proto == icmp6
default allow
```

```python
@xdp(eth0)

# Case 2 — inequality. NOT mentioned in the activation rule.
drop if pkt.proto != icmp6
default allow
```

```python
@xdp(eth0)

# Case 3 — list membership. NOT mentioned in the activation rule.
drop if pkt.proto in [tcp, icmp6]
default allow
```

```python
@xdp(eth0)

# Case 4 — Tier 2 hoist. NOT mentioned in the activation rule.
def firewall(pkt):
  if pkt.src_ip in [0.0.0.0/0]:    # v4-establishing guard for stmt-level read
    p = pkt.proto
    if p == icmp6:
      drop
  allow
```

(Case 4 also reads `pkt.src_ip` for the dominator rule — that
guard is v4-only and does not by itself touch the v6 surface in
the spec's sense; the only v6-shaped reference is `icmp6`, which
appears in `p == icmp6`.)

| Case | v0.2 activation rule fires? | Operator's clear intent | Spec runtime |
|---|---|---|---|
| 1: `pkt.proto == icmp6` | yes | match v6 ICMPv6 frames | matches v6 ICMPv6 |
| 2: `pkt.proto != icmp6` | **no** | match every non-ICMPv6 frame, including v6 frames | v6 parse path inactive ⇒ v6 frames fall through every rule, never reach the comparison; the rule effectively becomes "drop every non-ICMP-byte v4 frame" |
| 3: `pkt.proto in [tcp, icmp6]` | **no** | match v4 TCP, v6 TCP, and v6 ICMPv6 | v6 parse inactive ⇒ matches v4 TCP only |
| 4: `p = pkt.proto; if p == icmp6` | **no** | match v6 ICMPv6 frames, exactly as Case 1 | v6 parse inactive ⇒ never matches |

Cases 2, 3, and 4 silently lose their v6 semantics. The user typed
the `icmp6` keyword in a position the spec does not register as
"touching v6", so the emitter generates a v4-only program.

## Why this matters

Three concrete failure modes:

1. **The activation rule is keyed off textual surface, not
   semantics.** A user-visible refactor — `pkt.proto == icmp6`
   ⇒ `not (pkt.proto != icmp6)` — silently changes program
   semantics. Same story for `pkt.proto == icmp6` ⇒ `pkt.proto in
   [icmp6]`. Both refactors should be no-ops, but the activation
   rule observes only the textual `pkt.proto == icmp6` form.

2. **The Tier 2 hoist invariant is broken at the activation
   layer, not just the equality layer.** The companion finding
   (`…-proto-equality-table-vs-opaque-byte-equality`) made the
   resolution that `p = pkt.proto; if p == kw:` is byte-equivalent
   to `if pkt.proto == kw:`. The fix to that finding presumes the
   v6 path is *active* in both forms. But under the current
   activation rule, only the inline form activates v6 — the
   hoisted form does not. So the resolution to the companion
   finding doesn't actually deliver the promised equivalence.

3. **Cross-oracle drift on a non-trivial fraction of v0.2
   programs.** Any program whose only v6 surface is `icmp6`-via-
   `!=`/`in`/local-eq will compile to two different machines
   under two reasonable readings of the spec:
   - **Reading X (literal activation rule).** The textual form
     doesn't match `:912`'s enumeration, so the v6 path is
     inactive, and the proto comparison is v4-only.
   - **Reading Y (semantic activation: any reference to `icmp6`,
     `pkt.src_ip6`, `pkt.dst_ip6`, or v6-typed `geoip`
     activates).** The proto comparison is v4+v6, matching what
     a user with the proto-enum table on screen would expect.

   The interpreter (likely written from the proto-enum table)
   will produce Reading Y; the emitter (whose activation gating
   is the cheap textual scan) will produce Reading X. Hone's
   interpreter-vs-BPF oracle disagrees on every Case 2/3/4
   program.

## Concrete `.pkt` evidence

```yaml
# corpus/from_hunt/v6-activation-from-inequality.pkt
program: |
  @xdp(eth0)
  drop if pkt.proto != icmp6
  default allow

description: |
  An IPv6 TCP packet (next_header == 6).
  Reading X (textual activation): v6 path not generated; v6
    packet falls through every rule; default allow ⇒ XDP_PASS.
  Reading Y (semantic activation): v6 path generated;
    pkt.proto != icmp6 evaluates true on v6 TCP (read succeeds
    [v6 path active], byte=6 differs from 58); rule fires ⇒
    XDP_DROP.

packets:
  - tcp6(src_ip="2001:db8::1", dst_ip="2001:db8::2", dport=80)
expected:
  # Operator's natural intent: drop everything that isn't ICMPv6.
  - {action: drop}
```

```yaml
# corpus/from_hunt/v6-activation-from-list.pkt
program: |
  @xdp(eth0)
  drop if pkt.proto in [tcp, icmp6]
  default allow

description: |
  An IPv6 TCP packet (next_header == 6).
  Reading X: v6 path not generated; v6 packet falls through;
    default allow ⇒ XDP_PASS.
  Reading Y: v6 path active; pkt.proto in [tcp, icmp6] true on
    v6 TCP (next_header == 6 == tcp.byte); rule fires ⇒
    XDP_DROP.

packets:
  - tcp6(dport=80)
expected:
  - {action: drop}
```

```yaml
# corpus/from_hunt/v6-activation-from-proto-local-icmp6.pkt
program: |
  @xdp(eth0)
  def firewall(pkt):
    if pkt.src_ip in [0.0.0.0/0]:    # v4-establishing
      p = pkt.proto
      if p == icmp6:
        drop
    allow

description: |
  An IPv6 ICMPv6 packet (next_header == 58).
  Reading X: v6 path not generated; pkt.src_ip in [0.0.0.0/0]
    is false on v6 (EtherType mismatch); body skipped; final
    allow runs ⇒ XDP_PASS.
  Reading Y: v6 path active; same reasoning, BUT the operator's
    intent (per the companion fixed finding) was that hoisting
    pkt.proto into a local doesn't change behaviour. The inline
    `if pkt.proto == icmp6: drop` does activate v6 and would
    fire on the v6 ICMPv6 packet. The hoisted form is supposed
    to be equivalent.

packets:
  - icmp6(src_ip="2001:db8::1", dst_ip="2001:db8::2")
expected:
  # Operator's expectation given the companion finding's fix:
  # "hoisting pkt.proto into a local doesn't change behaviour"
  - {action: drop}
```

## Three resolutions

| Resolution | Effect |
|---|---|
| (a) Expand the activation rule semantically. | Rewrite `:912` so the activation triggers on *any* expression that mentions an IPv6 surface: `pkt.src_ip6`, `pkt.dst_ip6`, the `icmp6` proto keyword (anywhere in the program — not just inside `pkt.proto == icmp6`), an IPv6 literal, an IPv6 CIDR, or a `geoip(...)` call whose LHS has type `ipv6`. The analyser already needs to enumerate every `proto_keyword` token and every literal kind for type-checking; this is a one-line addition. |
| (b) Restrict the surfaces that can mention `icmp6`. | Reject `pkt.proto != icmp6`, `pkt.proto in [icmp6, …]`, and `p == icmp6` (where `p` is a local) at the analyser. Force users to write `pkt.proto == icmp6` explicitly. This is intrusive — Tier 2 specifically allows the proto-local idiom per `:559` and the companion finding's resolution. |
| (c) Add an explicit Compile-error row for "v6 keyword in non-activating position". | `error: 'icmp6' appears in <position>; v0.2 activates the v6 parse path only on '==' against pkt.proto, on pkt.src_ip6/pkt.dst_ip6, or on a v6-typed geoip RHS. Either rewrite as 'pkt.proto == icmp6' or also reference a v6 IP field`. Documents the gap but pushes the burden to the user. |

(a) is the cleanest and matches what the user's mental model
already is. (b) contradicts the resolution of the companion
finding and breaks Tier 2 ergonomics. (c) is the minimum that
makes the spec self-consistent.

## Proposed fix (option a)

Replace `FWL_V02_SPEC.md:912` with:

> For a program **touching IPv6 surfaces** — defined as the
> source mentioning **any** of the following: `pkt.src_ip6`,
> `pkt.dst_ip6`, an IPv6 literal, an IPv6 CIDR, the `icmp6` proto
> keyword (in any position — `==`, `!=`, `in`, or against a
> `proto`-typed local), or a `geoip(...)` call whose host
> comparison's LHS has type `ipv6` — the analyser activates the
> IPv6 parse path in the emitter.

Update the negative case at `:918` symmetrically:

> For a program **touching no IPv6 surface** (no occurrence of
> any of the surfaces enumerated above), the v6 parse path is
> not generated. Frames with EtherType `0x86DD` (IPv6) fall
> through every rule, exactly as in v0.1.

Add the three Edge cases under "IPv6 Fields":

> *`pkt.proto != icmp6` activates the v6 parse path.* The
> `icmp6` keyword references an IPv6-only proto family; whether
> the comparison is `==` or `!=`, the analyser must generate
> the v6 parse to ever evaluate the comparison on v6 frames.

> *`pkt.proto in [tcp, icmp6]` activates the v6 parse path.*
> Likewise — `icmp6` in an `in`-list is the same surface
> reference.

> *Tier 2 hoist of `pkt.proto` keeps v6 active.* If a
> Tier 2 function compares a `proto`-typed local against
> `icmp6` (e.g. `if p == icmp6:` or `if p != icmp6:`), the v6
> parse path is active. This preserves the refactor invariance
> guarantee at `:170`.

Add the corpus cases above to `corpus/from_hunt/<date>/`.

## Class

Class B at the spec layer (incompleteness in the activation
rule), with downstream Class A behaviour: any implementor that
follows the rule literally produces a v4-only emitter for
programs whose only v6 surface is `icmp6`-via-`!=`/`in`/local-eq;
any implementor that activates v6 on every occurrence of `icmp6`
produces a v4+v6 emitter. Hone's interpreter-vs-BPF oracle
disagrees with no spec text to point at.
