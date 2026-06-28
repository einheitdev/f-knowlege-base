---
id: finding/2026-05-01-v02-spec-tier2-scalar-expr-missing-proto-keyword
type: finding
protocol: [tcp, udp, icmp, icmp6]
builtins: []
severity: medium
layer: spec
pattern_tags: [spec-inconsistency, grammar, tier2, locals, proto-type]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: [finding/2026-05-01-v02-spec-grammar-value-field-missing-proto-and-tcp-fields, finding/2026-05-01-v02-spec-grammar-rvalue-rejects-pkt-field-contradicts-mybool-eq-tcp-syn]
---

# 2026-05-01-v02-spec-tier2-scalar-expr-missing-proto-keyword

## Summary

The Tier 2 grammar makes `proto`-typed locals one-way: `p = pkt.proto`
is admitted (parses, types as `proto`), but `p = tcp` does **not**
parse — `proto_keyword` is not in the `scalar_expr` production. So
the type system has a `proto` type with no way to construct a literal
of it inside an assignment. Once a `proto` local exists, the only way
its value can come into being is from a `pkt.proto` read.

Class B (analyzer/grammar inconsistency; spec layer).

## Locations

### `scalar_expr` enumerates literal kinds — no `proto_keyword`

`FWL_V02_SPEC.md:1107-1110`:

```ebnf
scalar_expr   = condition
              | identifier                        (* local read *)
              | value_field                       (* packet field read *)
              | integer | ipv4 | ipv6 ;
```

The literal alternatives are `integer`, `ipv4`, `ipv6`. There is no
`proto_keyword` alternative.

### `proto` is a first-class local type

`FWL_V02_SPEC.md:540-547`:

| Type | Width | Examples of values |
|---|---|---|
| `proto` | 8 bits | `pkt.proto` |

`FWL_V02_SPEC.md:549`:

> The `proto` type is opaque: locals of type `proto` may only
> participate in equality and inequality comparisons against
> `proto_keyword` values (`tcp`, `udp`, `icmp`, `icmp6`).

So the type-rules table promises `proto` locals exist and that they
can be compared against `proto_keyword` literals. But the grammar
gives no production that would let a user *create* a `proto` literal
in any non-comparison context.

### `pkt.proto` IS in `value_field`

After the earlier fix
(`finding/2026-05-01-v02-spec-grammar-value-field-missing-proto-and-tcp-fields`),
`value_field` now lists `pkt.proto`
(`FWL_V02_SPEC.md:1123-1127`). So `p = pkt.proto` parses (RHS is a
`value_field`). The first-assignment rule
(`FWL_V02_SPEC.md:537-538`) types `p` as `proto`. So `proto` locals
exist — they just can't be (re)assigned a literal.

### Concretely: programs the spec implies are valid that don't parse

```python
def firewall(pkt):
  my_proto = tcp                    # parse error: tcp is a proto_keyword,
                                    # not in scalar_expr
  if pkt.proto == my_proto:         # would be valid per the type rule
    drop                            # at line 549 (eq compare against proto)
  allow
```

```python
def firewall(pkt):
  p = pkt.proto                     # OK — value_field on RHS
  if p == tcp:                      # OK — comparison admits proto_keyword
    drop                            # via `rvalue=operand`
  p = udp                           # parse error again
  if p == udp:
    log
  allow
```

Reading and comparing `proto` locals works. Assigning a `proto`
literal does not. The asymmetry is invisible in the type-rules
table.

### And: the comparison form is symmetric in ways the assignment form isn't

`pkt.proto == tcp` parses (lvalue = value_field, rvalue = operand =
proto_keyword). `tcp == pkt.proto` does not parse (lvalue = ???; the
LHS of a comparison must be a `value_field` or `identifier`, neither
of which a bare `tcp` is). The same asymmetry hits the assignment:
the proto_keyword cannot stand on its own anywhere except inside a
comparison's rvalue slot.

## Why this matters

Three concrete consequences:

1. **No way to hoist a proto check.** A user who wants the same
   `pkt.proto == tcp` test repeated across several `if`s can't do
   `is_tcp = pkt.proto == tcp` (works — `is_tcp` is `bool`) and call
   it a day. They cannot do the symmetric `target = tcp; if pkt.proto
   == target:` factoring. A reader of the type-rules table at line
   540-547 would expect both factorings to work.

2. **Reassignment of a proto local is impossible.**
   `FWL_V02_SPEC.md:556-557` says "Re-assigning a local with the same
   type is permitted." But for a `proto` local, the only RHS that
   yields a `proto` value is another `pkt.proto` read — the exact same
   right-hand side as the first assignment. Reassignment is permitted
   in name only.

3. **The type system documents a fifth type that has no constructor.**
   `bool`/`u16`/`u32`/`ipv4`/`ipv6` all have literal constructors
   (booleans via comparisons, integers/addresses as literals). `proto`
   is the only type in the table whose only constructor is a packet
   read. That asymmetry is undocumented and surprising.

## Three possible resolutions

| Resolution | What changes |
|---|---|
| (a) Extend `scalar_expr` to admit `proto_keyword` | Add `proto_keyword` to the alternatives in `FWL_V02_SPEC.md:1107-1110`. `my_proto = tcp` then parses; the analyser types it as `proto`. Mirror of how `ipv4` and `ipv6` literals work. |
| (b) Spec out the asymmetry | Add a Type-rules paragraph saying "proto literals are comparison-only operands; they cannot be assigned to a local. Construct a `proto` local by reading `pkt.proto`." Then add a Compile-error row for `local = <proto_keyword>`. |
| (c) Drop the `proto` local type entirely | Remove `proto` from the local-type table; reads of `pkt.proto` in scalar_expr position become a compile error (or are coerced to a `u8` which the type rules then reject from `proto_keyword` comparisons). Loses the ability to hoist `p = pkt.proto`. |

(a) is the smallest change and matches user intuition. (b) preserves
the existing surface but documents the gap. (c) loses functionality
the spec already promises.

## Proposed fix

Pick (a). Concretely, replace `FWL_V02_SPEC.md:1107-1110` with:

```ebnf
scalar_expr   = condition
              | identifier                        (* local read *)
              | value_field                       (* packet field read *)
              | integer | ipv4 | ipv6
              | proto_keyword ;                   (* tcp/udp/icmp/icmp6 *)
```

Then add one Type-rules clarification:

> A `proto_keyword` literal (`tcp`, `udp`, `icmp`, `icmp6`) on the
> RHS of an assignment types the local as `proto`. The first-assignment
> rule applies symmetrically whether the RHS is a packet read or a
> literal.

The analyser's existing rule that locals of type `proto` accept only
`==`/`!=` against `proto_keyword` is unchanged.

## Class

Class B (grammar/type-rule contradiction). Spec-layer.
