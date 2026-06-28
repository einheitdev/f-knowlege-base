---
id: finding/2026-05-01-v02-spec-grammar-value-field-missing-proto-and-tcp-fields
type: finding
protocol: [tcp]
builtins: []
severity: medium
layer: spec
pattern_tags: [spec-inconsistency, grammar, tier2, type-rules]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: [finding/2026-05-01-v02-spec-grammar-local-field-undefined-nonterminal]
---

# 2026-05-01-v02-spec-grammar-value-field-missing-proto-and-tcp-fields

## Summary

The v0.2 grammar's `value_field` production (`FWL_V02_SPEC.md:1056-1058`)
lists only the six v4/v6 address and port fields. It omits
`pkt.proto`, `pkt.tcp.syn`, and `pkt.tcp.ack`. Tier 2's
`expression` production (`FWL_V02_SPEC.md:1046-1050`) admits a
`value_field` as a bare RHS but no other production for these
fields, so a Tier 2 assignment like

```python
syn = pkt.tcp.syn
proto_val = pkt.proto
```

does not parse. Yet the Tier 2 type-rules table at
`FWL_V02_SPEC.md:548-554` explicitly lists `pkt.tcp.syn` as an
example of a `bool` value source — implying the assignment *is*
expected to work.

Class B (grammar/type-rules inconsistency). Mirrors the earlier
`local_field` finding but for a different set of missing fields.

## Locations

### Type rules — `pkt.tcp.syn` is a valid `bool` source

`FWL_V02_SPEC.md:548-554`:

```
| Type | Width | Examples of values |
|---|---|---|
| `bool` | 1 bit (stored as u8) | `pkt.tcp.syn`, `pkt.dst_port == 80`, `pkt.src_ip in geoip(RU)` |
| `u16`  | 16 bits | `pkt.src_port`, `pkt.dst_port`, integer literals 0..65535 |
| `u32`  | 32 bits | integer literals fitting u32 |
| `ipv4` | 32 bits | `pkt.src_ip`, `pkt.dst_ip`, IPv4 literals |
| `ipv6` | 128 bits | `pkt.src_ip6`, `pkt.dst_ip6`, IPv6 literals |
```

The first row says `pkt.tcp.syn` is a bool. The natural reading:
`syn = pkt.tcp.syn` is a valid Tier 2 assignment with `syn :: bool`.

### Grammar — `value_field` doesn't include the bool fields

`FWL_V02_SPEC.md:1056-1058`:

```ebnf
value_field   = "pkt.src_ip"  | "pkt.dst_ip"
              | "pkt.src_ip6" | "pkt.dst_ip6"
              | "pkt.src_port" | "pkt.dst_port" ;
```

Six fields. `pkt.proto`, `pkt.tcp.syn`, `pkt.tcp.ack` are absent.

`expression` (`FWL_V02_SPEC.md:1046-1050`):

```ebnf
expression    = condition
              | identifier                        (* local read *)
              | value_field                       (* packet field read *)
              | integer | ipv4 | ipv6
              | cidr | list | cidr_list ;
```

A bare `pkt.tcp.syn` as the RHS of an assign matches none of these
alternatives:

- `condition` (v0.1) requires a comparison operator (`==`, `!=`,
  etc.). A bare boolean field by itself does not match `condition`.
  (`condition` in v0.1 is `<field> <op> <operand>` plus `and`/`or`/
  `not` over conditions.)
- `value_field` is enumerated and excludes `pkt.tcp.syn`.
- The remaining alternatives are literal types or lists.

So `syn = pkt.tcp.syn` does not parse under the v0.2 grammar.

The same gap applies to:

- `proto_val = pkt.proto` — `pkt.proto` is referenced throughout the
  spec but its **type** is not in the type-rules table either (no
  enum row), so this is doubly under-specified.
- `ack = pkt.tcp.ack` — same problem as `pkt.tcp.syn`.

### v0.1 doesn't help

`FWL_V01_SPEC.md` has `pkt.tcp.syn`/`pkt.tcp.ack`/`pkt.proto` only
as the LHS of a comparison; there is no v0.1 production that lets
them stand alone. Importing v0.1's `condition` doesn't lift them
into v0.2's `expression`.

## Why this matters

Tier 2 promises "locals can hold any pkt-field value" via the type
table. A user who wants to memoise a TCP-flag check writes:

```python
def firewall(pkt):
  if pkt.proto == tcp:
    syn = pkt.tcp.syn
    if syn and not pkt.tcp.ack:
      ...
```

Per the type table this should produce `syn :: bool`. Per the
grammar it is a parse error. The spec contradicts itself.

The dogfood example at `FWL_V02_SPEC.md:953-956` sidesteps the issue
by using `pkt.tcp.syn` only inside an `if` condition, never as a
RHS. So the spec's *own* worked examples never exercise the gap —
which is precisely the class of bug a reviewer must spot before the
analyser code lands.

There is no compile-error row covering "RHS of assignment is a
boolean pkt-field" either, so the diagnostic is undefined.

## Three possible resolutions

| Resolution | What changes |
|---|---|
| (a) Extend `value_field` | Add `pkt.proto`, `pkt.tcp.syn`, `pkt.tcp.ack` to the `value_field` enumeration. Add a `proto_enum` row to the type-rules table for `pkt.proto`. |
| (b) Reject the surface | Drop `pkt.tcp.syn` from the `bool` row of the type table. Add a compile-error row: `error: '<field>' may only appear inside an if-condition`. The motivation: keep boolean and proto-enum reads scoped to predicates. |
| (c) Status quo + analyser-only check | Leave grammar as-is; have the analyser raise `error: cannot read '<field>' in expression position` when it sees one. Then revise the type-rules table's `bool` row to remove `pkt.tcp.syn` from "Examples of values" because it is *not* a usable bool source. |

## Proposed fix

Pick (a). It's the most consistent with the rest of the type table
and matches the natural reading of "Locals are statically typed.
Their type is the type of the right-hand side of the **first**
assignment to the name within the function" (`FWL_V02_SPEC.md:545-546`).

Concretely, replace `FWL_V02_SPEC.md:1056-1058` with:

```ebnf
value_field   = "pkt.src_ip"  | "pkt.dst_ip"
              | "pkt.src_ip6" | "pkt.dst_ip6"
              | "pkt.src_port" | "pkt.dst_port"
              | "pkt.proto"
              | "pkt.tcp.syn" | "pkt.tcp.ack" ;
```

And add a row to the type-rules table:

```
| `proto` | 8 bits | `pkt.proto` (compares against proto_keyword only) |
```

with a clarifying note that `proto` locals are restricted to
equality with `proto_keyword` literals (no arithmetic, no ordered
compares).

## Class

Class B (spec/grammar inconsistency). Spec-layer.
