---
id: finding/2026-05-01-v02-spec-tier2-empty-body-example-uses-pass-which-fires-different-error
type: finding
protocol: []
builtins: []
severity: medium
layer: spec
pattern_tags: [spec-inconsistency, tier2, error-message, example-vs-rule]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: [finding/2026-05-01-v02-spec-tier2-reserved-keyword-set-not-enumerated]
---

# 2026-05-01-v02-spec-tier2-empty-body-example-uses-pass-which-fires-different-error

## Summary

The "Empty function body" Tier 2 edge case at
`FWL_V02_SPEC.md:640-643` says:

> *Empty function body.* `def f(pkt):\n    pass` — `pass` is not in
> v0.2. An empty body is a compile error: `error: function '<name>'
> has empty body; expected at least one statement`.

Two contradictory things in one paragraph:

1. The example *includes* a `pass` statement. The body is therefore
   not empty by line count — it has one token, namely the `pass`
   keyword.
2. `pass` is in the v0.2 reserved-word list (`FWL_V02_SPEC.md:67`)
   and the Compile-errors table (`FWL_V02_SPEC.md:702`) maps it to a
   distinct error: `error: '<kw>' is not supported in v0.2 Tier 2
   functions`.

So when the user writes `def f(pkt):\n    pass`, the analyser fires the
unsupported-keyword error, not the empty-body error. The edge case's
example does not exhibit the error it cites.

Class B (spec/example contradiction; spec layer; pre-implementation,
the fix is editorial). Downstream Class-A risk: a `.pkt` test author
who copies this example and asserts `compile_error: function 'f' has
empty body...` gets the wrong error string and the test fails.

## Locations

### The example as written

`FWL_V02_SPEC.md:640-643`:

```
*Empty function body.* `def f(pkt):\n    pass` — `pass` is not in
v0.2. An empty body is a compile error: `error: function '<name>'
has empty body; expected at least one statement`.
```

### `pass` is a reserved keyword

`FWL_V02_SPEC.md:67`:

> *Statement keywords:* `def`, `if`, `elif`, `else`, `for`, `while`,
> `pass`, `return`, `default`.

`FWL_V02_SPEC.md:75-76`:

> `for`, `while`, `pass`, `return` are reserved even though v0.2 does
> not admit them in any production — reserving them now means a
> future v0.3 that adds them does not retroactively rename a v0.2
> user's local.

### `pass` has its own dedicated error

`FWL_V02_SPEC.md:702`:

> | `pass`, `return`, `return <expr>` | `error: '<kw>' is not
> supported in v0.2 Tier 2 functions` |

### The grammar's `function_def` requires at least one statement

`FWL_V02_SPEC.md:1042-1043`:

```
function_def  = "def" identifier "(" "pkt" ")" ":" NEWLINE
                INDENT statement { statement } DEDENT ;
```

`statement` is `if_stmt | assign_stmt | action_stmt`, none of which
admits `pass`. So the source `def f(pkt):\n    pass` fails to parse
at the `pass` token (no statement production matches it). The
analyser never reaches the "is the body empty?" question — the
*lexer/parser* rejects `pass` as a malformed-statement token, and the
spec routes that to the unsupported-keyword error, not to an
"empty body" message.

## Two distinct programs, two distinct errors

| Source | What happens | Error string |
|---|---|---|
| `def f(pkt):` followed by no further indented lines | parser hits DEDENT immediately after the `:`-NEWLINE; `INDENT statement { statement } DEDENT` cannot match because there is no INDENT token; *parser* error, not the "empty body" semantic error | (parser-level: "unexpected token DEDENT" or similar) |
| `def f(pkt):\n    pass` (the example as written) | `pass` is a reserved keyword that has no statement production; analyser/parser rejects it | `error: 'pass' is not supported in v0.2 Tier 2 functions` |

Neither path produces the "empty body" error the edge case claims.
For the "empty body" error to ever fire, the analyser would need to
accept `pass` as a no-op (it doesn't, by `:75-76`) and *then* notice
the body has no semantically-significant statements. That two-step
walk is not in the spec.

## Why this matters

The edge case is the only positive demonstration of the
"empty-body-is-an-error" rule. A `.pkt` author copying it gets the
wrong error string. More importantly, the edge case papers over a
question the spec never answers:

> What is the source for an *actually* empty function body?

The grammar's `INDENT statement { statement } DEDENT` requires at
least one statement, so an empty body is a *parser* error. The
"empty body" semantic error in the table is therefore unreachable
under the v0.2 grammar — the parser fires first.

## Proposed fix

Either:

**(a) Drop the empty-body edge case**, since the grammar already
rejects empty bodies. Replace the entry with a pointer to the
grammar:

> *Empty function body.* The grammar's `function_def` requires at
> least one `statement` after the `INDENT`. `def f(pkt):` followed
> by no statement is a parser error.

And drop the "function '<name>' has empty body" row from the
Compile-errors table — it is unreachable.

**(b) Keep the edge case but fix the example.** Replace `pass` with
nothing — i.e. show the source as a bare `def f(pkt):` line followed
by a DEDENT. Acknowledge that this is structurally impossible in the
grammar and that the error fires at parse time:

> *Empty function body.* `def f(pkt):` with no following indented
> statement is a parser error. (`pass` is a reserved keyword, not
> a statement; using `pass` as a body produces the
> `'pass' is not supported in v0.2` error, not "empty body".)

Either fix removes the contradiction. (a) is simpler and matches
the grammar.

## Class

Class B — example contradicts the spec's own reserved-word and
grammar rules. Spec layer.
