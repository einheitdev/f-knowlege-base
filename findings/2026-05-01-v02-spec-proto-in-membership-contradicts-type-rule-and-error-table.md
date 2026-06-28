---
id: finding/2026-05-01-v02-spec-proto-in-membership-contradicts-type-rule-and-error-table
type: finding
protocol: [tcp, udp, icmp, icmp6]
builtins: []
severity: high
layer: spec
pattern_tags: [spec-inconsistency, tier2, type-rules, proto, in-operator, activation-rule, error-table]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: [finding/2026-05-01-v02-spec-v6-activation-rule-only-covers-equality-not-inequality-or-in-or-locals, finding/2026-05-01-v02-spec-tier2-proto-local-vs-local-comparison-disallowed-but-no-error]
---

# 2026-05-01-v02-spec-proto-in-membership-contradicts-type-rule-and-error-table

## Summary

Three different sections of `FWL_V02_SPEC.md` disagree about whether
`pkt.proto in [tcp, icmp6]` (and any other `<proto-typed-LHS> in
<list-of-proto-keywords>` form) is a valid v0.2 program:

| Place | What it says | Implication |
|---|---|---|
| `:566` (Tier 2 type rules — `proto` paragraph) | "Arithmetic, ordered comparisons (`<`, `<=`, `>`, `>=`), `in`, and comparisons against any non-`proto`-typed value are all type errors." | `pkt.proto in [tcp, icmp6]` is a **type error**. |
| `:718` (Tier 2 compile-errors table) | `\| `proto`-typed value used with `<`, `<=`, `>`, `>=`, `in`, or arithmetic \| error: 'proto' values support only equality (==, !=); got '<op>' \|` | Same: a hard compile error, with a specific message. |
| `:918` (v6 activation rule, "touches an IPv6 surface" list) | "The `icmp6` proto keyword — in **any** position: `pkt.proto == icmp6`, `pkt.proto != icmp6`, **`pkt.proto in [tcp, icmp6]`**, `if p == icmp6:` against a Tier 2 `proto` local, etc." | Lists `pkt.proto in [tcp, icmp6]` as a real, compile-able program form whose presence activates the v6 parse path. |
| `:921` (semantic-activation paragraph) | "Refactors that preserve the program's semantics (e.g. … expanding a list-membership `in` into an `or`-chain) preserve activation." | Treats `pkt.proto in [tcp, icmp6]` as semantically equivalent to `pkt.proto == tcp or pkt.proto == icmp6`, i.e. as a permitted form. |

The grammar at `:1126-1149` permits the form (`comparison =
lvalue "in" set_or_range`; `set_or_range` admits `list`; `list`
admits `proto_keyword` operands; `lvalue` admits `value_field` which
includes `pkt.proto`).

So:

- **Parser:** accepts `pkt.proto in [tcp, icmp6]`.
- **Type rule (`:566`) + compile-errors table (`:718`):** rejects it
  with `error: 'proto' values support only equality (==, !=); got
  'in'`.
- **Activation rule (`:918`):** assumes it compiles, and activates
  the v6 parse path on its presence.

These three positions cannot all be right. Either the form compiles
(in which case `:566` and `:718` are wrong to ban it), or it does
not compile (in which case `:918`'s example list is wrong to cite it
and `:921`'s "expanding a list-membership `in` into an or-chain"
refactor example is dead code — there is no surface to refactor
*from*).

Class A precursor (cross-oracle drift): two implementers reading
this spec will land on different decisions for the same source. One
follows `:566`/`:718` and emits the compile error; the other
follows `:918` and emits a working v6-active BPF program.

The companion finding
`finding/2026-05-01-v02-spec-v6-activation-rule-only-covers-equality-not-inequality-or-in-or-locals`
fixed the activation rule (`:918`) to enumerate `pkt.proto in [...,
icmp6, ...]`. That fix is correct *only if* `:566`/`:718` are also
relaxed. They were not — the `in` ban survived the activation-rule
fix unchanged. This finding catches the survivor.

## Locations

### `:566` — Tier 2 type rules ban `in` on proto

> The `proto` type is opaque. Locals of type `proto` may participate
> in equality and inequality comparisons (`==`, `!=`) against any
> other `proto`-typed value — a `proto_keyword` token (`tcp`, `udp`,
> `icmp`, `icmp6`), the `pkt.proto` field, or another
> `proto`-typed local. Arithmetic, ordered comparisons (`<`, `<=`,
> `>`, `>=`), **`in`**, and comparisons against any non-`proto`-typed
> value are all type errors.

The "all type errors" clause covers `pkt.proto in [tcp]` because
`pkt.proto`'s type is `proto` (per the `pkt.proto` row in the type
table at `:562`). The text does not exempt the `pkt.proto` field
from the rule it states for "Locals of type `proto`" — and even if
it did, two `proto`-typed values (a local on the LHS and a
`proto_keyword` on the RHS-of-`in`-list) would still trigger the
`in` ban.

### `:718` — compile-errors table makes the ban concrete

| Condition | Error |
|---|---|
| `proto`-typed value used with `<`, `<=`, `>`, `>=`, **`in`**, or arithmetic | `error: 'proto' values support only equality (==, !=); got '<op>'` |

This row does not say "Locals of type `proto`" — it says
"`proto`-typed value", which the type table includes both `pkt.proto`
and any `proto`-typed local under.

### `:918` — activation rule cites the form as legal

> A program **touches an IPv6 surface** if its source mentions any of
> the following anywhere — in a condition, an assignment RHS, an
> `in`-list, or a comparison operand on either side:
> …
> - The `icmp6` proto keyword — in **any** position: `pkt.proto ==
>   icmp6`, `pkt.proto != icmp6`, **`pkt.proto in [tcp, icmp6]`**,
>   `if p == icmp6:` against a Tier 2 `proto` local, etc.

The bullet enumerates `pkt.proto in [tcp, icmp6]` alongside three
other forms that the spec elsewhere accepts (the two equality forms
and the proto-local hoist). It does **not** caveat the form as
"grammar-legal but rejected by the analyser" — it treats it as a
sibling of `pkt.proto == icmp6` for the purpose of activation.

### `:921` — refactor-invariance language presumes the form is real

> Refactors that preserve the program's semantics (e.g. hoisting
> `pkt.proto` into a Tier 2 local, swapping `==` for `!=` with
> negation, **expanding a list-membership `in` into an `or`-chain**)
> preserve activation.

The "list-membership `in`" being refactored into an or-chain has
no other v0.2 referent than `pkt.proto in [tcp, icmp6]` (the only
other proto-local-typed `in` form would be on a hoisted local, but
the type rule bans that too). If `:718` is taken literally, this
example refactor is between two non-existent-and-impossible-to-write
program states.

## Concrete examples that the spec leaves undefined

### Example 1 — Tier 1 list

```python
@xdp(eth0)

drop if pkt.proto in [tcp, icmp6]
default allow
```

- Per `:566` / `:718`: compile error (`error: 'proto' values support
  only equality (==, !=); got 'in'`).
- Per `:918`: compiles; activates the v6 parse path; the rule fires
  on v4 TCP, v6 TCP, and v6 ICMPv6 packets.

### Example 2 — Tier 2 hoist + list

```python
@xdp(eth0)

def firewall(pkt):
  p = pkt.proto              # p : proto
  if p in [tcp, icmp6]:      # type error per :566; OK per :918
    drop
  allow
```

The hoist makes the LHS a `proto`-typed *local* (the case `:566`'s
text most directly addresses), so the `in` ban is unambiguous —
yet the activation rule's "in **any** position" wording still picks
up the `icmp6` keyword from the list, suggesting v6 activation.

### Example 3 — single-element list (degenerate)

```python
@xdp(eth0)

drop if pkt.proto in [tcp]
default allow
```

The v0.1 spec at `:247` says `in [80]` and `== 80` are equivalent.
By that precedent a `proto in [tcp]` ought to be equivalent to
`proto == tcp`. But `:566` and `:718` ban `in` on proto values
unconditionally, even for the single-element list — contradicting
the v0.1 equivalence rule that the v0.2 spec inherits unchanged at
the operators section (`:874-887`).

## Why this matters

Beyond the obvious cross-oracle Class-A drift, the inconsistency
makes one of the spec's marquee design claims — "v0.2 is a strict
superset of v0.1" — quietly false on the `proto in [single]`
surface. v0.1 admits that form (it's the natural reading of the
v0.1 grammar plus the `in [N] ≡ == N` rule); v0.2's `:566`/`:718`
forbid it.

It also cracks the "refactors preserve activation" claim at `:921`:
the spec invites the user to refactor between forms (`==` ↔ `!=`,
inline ↔ list-`in`, inline ↔ hoisted-local) that the type rule
disallows.

## Proposed fix

The activation-rule fix already landed assumed proto-`in` is a real
form. The clean way to make the spec self-consistent is to
**permit `in` on proto values**, with the same type discipline as
`==` (every list element must be `proto`-typed):

1. **Edit `:566`** — strike `in` from the "all type errors" list and
   replace with positive language:

   > Arithmetic and ordered comparisons (`<`, `<=`, `>`, `>=`) on
   > `proto` values, and comparisons against any non-`proto`-typed
   > value, are type errors. The `in` operator is permitted when the
   > right-hand side is a list of `proto_keyword` tokens (`tcp`,
   > `udp`, `icmp`, `icmp6`); other right-hand-side shapes (CIDR,
   > range, `geoip(...)`) are type errors. `pkt.proto in [tcp]` is
   > equivalent to `pkt.proto == tcp` per the v0.1 `in [N] ≡ == N`
   > rule.

2. **Edit `:718`** — split the row:

   | Condition | Error |
   |---|---|
   | `proto`-typed value used with `<`, `<=`, `>`, `>=`, or arithmetic | `error: 'proto' values support only equality (==, !=) and 'in' over proto-keyword lists; got '<op>'` |
   | `proto`-typed LHS with `in` over a non-proto-keyword RHS (CIDR, range, geoip) | `error: 'proto' values may only appear with 'in' over a list of proto_keyword tokens; got <T>` |

3. Leave `:918` and `:921` unchanged — they already describe the
   intended behaviour.

Alternative fix (much more disruptive, not recommended): keep the
ban and remove `pkt.proto in [...]` from the activation rule's
example list and the refactor-invariance paragraph. This would
require also striking the v0.1 `in [N] ≡ == N` rule for the proto
case, which contradicts the strict-superset claim.

## Class

Class A precursor (cross-oracle drift between an implementer who
follows `:566`/`:718` and one who follows `:918`/`:921`) with a
spec-layer root cause. Spec disagrees with itself.

## Test idea

A `.pkt` case once an implementation lands:

```yaml
program: |
  @xdp(eth0)
  drop if pkt.proto in [tcp, icmp6]
  default allow
expected:
  compiles: true            # under the proposed fix
  v6_path_active: true
  cases:
    - packet: tcp(src_ip="1.2.3.4")
      action: drop
    - packet: tcp6(src_ip="2001:db8::1")
      action: drop
    - packet: icmp6(src_ip="2001:db8::1")
      action: drop
    - packet: udp(src_ip="1.2.3.4")
      action: allow         # default
```

A regression case for the strict-superset claim, on an implementation
that fixes `:566`/`:718` but leaves the v0.1 surface unchanged:

```yaml
program: |
  @xdp(eth0)
  drop if pkt.proto in [tcp]
  default allow
expected:
  compiles: true
  v6_path_active: false     # no IPv6 surface mentioned
  cases:
    - packet: tcp(src_ip="1.2.3.4")
      action: drop
    - packet: udp(src_ip="1.2.3.4")
      action: allow
    - packet: tcp6(src_ip="2001:db8::1")
      action: allow         # v6 parse inactive ⇒ falls through
```
