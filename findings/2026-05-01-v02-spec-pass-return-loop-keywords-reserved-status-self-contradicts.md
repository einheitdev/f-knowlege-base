---
id: finding/2026-05-01-v02-spec-pass-return-loop-keywords-reserved-status-self-contradicts
type: finding
protocol: []
builtins: []
severity: medium
layer: spec
pattern_tags: [spec-inconsistency, lexical, tier2, reserved-words, error-message]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related:
  - finding/2026-05-01-v02-spec-tier2-empty-body-example-uses-pass-which-fires-different-error
  - finding/2026-05-01-v02-spec-tier2-reserved-keyword-set-not-enumerated
---

# 2026-05-01-v02-spec-pass-return-loop-keywords-reserved-status-self-contradicts

## Summary

Three places in `FWL_V02_SPEC.md` disagree on whether `pass`, `return`,
`for`, and `while` are reserved words in v0.2:

1. **Lexical Structure (`:75`)** — explicitly declares these four are
   **not** reserved and are valid as Tier 2 function names, locals,
   and counter names.
2. **Tier 2 Edge cases (`:655` and `:697-701`)** — calls `pass` "a
   reserved word in v0.2 (see Lexical Structure)" and prescribes
   specific keyword-not-supported errors for `for`, `while`, and
   `return`.
3. **Tier 2 Compile errors table (`:723-724`)** — lists rows mapping
   `pass`, `return`, `return <expr>`, `for`, and `while` to the
   dedicated error string `error: '<kw>' is not supported in v0.2
   Tier 2 functions`.

The contradiction makes two questions undecidable:

- **Is `def f(pkt):\n    pass` a parse error or a "not supported"
  error?** If `pass` is an identifier (per `:75`), the parser's
  `assign_stmt = identifier "=" scalar_expr` cannot match (no `=`
  follows), so the diagnostic is a generic "unexpected NEWLINE,
  expected `=`". If `pass` is a reserved keyword (per `:655`,
  `:723`), a dedicated keyword token fires the specific
  `'pass' is not supported in v0.2 Tier 2 functions` error. The
  two paths produce two different user-visible strings.
- **Is `def f(pkt):\n    pass = 1\n    drop` a valid program?** Per
  `:75`, yes — `pass` is a local name of type `u16`, the program
  parses, the analyser emits a "declared but never read" warning, and
  the function drops every packet. Per `:655`, `:723`, no — the parser
  rejects `pass` as an unsupported keyword regardless of context.
  Same for `return = 5`, `for = pkt.dst_port`, `while = pkt.src_ip`.

This is the dual of
`finding/2026-05-01-v02-spec-tier2-empty-body-example-uses-pass-which-fires-different-error`:
that finding's fix turned `pass` *non-reserved* in `:75`, but the Edge
cases and Compile errors tables were not updated to match. The
inversion of the original contradiction is now in the spec.

Class B (spec self-contradiction; spec layer; pre-implementation, fix
is editorial). Downstream Class-A risk: an oracle that follows `:75`
will produce different parse outcomes than an oracle that follows
`:655`/`:723` for any program touching the four keywords. The
implementer cannot pick a side without choosing which paragraph to
believe.

## Locations and exact wording

### Lexical Structure says NOT reserved

`FWL_V02_SPEC.md:75`:

> `for`, `while`, `pass`, `return` are **not** reserved in v0.2 — they
> are valid identifiers (counter names, locals, function names). v0.3
> may introduce loops/`return`, in which case those programs would
> need a one-time rename. v0.2 trades forward-compat tidiness for
> backward-compat with v0.1 (which admits the same identifiers as
> counter names). The three new reservations relative to v0.1 — `def`,
> `elif`, `else` — are unavoidable: Tier 2 grammar treats them as
> syntactic keywords, and the parser cannot accept them as
> identifiers without an LL(*) lookahead the spec doesn't require.

The reserved-words enumeration at `:66-73` does **not** include
`for`, `while`, `pass`, `return`. The closing sentence at `:75` is
explicit that the omission is intentional.

### Tier 2 Edge cases says `pass` IS reserved

`FWL_V02_SPEC.md:655` (the "Empty function body" entry):

> `pass` is *not* a workaround — `pass` is a reserved word in v0.2
> (see Lexical Structure) and writing `def f(pkt):\n    pass` fires
> the reserved-keyword error `error: 'pass' is not supported in v0.2
> Tier 2 functions`, not an empty-body error.

The "see Lexical Structure" cross-reference points at `:64-79`, which
is exactly the section that says the opposite. Following the link
takes the reader to a contradicting passage.

`FWL_V02_SPEC.md:697-701`:

> *Loop keywords.* `for`, `while` → compile error: `error: '<kw>' is
> not supported in v0.2 Tier 2 functions`. (BPF verifier dislikes
> loops; v0.2 sidesteps the question.)
>
> *Bare `return`, `return <expr>`.* Compile error: `error: 'return'
> is not supported in v0.2 Tier 2 functions; use 'allow' or 'drop'`.

For these errors to fire with the exact wording shown, the lexer has
to emit FOR, WHILE, RETURN tokens (so the analyser knows which
keyword to substitute into `<kw>`). That is the lexical behaviour of
a reserved keyword, not an identifier.

### Tier 2 Compile errors table says `pass`/`return`/`for`/`while` are reserved

`FWL_V02_SPEC.md:723-724`:

| Condition | Error |
|---|---|
| `pass`, `return`, `return <expr>` | `error: '<kw>' is not supported in v0.2 Tier 2 functions` |
| `for`, `while` keywords | `error: '<kw>' is not supported in v0.2 Tier 2 functions` |

The table calls them "keywords" and prescribes the same dedicated
error message. Two more rows that contradict `:75`.

## Two distinct programs, two distinct outcomes per interpretation

| Source | If `:75` is right | If `:655`/`:723` are right |
|---|---|---|
| `def f(pkt):\n    pass` | Parser error: `pass` lexes as IDENTIFIER, no `=` follows, no statement production matches, generic "unexpected NEWLINE, expected `=`" diagnostic. | `error: 'pass' is not supported in v0.2 Tier 2 functions` (dedicated). |
| `def f(pkt):\n    pass = 1\n    drop` | Valid: `pass` is a local of type u16. Compiles; "declared but never read" warning. | `error: 'pass' is not supported in v0.2 Tier 2 functions`. |
| `def f(pkt):\n    return = 5\n    drop` | Valid: `return` is a u16 local. | `error: 'return' is not supported in v0.2 Tier 2 functions; use 'allow' or 'drop'`. |
| `def for(pkt):\n    drop` | Valid: function name `for`. | `error: '<kw>' is not supported in v0.2 Tier 2 functions`. |
| `def f(pkt):\n    while = pkt.dst_port\n    drop` (with proto guard hoisted) | Valid: `while` is a u16 local. | `error: '<kw>' is not supported in v0.2 Tier 2 functions`. |
| `count return if pkt.proto == tcp` (Tier 1) | Valid v0.1-shaped program: `return` is a counter name. | Inadmissible: `return` is reserved, error fires (also breaks the strict-superset claim from `:9` and `:46`, since v0.1 admits `count return` as a counter rule). |

The last row is the most damaging. v0.1 explicitly admits any
`[a-z_][a-z0-9_]*` as a counter name, including `return`, `for`,
`while`, `pass`. The strict-superset guarantee at `:9` is qualified
by exactly three new reservations (`def`, `elif`, `else`) — not by
seven. If `:655`/`:723` are taken as authoritative, the carve-out
list is wrong by four entries, and the Phase 1 corpus regression
oracle (`hone regress --corpus ../f-knowlege-base/corpus/`) breaks on
any pre-existing case using one of the four words as a counter name.

## Why this matters for the v0.2 sign-off

Gate 1 of `planning/PHASE_2_PLAN.md` requires zero open `layer: spec`
findings before construct work begins. This contradiction is exactly
the kind of `layer: spec` issue that pollutes downstream verification
oracles:

- The corpus author writing a `.pkt` for `def f(pkt):\n    pass` has
  to assert one of two distinct error strings; the spec doesn't say
  which.
- The analyser implementer has to choose whether the Lark grammar
  emits PASS/RETURN/FOR/WHILE keyword tokens (forcing reservation) or
  treats them as identifiers (forcing parse-error path).
- The hone-hunt agent generating adversarial cases will produce
  programs that hit either interpretation — half its findings will
  be cross-oracle disagreements that resolve to "spec contradicts
  itself", not to a real implementation bug.

## Proposed fix

Pick one interpretation and apply it everywhere. The
prior-finding's fix already chose **non-reserved** at `:75`; the
clean resolution is to propagate that choice into the rest of the
spec.

1. **`:655` ("Empty function body" edge case).** Drop the "`pass` is
   a reserved word in v0.2" claim. Replace with:
   > `def f(pkt):` followed by no indented statement is a parser-level
   > error (the grammar's `INDENT statement { statement } DEDENT`
   > requires at least one statement). `pass` is not reserved (see
   > Lexical Structure), so writing `def f(pkt):\n    pass` fails as
   > `pass = ...` is required for an `assign_stmt` and no `=` follows
   > — i.e. parser error, not a "pass not supported" error.

2. **`:697-701` ("Loop keywords", "Bare `return`").** Drop both
   bullets. `for`, `while`, `return` lex as identifiers per `:75`;
   their misuse produces generic parser errors at the offending
   position, not dedicated keyword-not-supported errors. Either
   delete the bullets or replace with:
   > *`for`, `while`, `return`, `pass` as statement leaders.* These
   > tokens lex as identifiers (see Lexical Structure). Used as a
   > bare statement leader they fail to match any of `if_stmt`,
   > `assign_stmt`, or `action_stmt` (no `=` follows, no production
   > consumes them at statement position) and produce a parser-level
   > "unexpected token" error.

3. **`:723-724` (Compile errors table).** Delete both rows. The
   diagnostics they prescribe are unreachable under `:75`.

4. **Cross-check the strict-superset claim at `:9` and `:46`.** Once
   the four words are confirmed non-reserved, the claim "v0.1
   programs that use one of the three Tier 2 syntactic keywords
   (`def`, `elif`, `else`) as a counter name require a one-time
   rename" stays correct. No further edit needed there — but the
   contradiction at `:655`/`:697-701`/`:723-724` was silently
   broadening the carve-out to seven keywords, which would have
   broken the v0.1 corpus regression oracle.

The alternative — making the four words reserved — would re-open
the original finding (and break v0.1 corpus regression). Sticking
with `:75`'s decision and editing the three downstream sections is
the correct direction.

## Class

Class B (spec self-contradiction). Spec layer.
