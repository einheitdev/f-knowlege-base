---
id: finding/2026-05-01-v02-spec-grammar-no-function-call-form-promises-unreachable-errors
type: finding
protocol: []
builtins: [geoip, rate_limit]
severity: medium
layer: spec
pattern_tags: [spec-inconsistency, grammar, tier2, error-messages, dead-rule]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: [finding/2026-05-01-v02-spec-grammar-operand-overpermissive-for-geoip-call]
---

# 2026-05-01-v02-spec-grammar-no-function-call-form-promises-unreachable-errors

## Summary

The v0.2 grammar admits **no general function-call production**.
The only function-shaped surfaces are the two explicit terminals
`geoip_call` and `rate_limit_call`, and each appears in exactly one
position (`set_or_range` for `geoip_call`, `primary` for
`rate_limit_call`). There is no `call_expr` and no
`identifier "(" ... ")"` form anywhere in the grammar.

But the spec lists three compile errors whose triggering surface is
"a user wrote a function call":

1. `FWL_V02_SPEC.md:692-695` (Tier 2 edge case): *Calling a function
   other than `rate_limit` or `geoip`.* → `error: unknown function
   '<name>'`.
2. `FWL_V02_SPEC.md:696-698` (Tier 2 edge case): *Recursion.*
   Calling the entry function from within itself, by name → `error:
   recursion not allowed; '<name>' cannot call itself`.
3. `FWL_V02_SPEC.md:463-464` (geoip Compile errors table): *Nested
   function call* → `error: geoip() arguments must be country
   codes; nested calls are not allowed`.

The same three rows are restated in the Tier 2 Compile errors table
(`FWL_V02_SPEC.md:730-731`).

Per the grammar, a user who writes `firewall(pkt)`, `f(x)`, or
`geoip(geoip(RU))` does **not** trigger any of these analyzer
messages — they trigger a parser-level "unexpected token `(`" /
"expected `)`" error, because no production derives the call form.
The promised user-visible strings are unreachable.

Class B (spec/grammar inconsistency; spec layer).

## What the grammar actually admits

### Statements (`FWL_V02_SPEC.md:1062-1069`)

```ebnf
statement     = if_stmt
              | assign_stmt
              | action_stmt ;

action_stmt   = "allow"            NEWLINE
              | "drop"             NEWLINE
              | "log"              NEWLINE
              | "count" identifier NEWLINE ;
```

A bare `firewall(pkt)` statement matches none of `if_stmt`,
`assign_stmt`, or `action_stmt`. Parser rejects.

### Assignment RHS (`FWL_V02_SPEC.md:1071, 1133-1137`)

```ebnf
assign_stmt   = identifier "=" scalar_expr NEWLINE ;

scalar_expr   = condition
              | identifier                        (* local read *)
              | value_field                       (* packet field read *)
              | integer | ipv4 | ipv6
              | proto_keyword ;
```

`scalar_expr` does not derive any function-call form. `x =
firewall(pkt)` is a syntax error at `(`.

### Comparison sides (`FWL_V02_SPEC.md:1114-1124`)

```ebnf
comparison      = lvalue comp_op rvalue
                | lvalue "in" set_or_range ;

lvalue          = value_field | identifier ;
rvalue          = operand | identifier | value_field ;
```

Neither `lvalue` nor `rvalue` derives a call form. `x == f(y)` is a
syntax error.

### Primary (`FWL_V02_SPEC.md:1105-1112`)

```ebnf
primary         = comparison
                | bool_primary
                | rate_limit_call
                | "(" condition ")" ;

bool_primary    = "pkt.tcp.syn"
                | "pkt.tcp.ack"
                | identifier ;
```

The only call-shaped primary is `rate_limit_call`. A bare
`if firewall(pkt):` doesn't match; `firewall` parses as a
`bool_primary`'s `identifier`, then the parser sees `(` and has no
production that takes a `(` after a primary. Syntax error.

### `geoip_call` argument list (`FWL_V02_SPEC.md:1185-1186`)

```ebnf
geoip_call    = "geoip" "(" cc_code { "," cc_code } ")" ;
cc_code       = upper letter , upper letter ;       (* exactly two *)
```

Inside `geoip(...)`, only `cc_code` (exactly two uppercase letters)
is admitted. `geoip(geoip(RU))`: the inner `geoip` does not match
`cc_code` (six letters, lowercase); parser rejects with the same
"expected uppercase letter pair" error it would emit for `geoip(ru)`
or `geoip(toolong)`. The "nested calls are not allowed" message is
unreachable.

## The unreachable error messages

| Promised error | Triggering surface per spec | Why unreachable |
|---|---|---|
| `error: unknown function '<name>'` | "Calling a function other than `rate_limit` or `geoip`" (`:692-695`, `:730`) | Grammar has no general call form; `f(x)` is a parser error before the analyzer runs |
| `error: recursion not allowed; '<name>' cannot call itself` | "Calling the entry function from within itself, by name" (`:696-698`, `:731`) | Same — the body cannot syntactically contain a call to its own name |
| `error: geoip() arguments must be country codes; nested calls are not allowed` | `geoip(geoip(RU))`, `geoip(rate_limit(...))`, `geoip(f(...))` (`:463-464`) | `geoip(...)` only accepts `cc_code` tokens; inner identifier fails the `[A-Z][A-Z]` shape and parser rejects with the same "not a country code" error |

## Why this matters

Three angles, in order of severity:

1. **Implementor confusion.** A compiler engineer reading the spec
   sees specific error strings that the grammar makes impossible to
   emit. They will either spend time wiring up unreachable analyzer
   passes, or they will leave them out and a reviewer will flag the
   missing test coverage. The spec gives them no way to know which
   answer is correct.

2. **`.pkt` corpus author confusion.** A test author writing a
   `compile_error.pattern` for "user calls `firewall(pkt)` from
   itself" reads the spec, copies the literal `"recursion not
   allowed"` text into the regex, and the test fails forever
   because the actual emitted error is the parser's "unexpected
   token `(`". The corpus oracle disagrees with the spec.

3. **Spec credibility.** The spec gives the impression that v0.2
   has a function-call surface (the table entries imply users
   *could* try to call functions and the analyser will catch it).
   The deferral list at `:1041` also reinforces this:

   > **Loops**, **recursion**, **closures**, **list/dict mutation**,
   > **user-defined functions other than the entry point**.

   "Recursion" being deferred presumes recursion was ever a
   syntactic possibility; it isn't.

## Proposed fix

Pick one of two resolutions consistently across the three sites:

**Option A — drop the unreachable error rows.** The grammar already
makes the surfaces impossible. Remove:

- `FWL_V02_SPEC.md:692-698` (Tier 2 edge cases for "Calling a
  function other than..." and "Recursion").
- `FWL_V02_SPEC.md:730-731` (Tier 2 Compile errors table rows for
  the same).
- `FWL_V02_SPEC.md:463-464` (geoip Compile errors table row for
  "Nested function call").

Replace each with a one-line note: "*Function-call syntax other
than `geoip(...)` and `rate_limit(...)` is not admitted by the
grammar; an attempted call is a parser error.*"

**Option B — extend the grammar to admit a general call form, then
catch it in the analyzer.** Add a `call_expr` production:

```ebnf
call_expr   = identifier "(" [ expr { "," expr } ] ")" ;
primary    += call_expr ;
scalar_expr += call_expr ;
```

Then the analyser sees `f(x)` as a `call_expr` whose `identifier`
is not `rate_limit` or `geoip`, and emits the promised "unknown
function" message. Recursion is detected by the analyser when
`identifier` matches the entry function's name. Nested calls inside
`geoip(...)` are detected by adding `cc_code | call_expr` to the
geoip arg list and emitting the promised "nested calls" message
when a `call_expr` shows up.

Option B grows the grammar surface for nicer error messages; option
A trims the spec to match the grammar. v0.2's deferral list ("no
user-defined functions other than the entry point") suggests A is
the intent, but the spec needs to commit to one.

## Class

Class B — spec/grammar inconsistency. The grammar makes three
compile errors structurally unreachable; the prose and the
compile-error tables nonetheless promise specific user-visible
messages for them. Spec layer.
