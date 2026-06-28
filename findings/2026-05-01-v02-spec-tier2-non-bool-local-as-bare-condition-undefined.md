---
id: finding/2026-05-01-v02-spec-tier2-non-bool-local-as-bare-condition-undefined
type: finding
protocol: []
builtins: []
severity: medium
layer: spec
pattern_tags: [spec-ambiguity, grammar, tier2, locals, type-rules]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: [finding/2026-05-01-v02-spec-tier2-locals-not-in-condition-grammar]
---

# 2026-05-01-v02-spec-tier2-non-bool-local-as-bare-condition-undefined

## Summary

The v0.2 grammar admits any local identifier as a bare `primary`
inside an `if` / `elif` / `not` condition (via `bool_primary =
"pkt.tcp.syn" | "pkt.tcp.ack" | identifier`). The Tier 2 type rules
explicitly bless this surface for `bool` locals only:

> *Boolean in non-bool context.* Locals of type `bool` are not
> numerically convertible. Inside a Tier 2 `if` condition, a bare
> bool local is a valid `primary` (`if my_bool:`).
> — `FWL_V02_SPEC.md:707`.

But the spec never says what happens for non-`bool` locals. `if
my_u16:`, `if my_ipv4:`, `if my_proto:`, `if my_ipv6:` all parse,
but no type rule, edge-case bullet, or Compile-errors row addresses
them. An implementor is left to guess whether they:

(a) implicitly truthy-coerce (Python-style: nonzero ⇒ true);
(b) raise a type error ("expected bool");
(c) silently accept and emit BPF that compares the value against
    zero;
(d) reject as a grammar error (which the grammar contradicts).

Two conformant analysers can choose different answers. This is a
direct Class A surface: `if my_u16: drop` followed by a `.pkt` test
where `my_u16 == 0` lands in `XDP_PASS` on one analyser and
`XDP_DROP` on another — **silently**, with no spec rule to
adjudicate.

Class B (spec underspecification; spec layer) with a Class A
runtime surface downstream.

## What the spec says (and doesn't say)

### Grammar — bare identifier is a `primary` (line 1110-1112)

```ebnf
primary         = comparison
                | bool_primary
                | rate_limit_call
                | "(" condition ")" ;

bool_primary    = "pkt.tcp.syn"
                | "pkt.tcp.ack"
                | identifier ;
```

`bool_primary` derives any identifier without a type filter — the
grammar happily accepts `if x:` for any local `x`.

### Type table — the local-type universe (line 569-577)

| Type | Width | Examples of values |
|---|---|---|
| `bool` | 1 bit (stored as u8) | ... |
| `u16`  | 16 bits | `pkt.src_port`, `pkt.dst_port`, integer literals 0..65535 |
| `u32`  | 32 bits | integer literals fitting u32 |
| `ipv4` | 32 bits | `pkt.src_ip`, `pkt.dst_ip`, IPv4 literals |
| `ipv6` | 128 bits | `pkt.src_ip6`, `pkt.dst_ip6`, IPv6 literals |
| `proto`| 8 bits  | `pkt.proto` |

Six types. Five of them (`u16`, `u32`, `ipv4`, `ipv6`, `proto`)
have no defined behaviour as a bare condition primary.

### "Boolean in non-bool context" edge case — only one direction (line 707)

```
*Boolean in non-bool context.* Locals of type `bool` are not
numerically convertible. Inside a Tier 2 `if` condition, a bare
bool local is a valid `primary` (`if my_bool:`). Inside a
comparison, both sides of `==`/`!=` may be a `bool` local or a
`bool` packet field, but mixing types is a type error: `pkt.dst_port
== my_bool` — type error (`u16` vs `bool`); `my_bool == pkt.tcp.syn`
— fine (both `bool`).
```

This bullet's only positive statement is "bare *bool* local is a
valid primary". By symmetry one might expect the converse — "a bare
non-bool local in `primary` position is a type error" — but the
spec does not state it. The parallel rule for the comparison
direction *is* stated (`pkt.dst_port == my_bool` is rejected); the
bare-condition direction is missing.

### Compile-errors table — no entry (line 712-737)

The Tier 2 Compile-errors table lists "Local read before assigned",
"Local re-assigned with different type", "Local named pkt", "RHS of
local assignment is a list, range, or CIDR literal", and many
others. There is no row for "non-bool local used as bare condition
primary".

## Concrete program where two implementations diverge

```python
@xdp(eth0)

def firewall(pkt):
  port = 0                  # u16 local, value 0
  if port:                  # bare u16 local as condition primary
    drop
  allow
```

- **Implementation A** (truthy-coerces u16): `port == 0` ⇒ false ⇒
  `drop` skipped ⇒ `XDP_PASS` on every packet.
- **Implementation B** (rejects with "expected bool, got u16"):
  compile error.
- **Implementation C** (silently emits a `port != 0` test in the
  BPF C): same end behaviour as A but undocumented. A user who
  later writes `port = pkt.dst_port` and `if port: drop` thinks
  they're testing presence; in fact they're dropping every packet
  with non-zero dst_port (i.e., every packet on a TCP/UDP
  connection).

Same source, three behaviours, all spec-conformant — because the
spec doesn't say.

## Why this matters

This is the same shape of hole that
`finding/2026-05-01-v02-spec-tier2-stmt-level-pkt-read-on-wrong-protocol`
opened: the analyser has a defined surface for one combination
(field+condition position) and undefined behaviour for an adjacent
one (field+statement position). Here, the local-type universe
multiplied by the condition-primary position has six cells and only
one is filled in.

The Tier 2 dogfood example
(`FWL_V02_SPEC.md:976-1011`) avoids the surface — every condition
primary is either a comparison, a `bool_primary` packet field
(`pkt.tcp.syn`, `pkt.tcp.ack`), or a parenthesized condition.
Example 4 (`internal = pkt.src_ip in [...]; if internal:`,
`:801-810`) uses a `bool` local. So the corpus accidentally never
exercises the gap, and the gap will land in users' programs.

## Proposed fix

Insert one explicit row in the Tier 2 Compile-errors table
(`FWL_V02_SPEC.md:712-737`):

| Condition | Error |
|---|---|
| Non-`bool` local (or non-`bool` packet field) used as a bare condition primary | `error: '<name>' is type <T>; only bool values are valid as a bare 'if' condition` |

And one positive statement at the end of the "Boolean in non-bool
context" bullet (`:707`):

> Symmetrically, a non-`bool` local or non-`bool` packet field
> used as a bare condition primary is a type error — `if
> my_u16:`, `if pkt.dst_port:`, `if my_proto:` are all rejected.
> The only bare-condition forms are `bool` locals and the two
> `bool` packet fields `pkt.tcp.syn`, `pkt.tcp.ack`.

Optionally, tighten the grammar by adding a comment under
`bool_primary` that the analyser narrows `identifier` to bool-typed
locals. The grammar production stays as-is (the analyser carries
the type filter) — same approach the spec already uses for
`rate_limit_call` (grammar-legal as any `primary`, analyser narrows
to two valid positions, see `FWL_V02_SPEC.md:1194-1199`).

## Class

Class B — spec underspecification at the type-rules layer. The
spec defines behaviour for one cell of a (local-type × condition-
primary) matrix and leaves five cells undefined. Implementations
will diverge silently.
