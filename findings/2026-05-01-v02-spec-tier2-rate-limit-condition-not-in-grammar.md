---
id: finding/2026-05-01-v02-spec-tier2-rate-limit-condition-not-in-grammar
type: finding
protocol: []
builtins: [rate_limit]
severity: high
layer: spec
pattern_tags: [spec-inconsistency, grammar, tier2, rate-limit, example-vs-rule]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: [finding/2026-05-01-v02-spec-tier2-locals-not-in-condition-grammar]
---

# 2026-05-01-v02-spec-tier2-rate-limit-condition-not-in-grammar

## Summary

The v0.2 spec introduces a new syntactic position for
`rate_limit(...)` — as a bool-valued expression inside a Tier 2
`if` condition (`FWL_V02_SPEC.md:614-620`):

> `rate_limit(N, per=<field>)` is valid only inside an `if`
> condition. It evaluates to `bool` (true when the bucket is at or
> above the threshold).

The Tier 2 dogfood example exercises it
(`FWL_V02_SPEC.md:960-963`):

```python
if pkt.proto == tcp and pkt.dst_port == 22:
  if pkt.tcp.syn and not pkt.tcp.ack:
    if rate_limit(10, per=src_ip):
      drop
```

But the v0.2 grammar block (`FWL_V02_SPEC.md:1018-1103`) does **not**
extend `condition` (or `primary`) to admit a `rate_limit_call`. The
v0.1 `primary = comparison | bool_field | "(" condition ")"` is
inherited unchanged, and v0.1's only mention of `rate_limit` in the
grammar is as part of a `modifier` on a Tier 1 rule:

```ebnf
modifier  = "limited" "by" "rate_limit" "(" integer "," "per" "="
            rl_field ")" ;
```

A modifier is not a condition. The grammar admits no syntactic
position where `rate_limit(...)` is itself a boolean expression.

The spec's commentary at line 1049-1051 says `condition` "is reused
from v0.1 unchanged" except for two listed v6/geoip extensions —
`rate_limit_call` is not in the list.

So the spec narrative + dogfood example require a grammar
production the spec does not define.

Class B (analyzer/spec inconsistency, mirrors the locals-in-
conditions gap but for the `rate_limit` built-in).

## Locations

### Narrative — rate_limit as bool in if condition

`FWL_V02_SPEC.md:614-620`:

> - *`rate_limit(...)` and `geoip(...)` inside Tier 2.* Both
>   built-ins are valid inside Tier 2, but only inside the contexts
>   that v0.2 permits:
>   - `rate_limit(N, per=<field>)` is valid only inside an `if`
>     condition. It evaluates to `bool` (true when the bucket is at
>     or above the threshold). A bare statement form
>     (`rate_limit(10, per=src_ip)` on its own line) is a compile
>     error.
>   - `geoip(...)` is valid only as the right-hand side of an `in`
>     expression, exactly as in Tier 1.

Note the asymmetry of treatment in the grammar (lines 1086-1098):

```ebnf
set_or_range  = list | range | cidr | cidr_list | geoip_call ;
...
geoip_call    = "geoip" "(" cc_code { "," cc_code } ")" ;
```

`geoip_call` is grammar-explicit. `rate_limit_call` (in
condition position) has no analogous production.

### Compile errors table

`FWL_V02_SPEC.md:695`:

| Bare `rate_limit(...)` statement | `error: rate_limit(...) is only valid as the condition of an if-statement in Tier 2` |

The error message says rate_limit is only valid "as the condition
of an if-statement". For the analyser to emit this error rather
than a parse error, the parser must accept `rate_limit(...)` in
condition position. The grammar does not provide that.

### Examples that depend on the unspec'd production

- `FWL_V02_SPEC.md:711-712` (Example 1, SSH brute-force):
  ```python
  if rate_limit(10, per=src_ip):
    drop
  ```
- `FWL_V02_SPEC.md:962-963` (consolidated dogfood):
  ```python
  if rate_limit(10, per=src_ip):
    drop
  ```

Both lift `rate_limit(...)` to the condition position of an `if`.
Both fail to parse per the v0.2 grammar.

### v0.1 condition (the import target)

`FWL_V01_SPEC.md:563-602` defines `primary = comparison |
bool_field | "(" condition ")"`. Function calls are not a
`primary`; rate_limit is only ever the body of a `modifier` clause
hanging off a rule. v0.2 does not extend this.

## Why this matters

This is the same shape of gap as
`finding/2026-05-01-v02-spec-tier2-locals-not-in-condition-grammar`:
the spec narrative invents a new syntactic position for an
existing construct, but the grammar block does not admit it. Two
implementors of the v0.2 parser, both reading the spec carefully,
will produce different surfaces:

- One will read the grammar as authoritative and reject every
  `if rate_limit(...):` form, contradicting the dogfood example.
- The other will read the prose as authoritative and silently
  invent a `rate_limit_call` production, with whatever scoping
  rules it picks (does it admit `not rate_limit(...)`? does it
  admit `rate_limit(...) and ...`? can it appear in `(...)`?).

The dogfood program at `f/fwl/examples/dogfood_v02.fw` cannot be
the corpus's regression case if the parser cannot agree with the
example.

## Proposed fix

Extend `primary` in the v0.2 grammar (or replace the v0.1-import
comment with an explicit re-statement) to add a
`rate_limit_call` form, mirroring how `geoip_call` is treated:

```ebnf
primary    = comparison
           | bool_field
           | identifier             (* see locals-in-conditions
                                       finding *)
           | rate_limit_call        (* NEW in v0.2: bool-valued *)
           | "(" condition ")" ;

rate_limit_call =
              "rate_limit" "(" integer ","
              "per" "=" rl_field ")" ;
```

State the contextual restriction in prose: "a `rate_limit_call`
appears only inside an `if` or `elif` condition; using it as the
RHS of a comparison, the operand of `not`, or as a bare statement
is a compile error." (The bare-statement error is already in the
table at line 695.)

If the spec authors prefer a stricter surface — only admit
`rate_limit_call` as the *whole* condition and not as a
sub-expression — say so explicitly and amend the grammar:

```ebnf
if_stmt    = "if" if_condition ":" NEWLINE INDENT … ;
if_condition = condition | rate_limit_call ;
```

Either path closes the gap. The current spec leaves the
implementer to choose silently.

## Class

Class B — spec/grammar inconsistency. The narrative invents a new
position for `rate_limit(...)` (as a bool expression in an if
condition) that the grammar block does not admit. Spec layer.
Blocks parser implementation: every Tier 2 example that uses
rate_limit (including the dogfood) is grammar-illegal as written.
