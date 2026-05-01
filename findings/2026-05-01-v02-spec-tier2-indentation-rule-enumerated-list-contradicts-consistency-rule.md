---
id: finding/2026-05-01-v02-spec-tier2-indentation-rule-enumerated-list-contradicts-consistency-rule
type: finding
protocol: []
builtins: []
severity: medium
layer: spec
pattern_tags: [spec-ambiguity, lexical, tier2, indentation]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: []
---

# 2026-05-01-v02-spec-tier2-indentation-rule-enumerated-list-contradicts-consistency-rule

## Summary

The Tier 2 indentation rule at `FWL_V02_SPEC.md:553` reads:

> The compiler accepts 2-space, 4-space, or 1-tab indentation as
> long as it is consistent.

The two halves of the sentence say different things:

- *"as long as it is consistent"* — any indentation step that is
  applied uniformly is accepted. Python's lexer behaves this way.
- *"2-space, 4-space, or 1-tab indentation"* — only those three
  step sizes are accepted; 3-space, 5-space, 6-space, 8-space, and
  2-tab indentation are rejected, even if they are uniformly
  applied.

A user who writes a Tier 2 function with consistent 3-space
indentation reads the spec and cannot tell whether their program
should compile. Two conformant compilers — both implementing
"the spec" — can land on different answers, and a `.pkt` test that
exercises a 3-space file lands on a Class-A interpreter-vs-BPF
disagreement at the level of "did this even parse?".

The Compile-errors table at `FWL_V02_SPEC.md:735-736` is also no
help — it lists `error: inconsistent indentation in function
'<name>'` and `error: tabs and spaces mixed in indentation`, but
not `error: only 2-space, 4-space, or 1-tab indentation supported`.
So even an implementor who decides the enumerated list is the
binding rule has no spec text giving a user-visible message for the
3-space case.

Class B (spec ambiguity; lexical layer).

## Side-by-side

### Sentence A — consistency rule

> The level is chosen by the first body line and must be consistent
> for all statements at the same nesting depth (mismatched
> indentation is a compile error). — `:550-551`.

Read on its own, this is the Python rule: pick a step on the first
indented line, then enforce that every block at the same depth
uses it.

### Sentence B — enumerated list

> The compiler accepts 2-space, 4-space, or 1-tab indentation as
> long as it is consistent. — `:552-553`.

Read on its own, this is a narrowing: only three step sizes are
accepted.

The two sentences live one paragraph apart and contradict
unless one is read as subordinate to the other. The spec gives no
hint which.

## Concrete cases the spec doesn't disambiguate

| Source's actual indentation | Sentence A says | Sentence B says |
|---|---|---|
| 2-space, applied consistently | accept | accept |
| 4-space, applied consistently | accept | accept |
| 1-tab,   applied consistently | accept | accept |
| 3-space, applied consistently | accept | reject |
| 5-space, applied consistently | accept | reject |
| 8-space, applied consistently | accept | reject |
| 2-tab,   applied consistently | accept | reject |
| Mix of 2-space at depth 1 and 4-space at depth 2 (each consistent at its depth) | reject by "consistent for all statements at the same nesting depth" | rejected by enumeration *and* reject by "consistent" — but error message differs |

## Why this matters

Two angles:

1. **Implementor decision is unforced.** The grammar's `INDENT` /
   `DEDENT` terminals (`FWL_V02_SPEC.md:1060, 1074, 1079, 1082`) say
   nothing about step size — the convention is that those tokens are
   produced by a Python-style lexer that infers the step from the
   first indented line. A lexer author has to choose between
   sentence A (Python-style, infer-and-enforce) and sentence B
   (whitelist three step sizes). Each choice yields a different set
   of accepted programs.

2. **Corpus and dogfood pin one answer accidentally.** The seven
   Tier 2 examples in the spec (`:743-846`, `:983-1011`) all use
   2-space indentation. The dogfood `.fw` will too. The corpus
   never exercises 3-space, 1-tab, or 4-space, so whichever rule
   the implementor picks goes unverified for years. By the time a
   real user hits the gap (writing 3-space because their editor
   defaults that way) the corpus oracle gives the user no recourse.

   This is the same shape as the Phase-1 rate_limit inversion
   (existing findings
   `2026-04-25-rate-limit-inversion-ssh-brute-force` and
   `-web-server-ddos`): the corpus accidentally only used the
   "right" form, and the spec's ambiguity hid the gap until
   dogfood traffic surfaced it.

## Proposed fix

Pick one resolution and rewrite `:547-553` accordingly. Either:

**Option A — strict enumeration (matches the v0.1 grammar's
single-rule-per-line style).** Rewrite to:

> *Indentation rules:* the function body is indented by exactly
> one level relative to the `def` line. `if`/`elif`/`else` blocks
> introduce one further level of indentation. The accepted step
> sizes are exactly **2 spaces**, **4 spaces**, or **1 tab**; the
> compiler rejects any other step size with `error: indentation
> step must be 2 spaces, 4 spaces, or 1 tab; got <N> <chars>`. The
> chosen step size is fixed by the first indented line of the
> function body and must be applied uniformly at every depth.
> Tabs and spaces may not be mixed within one indentation level.

Add a Compile-errors row:

| Step size other than 2-space, 4-space, or 1-tab | `error: indentation step must be 2 spaces, 4 spaces, or 1 tab; got <N> <chars>` |

**Option B — Python-style consistency (matches the prose's first
half).** Rewrite to:

> *Indentation rules:* the function body is indented by exactly
> one level relative to the `def` line. `if`/`elif`/`else` blocks
> introduce one further level of indentation. The compiler infers
> the indentation step from the first indented line of the function
> body — any positive number of spaces, or any positive number of
> tabs, is accepted. Once chosen, the same step must be applied
> uniformly at every depth. Tabs and spaces may not be mixed within
> one indentation level.

Drop the enumerated-list sentence from `:552-553`.

The Compile-errors rows at `:735-736` are already adequate for
option B.

Either option resolves the contradiction; the choice is
implementor-cost vs user-flexibility. The spec's job is to commit
to one.

## Class

Class B — spec ambiguity at the lexical layer. A single sentence
contradicts itself; two conformant lexers can accept disjoint sets
of programs; the Compile-errors table doesn't supply the message
needed under one of the two readings.
