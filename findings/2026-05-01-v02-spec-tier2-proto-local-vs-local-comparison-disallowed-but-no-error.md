---
id: finding/2026-05-01-v02-spec-tier2-proto-local-vs-local-comparison-disallowed-but-no-error
type: finding
protocol: []
builtins: []
severity: medium
layer: spec
pattern_tags: [spec-ambiguity, spec-inconsistency, tier2, locals, type-rules, proto, error-messages]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: [finding/2026-05-01-v02-spec-tier2-non-bool-local-as-bare-condition-undefined, finding/2026-05-01-v02-spec-tier2-scalar-expr-missing-proto-keyword]
---

# 2026-05-01-v02-spec-tier2-proto-local-vs-local-comparison-disallowed-but-no-error

## Summary

The Tier 2 type rules at `FWL_V02_SPEC.md:578` constrain
`proto`-typed locals with one short sentence:

> The `proto` type is opaque: locals of type `proto` may only
> participate in equality and inequality comparisons against
> `proto_keyword` values (`tcp`, `udp`, `icmp`, `icmp6`).
> Arithmetic, ordered comparisons, and `in` are all type errors on
> a `proto` local.

Read literally, this **disallows** comparing a `proto` local
against another `proto` local or against `pkt.proto`, because those
right-hand sides are not `proto_keyword` tokens — they are an
`identifier` and a `value_field`, respectively, both of which the
v0.2 grammar admits as `rvalue` (`FWL_V02_SPEC.md:1122-1124`).

The spec creates two problems by stopping there:

1. **No compile-error message is specified.** The Tier 2 Compile-
   errors table (`:712-737`) does not list "proto local compared
   against a non-`proto_keyword` rvalue" as an error class. The
   Operators delta section (`:882-898`) doesn't mention it. An
   implementor who reads the type-rules sentence and tries to
   honour it has nothing to put in the compiler's error message.

2. **The restriction is unmotivated and surprises the user.** A
   natural Tier 2 idiom that the type rule blocks:

   ```python
   def firewall(pkt):
     p = pkt.proto
     if p == tcp:
       count tcp_seen
     if p == udp:
       count udp_seen
     # ... the user adds a "save the previous packet's proto"
     # idiom and the spec rejects it:
     last_proto = p           # u8/proto local, fine
     if p == last_proto:      # rejected: rvalue is identifier,
                              # not proto_keyword
       count repeat
     allow
   ```

   The motivation cited in the spec — that treating `pkt.proto` as
   an integer "invites bugs where, for example, `pkt.proto < udp`
   happens to be defined by the byte values `1, 6, 17, 58` but
   means nothing operationally" — argues against ordered
   comparisons and arithmetic, not against equality of two opaque
   values. `p1 == p2` makes perfect sense for opaque enums.

Class B (spec inconsistency: the type-rules sentence forbids a
form, the grammar permits it, the error table doesn't surface it).

## What the spec actually says vs. what the grammar admits

### Type rule — proto restriction (line 578)

```
The `proto` type is opaque: locals of type `proto` may only
participate in equality and inequality comparisons against
`proto_keyword` values (`tcp`, `udp`, `icmp`, `icmp6`).
```

Strict reading: the only legal RHS for `==`/`!=` on a `proto`
local is one of the four `proto_keyword` tokens. `p == otherp`,
`p == pkt.proto`, `pkt.proto == p` are all forbidden.

### Grammar — `rvalue` admits identifier and value_field (line 1122-1124)

```ebnf
rvalue          = operand
                | identifier
                | value_field ;             (* a comparison may read *)
                                            (* a packet field on    *)
                                            (* either side; analyser *)
                                            (* still type-checks    *)
```

`operand = integer | ipv4 | ipv6 | proto_keyword`
(`FWL_V02_SPEC.md:1164`). So `rvalue` includes `proto_keyword`
*plus* `identifier` (any local, including a `proto`-typed one)
*plus* `value_field` (including `pkt.proto`).

### Compile-errors table — no row (line 712-737)

The Tier 2 Compile-errors table covers stack-budget exceeded,
local read before assigned, local re-assigned with different type,
list/range/cidr RHS in assignment, local named `pkt`, and so on.
There is no row for "proto local compared against a non-
`proto_keyword` rvalue" or "proto local compared against another
proto local". The implementor is expected to derive the error
message from the type-rules sentence — but the sentence gives no
message.

## Concrete program the spec self-contradicts on

```python
@xdp(eth0)

def firewall(pkt):
  if pkt.src_ip in [10.0.0.0/8]:
    p = pkt.proto         # bound as proto local
    q = pkt.proto         # also bound as proto local
    if p == q:            # spec rejects (q is identifier, not
                          # proto_keyword), but no error message
                          # is defined and `p == q` is always true
      count internal_proto_match
  allow
```

Per `:578`, `if p == q:` is invalid. Per the grammar, it parses.
Per the Compile-errors table, no error message exists. An
implementor faces three options, all spec-conformant:

(a) Honour the type rule, invent an error message.
(b) Ignore the type rule, accept `p == q` as always-true (since
    both reads of `pkt.proto` on the same packet yield the same
    value).
(c) Honour the type rule for `p == p` but accept `p == pkt.proto`
    (the spec doesn't distinguish identifier-rvalue from
    value-field-rvalue for proto locals).

Different analysers will land on different answers.

## Why this matters

This is structurally the same hole as
`finding/2026-05-01-v02-spec-tier2-non-bool-local-as-bare-condition-undefined`:
a type-rule sentence that constrains one column of a (local-type ×
operator) matrix without filling in the corresponding error-table
row. The proto case is sharper because the constraint is *more*
restrictive than the grammar — every implementor will either
under-implement (let `p == q` through) or over-implement (invent
an error message), and the corpus oracle has no spec text to
appeal to.

The lossy-equality case for opaque enums is also a real user
idiom. Phase-2's only motivation for the proto-keyword-only rule
is "ordered comparisons and arithmetic invite bugs". Equality
between two proto locals doesn't share that pathology.

## Proposed fix

Two changes, both small.

**1. Loosen the type rule to admit equality between proto-typed
operands.** Replace `:578` with:

> The `proto` type is opaque. Locals of type `proto` may
> participate in equality and inequality comparisons (`==`, `!=`)
> against any other `proto`-typed value — a `proto_keyword` token
> (`tcp`, `udp`, `icmp`, `icmp6`), the `pkt.proto` field, or
> another `proto`-typed local. Arithmetic, ordered comparisons,
> and `in` are all type errors on a `proto` local. The motivation
> for excluding ordered comparisons is that `pkt.proto < udp`
> happens to be defined by the byte values `1, 6, 17, 58` but
> means nothing operationally; equality, by contrast, has the
> obvious meaning for an opaque enum.

This matches the bool-vs-bool rule (`:707`: "both sides of `==`/
`!=` may be a `bool` local or a `bool` packet field").

**2. Add an explicit Compile-errors row.** In the Tier 2 table
(`:712-737`):

| Condition | Error |
|---|---|
| `proto`-typed value used with `<`, `<=`, `>`, `>=`, `in`, or arithmetic | `error: 'proto' values support only equality (`==`, `!=`) — got '<op>'` |
| `proto`-typed value compared against a non-`proto`-typed rvalue | `error: cannot compare proto value with <T> value` |

The first row picks up the operators the spec already disallows;
the second row picks up the type-mismatch case (which the spec
already covers in the abstract under "mixing types is a type
error" at `:707`, but doesn't enumerate for `proto`).

## Class

Class B — spec inconsistency. A type-rule sentence forbids a form
the grammar admits; the Compile-errors table omits the
corresponding error message; the prohibition is also tighter than
the cited motivation justifies (excludes equality between two
opaque enums, which is sound). Spec layer.
