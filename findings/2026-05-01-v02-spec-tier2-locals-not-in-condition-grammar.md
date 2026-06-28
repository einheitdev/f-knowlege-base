---
id: finding/2026-05-01-v02-spec-tier2-locals-not-in-condition-grammar
type: finding
protocol: []
builtins: []
severity: high
layer: spec
pattern_tags: [spec-inconsistency, grammar, tier2, locals, example-vs-rule]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: [finding/2026-05-01-v02-spec-grammar-local-field-undefined-nonterminal, finding/2026-05-01-v02-spec-tier2-example5-violates-grammar-and-dominator-rule]
---

# 2026-05-01-v02-spec-tier2-locals-not-in-condition-grammar

## Summary

The v0.2 grammar at `FWL_V02_SPEC.md:1049-1051` declares `condition`
"reused from v0.1 unchanged", with only two listed extensions:
v6 fields (`pkt.src_ip6`, `pkt.dst_ip6`) on the LHS and `geoip_call`
on the RHS of `in`.

Locals are **not** added to `condition`. But Tier 2 examples and
edge cases require them in three positions:

1. As a bare boolean condition: `if internal:` (Example 4).
2. As the LHS of a comparison: `if a == 10.0.0.1:`, `if e == 22:`
   (Example 5 — see the rewrite proposed in
   `finding/2026-05-01-v02-spec-tier2-example5-violates-grammar-and-dominator-rule`).
3. As the RHS of a comparison: `dst_port == my_bool` (Edge case
   line 673-674), `if x == 22` semantics implied throughout the
   "Type rules" prose.

Per the grammar, none of these forms parse. v0.1's `condition` →
`primary` is `comparison | bool_field | "(" condition ")"`; its
`comparison = field comp_op operand` where `field = value_field |
enum_field | bool_field` (all packet fields, no locals); its
`operand = integer | ipv4 | proto_keyword` (no locals).

This is a complementary gap to the already-fixed
`finding/2026-05-01-v02-spec-grammar-local-field-undefined-nonterminal`
— that one extended `expression` (RHS of `assign_stmt`) to admit
`identifier` for local reads. But `condition` was not extended in
parallel, even though Tier 2 if/elif conditions need to read
locals.

Class B (analyzer/spec contradiction; spec layer).

## Locations

### Grammar declaration

`FWL_V02_SPEC.md:1049-1051`:

```
(* `condition` is reused from v0.1 unchanged; see FWL_V01_SPEC.md.
   v0.2 conditions may additionally use `pkt.src_ip6` / `pkt.dst_ip6`
   on the LHS and `geoip_call` on the RHS of `in`. *)
```

The list of additions does not include local identifiers.

### v0.1 condition (the import target)

`FWL_V01_SPEC.md:563-602`:

```ebnf
condition = or_expr ;
or_expr   = and_expr { "or" and_expr } ;
and_expr  = not_expr { "and" not_expr } ;
not_expr  = [ "not" ] primary ;
primary   = comparison | bool_field | "(" condition ")" ;

comparison = field comp_op operand
           | field "in" set_or_range ;

field    = value_field | enum_field | bool_field ;
operand  = integer | ipv4 | proto_keyword ;
```

`primary` admits no `identifier`. `field` admits no `identifier`.
`operand` admits no `identifier`. So a v0.2 `condition` cannot
mention a Tier 2 local in any position.

### Example 4 — bare boolean local as condition

`FWL_V02_SPEC.md:763-777`:

```python
def firewall(pkt):
  internal = pkt.src_ip in [10.0.0.0/8, 192.168.0.0/16]
  if internal:
    allow
  if pkt.proto == tcp and pkt.dst_port in [80, 443]:
    allow
  drop
```

The example introduces a bool local `internal` and tests it with
`if internal:`. The grammar's `primary` does not include
`identifier`, so the parser has no rule to admit `internal` as a
condition. The line below the example ("`internal` is a `bool`
local, assigned once, read once") asserts the construct works — but
the grammar disagrees.

### Example 5 (post-fix) — local as comparison LHS

The fix proposed in
`finding/2026-05-01-v02-spec-tier2-example5-violates-grammar-and-dominator-rule`
rewrites the stack-budget example to:

```python
if a == 10.0.0.1:
  allow
if b == 10.0.0.2:
  drop
if c == 2001:db8::1:
  allow
if d == 2001:db8::2:
  drop
if pkt.proto == tcp or pkt.proto == udp:
  e = pkt.src_port
  f = pkt.dst_port
  if e == 22:
    log
  if f == 22:
    count ssh
```

Each `if a == ...`, `if e == 22`, `if f == 22` puts a Tier 2 local
on the LHS of a `comparison`. Per the grammar, the LHS of
`comparison` is `field`, never `identifier`. None of these `if`s
parse.

The fix-finding's rewrite is grammar-illegal even after fixing the
single-line and dominator issues. The ~64-byte stack estimate the
example is supposed to demonstrate is unreachable.

### Edge case — local as comparison RHS

`FWL_V02_SPEC.md:672-674`:

> *Boolean in non-bool context.* Locals of type `bool` are not
> numerically convertible. `count <n> if my_bool` — the local is
> used as a bool and is fine. `dst_port == my_bool` — type error.

Two issues here:

a. `count <n> if my_bool` — this is **Tier 1 rule** syntax (action
   followed by `if cond`). In Tier 2, `count <n>` is a statement,
   not a rule, and has no trailing `if cond`. The example uses a
   Tier 2 local in a Tier 1 rule, which doesn't compose.

b. `dst_port == my_bool` is described as a "type error". For it to
   be a type error rather than a parse error, the parser must
   accept `dst_port == my_bool` (i.e., locals on the RHS of a
   comparison). But `operand = integer | ipv4 | proto_keyword`
   admits no identifier — this is a parse error, not a type error.

The narrative is consistent only if the grammar admits locals in
both LHS and RHS of comparisons; the grammar does not.

## Why this matters

Tier 2 is the v0.2 construct most dependent on local-variable
plumbing. The grammar is the authoritative parser reference, so
implementors of the v0.2 parser will face a choice:

| Choice | Consequence |
|---|---|
| Implement the grammar literally | All four examples and several edge cases fail to parse. The corpus author either edits the examples (defeating their purpose) or marks them `expected.compiles: false` (contradicting the spec's narrative). |
| Quietly extend `condition` to admit locals | The compiler accepts programs the spec does not document, and the surface area is undefined: are locals admitted as `primary` only? Also as `field`? Also as `operand`? Multi-implementer drift is guaranteed. |

This is the same class of bug Gate 1 is meant to catch: the spec
is the contract the implementer reads, and silent disagreement
between spec text and grammar leaves the contract ambiguous.

## Proposed fix

Replace the comment at `FWL_V02_SPEC.md:1049-1051` with explicit
grammar productions. One concrete proposal:

```ebnf
(* v0.2 condition extends v0.1's by adding two new primary forms
   and two new operand forms. The base productions are:        *)

condition  = or_expr ;
or_expr    = and_expr { "or" and_expr } ;
and_expr   = not_expr { "and" not_expr } ;
not_expr   = [ "not" ] primary ;

primary    = comparison
           | bool_field
           | identifier               (* NEW in v0.2: bool local *)
           | "(" condition ")" ;

comparison = lvalue comp_op rvalue
           | lvalue "in" set_or_range ;

lvalue     = value_field | enum_field | bool_field
           | identifier ;             (* NEW in v0.2: local read *)

rvalue     = operand
           | identifier ;             (* NEW in v0.2: local read *)

operand    = integer | ipv4 | ipv6 | proto_keyword ;
```

Then state the type-rule constraints in prose:

- A bare `identifier` in `primary` position must resolve to a local
  of type `bool`.
- The `lvalue` and `rvalue` of a comparison must resolve to types
  compatible per the v0.1 type-rules table (extended for `ipv6`
  in v0.2).

Or, if the spec authors do not want locals in conditions, fix the
examples:

- Replace `if internal:` with the inlined comparison
  (`if pkt.src_ip in [10.0.0.0/8, 192.168.0.0/16]:`).
- Replace Example 5's `if a == 10.0.0.1:` with `if pkt.src_ip ==
  10.0.0.1:` — but this defeats the example's purpose (it was
  supposed to lock the stack-budget estimator with six locals).

The first fix (extending the grammar) is consistent with the
already-applied fix to `expression` and is almost certainly the
intent.

## Class

Class B — spec/grammar inconsistency. Spec layer. Blocks the
parser implementation: an implementer who follows the grammar
literally rejects three of the five Tier 2 examples; an implementer
who follows the prose accepts a surface the grammar does not
document.
