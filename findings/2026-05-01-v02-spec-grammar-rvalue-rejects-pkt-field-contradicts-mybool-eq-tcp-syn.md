---
id: finding/2026-05-01-v02-spec-grammar-rvalue-rejects-pkt-field-contradicts-mybool-eq-tcp-syn
type: finding
protocol: [tcp]
builtins: []
severity: high
layer: spec
pattern_tags: [spec-inconsistency, grammar, tier2, locals, example-vs-rule]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: [finding/2026-05-01-v02-spec-grammar-value-field-missing-proto-and-tcp-fields, finding/2026-05-01-v02-spec-tier2-locals-not-in-condition-grammar]
---

# 2026-05-01-v02-spec-grammar-rvalue-rejects-pkt-field-contradicts-mybool-eq-tcp-syn

## Summary

The v0.2 grammar's `rvalue` production
(`FWL_V02_SPEC.md:1097-1098`) admits only `operand | identifier`. It
does **not** admit `value_field`. So a comparison whose right-hand
side is a packet field — for example, `my_bool == pkt.tcp.syn` — does
not parse.

But the Tier 2 "Boolean in non-bool context" edge case at
`FWL_V02_SPEC.md:684` explicitly presents that exact form as a valid
program:

> `my_bool == pkt.tcp.syn` — fine (both `bool`).

The spec contradicts itself: prose says fine, grammar says
syntax error.

Class B (analyzer/spec contradiction; spec layer).

## Locations

### Grammar — `rvalue` excludes `value_field`

`FWL_V02_SPEC.md:1089-1098`:

```ebnf
comparison      = lvalue comp_op rvalue
                | lvalue "in" set_or_range ;

comp_op         = "==" | "!=" | "<" | ">" | "<=" | ">=" ;

lvalue          = value_field
                | identifier ;

rvalue          = operand
                | identifier ;
```

`operand` is enumerated at `FWL_V02_SPEC.md:1131`:

```ebnf
operand       = integer | ipv4 | ipv6 | proto_keyword ;
```

`pkt.tcp.syn` is a `value_field`
(`FWL_V02_SPEC.md:1117-1121`), not a literal and not an
`identifier`. The only production where it appears is `value_field`,
which is admitted by `lvalue` but **not** by `rvalue`.

So a `comparison` cannot have a `value_field` on the right.
Concretely, the parser must reject every one of:

- `my_bool == pkt.tcp.syn`
- `my_bool == pkt.tcp.ack`
- `my_bool != pkt.tcp.syn`
- `my_local == pkt.proto`        (locals comparing to a proto enum)
- `my_local == pkt.dst_port`     (locals comparing to a port)
- `my_local == pkt.src_ip`       (locals comparing to an address)
- `my_local == pkt.src_ip6`      (same, v6)
- `pkt.dst_port == pkt.src_port` (two packet fields — both tied as
                                  rvalue cannot be a `value_field`)

### Edge case prose — claims `my_bool == pkt.tcp.syn` is "fine"

`FWL_V02_SPEC.md:684`:

> *Boolean in non-bool context.* Locals of type `bool` are not
> numerically convertible. Inside a Tier 2 `if` condition, a bare
> bool local is a valid `primary` (`if my_bool:`). Inside a
> comparison, both sides of `==`/`!=` may be a `bool` local or a
> `bool` packet field, but mixing types is a type error:
> `pkt.dst_port == my_bool` — type error (`u16` vs `bool`);
> `my_bool == pkt.tcp.syn` — fine (both `bool`).

The first half of that sentence (`pkt.dst_port == my_bool` is a type
error) parses fine — `pkt.dst_port` is a `value_field` (lvalue) and
`my_bool` is an `identifier` (admitted by `rvalue`). The analyser
catches the type mismatch.

The second half (`my_bool == pkt.tcp.syn`) does **not** parse —
`my_bool` is `lvalue=identifier`, comp_op is `==`, and the rvalue
slot demands an `operand` or `identifier` but is given a
`value_field`. The parser rejects the whole comparison before the
analyser ever runs the type rule.

So the spec promises a "fine" outcome that the grammar makes
unreachable.

### The earlier sibling finding fixed only one half

`finding/2026-05-01-v02-spec-grammar-value-field-missing-proto-and-tcp-fields`
(status: fixed) extended `value_field` to include `pkt.proto`,
`pkt.tcp.syn`, `pkt.tcp.ack`. That fix admits the LHS form
(`syn = pkt.tcp.syn`, `pkt.tcp.syn == 1`) but does not admit the
RHS form (`local == pkt.tcp.syn`). The finding's scope was the
`value_field` enumeration, not the `lvalue`/`rvalue` split.

The current grammar puts `value_field` on the lvalue side only.
The "fine (both `bool`)" example needs it on the rvalue side too.

## Why this matters

The Tier 2 type rules carve out a deliberate class of programs that
"compute a bool, store it, and compare it later":

```python
def firewall(pkt):
  if pkt.proto == tcp:
    is_handshake = pkt.tcp.syn
    if is_handshake == pkt.tcp.syn:   # parse error per current grammar
      log
    if pkt.tcp.syn == is_handshake:   # also rejected — RHS still a value_field-or-identifier mismatch
      log                              # (lvalue value_field OK; rvalue identifier OK; this one parses)
  drop
```

Wait — the second comparison flips operands, putting `pkt.tcp.syn`
on the LHS and the local on the RHS. That **does** parse: `lvalue =
value_field` and `rvalue = identifier`. So users can rephrase, but
the spec promises both directions work.

The asymmetry (`field op local` parses, `local op field` does not)
is invisible in the spec prose and surprising to users used to
Python's commutative comparisons. It's also the kind of asymmetry a
naive corpus author will not exercise — the dogfood example at
`FWL_V02_SPEC.md:953-986` keeps every comparison in `field op
literal` form, never `local op field`.

If the analyser later relies on the spec's "both sides may be a
bool packet field" claim to validate type rules — for example, if it
type-checks comparisons by walking the AST and checking
`type(lhs) == type(rhs)` without first checking that both sides
**parsed** — then a corpus case derived from the edge-case prose
will hit a parse error before the analyser ever runs.

## Three possible resolutions

| Resolution | What changes |
|---|---|
| (a) Extend `rvalue` to admit `value_field` | Replace `rvalue = operand \| identifier` with `rvalue = operand \| identifier \| value_field`. Symmetric with `lvalue`. Comparison becomes commutative w.r.t. parsing. |
| (b) Reject the surface | Drop the `my_bool == pkt.tcp.syn` example from the edge-case prose. Make the spec's narrow "comparisons read fields on the LHS only" rule explicit (a Compile-error row covering `local op pkt.<field>`). |
| (c) Add a separate production | Introduce `bool_local_eq_field = identifier ("==" \| "!=") ("pkt.tcp.syn" \| "pkt.tcp.ack")` so only the bool-bool case is permitted on the RHS. Extra grammar weight for a narrow surface. |

## Proposed fix

Pick (a). It mirrors the asymmetry-fix that the earlier
`value_field` finding applied to LHS reads. Concretely, replace
`FWL_V02_SPEC.md:1097-1098`:

```ebnf
rvalue          = operand
                | identifier ;
```

with:

```ebnf
rvalue          = operand
                | identifier
                | value_field ;
```

The analyser's existing type-checker is already responsible for
rejecting `pkt.dst_port == my_bool` (u16 vs bool) on type grounds
— extending the parser to admit the form does not change the
analyser's behaviour on already-rejected programs.

Optionally add a clarifying comment alongside `rvalue`:

```
(* `rvalue` admits `value_field` so that a comparison can read a
   packet field on either side. The analyser still enforces type
   compatibility (a `u16` field on one side and a `bool` local on
   the other is a type error). *)
```

## Class

Class B (spec/grammar contradiction). Spec-layer.
