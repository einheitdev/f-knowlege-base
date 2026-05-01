---
id: finding/2026-05-01-v02-spec-grammar-cc-code-upper-letter-with-space-illformed
type: finding
protocol: []
builtins: [geoip]
severity: low
layer: spec
pattern_tags: [spec-typo, grammar, geoip, ebnf]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: [finding/2026-05-01-v02-spec-grammar-local-field-undefined-nonterminal]
---

# 2026-05-01-v02-spec-grammar-cc-code-upper-letter-with-space-illformed

## Summary

The EBNF grammar block at `FWL_V02_SPEC.md:1187` and `:1190` defines
the `cc_code` non-terminal with a non-terminal symbol that contains a
space — `upper letter`. EBNF non-terminals are single identifiers
(no spaces); a space between two tokens is **concatenation**, not a
single name. So the production literally reads as four
non-terminals (`upper`, `letter`, `upper`, `letter`), and the
"definition" line `upper letter = "A" | … | "Z" ;` is itself
ill-formed — there is no `upper letter` non-terminal that EBNF can
parse, only an `upper` followed by a `letter`.

The intent is obviously a single non-terminal `upper_letter` (or
similar) standing for one ASCII uppercase character, with `cc_code`
being two of them. The typo doesn't change the spec's English
intent, but it's the kind of thing a parser-generator front-end
will reject outright.

Class B (spec/grammar typo).

## Locations

`FWL_V02_SPEC.md:1185-1191`:

```ebnf
geoip_call    = "geoip" "(" cc_code { "," cc_code } ")" ;
cc_code       = upper letter , upper letter ;       (* exactly two *)

ipv6          = (* RFC 5952 canonical form *) ;
upper letter  = "A" | "B" | ... | "Z" ;
```

Two issues in one block:

1. **Line 1187** — `upper letter , upper letter`. In ISO 14977
   EBNF, the comma is the concatenation operator and identifiers
   cannot contain spaces. The expression therefore parses as the
   four-term concatenation `upper, letter, upper, letter`, which
   makes `cc_code` a sequence of four non-terminals (none of which
   are defined as singletons either). Writing `cc_code` as
   "exactly two of `upper letter`" requires the symbol be one
   identifier — `upper_letter` is the obvious spelling and matches
   v0.1's snake_case convention (`proto_keyword`, `default_rule`,
   `terminal_action`, etc.; see `FWL_V01_SPEC.md:551-593`).

2. **Line 1190** — `upper letter = …` is the definition side of the
   same typo. EBNF's left-hand side is one identifier; "upper
   letter" with a space is not a valid LHS.

The v0.1 grammar consistently uses snake_case for non-terminals
(`hook_decl`, `default_rule`, `terminal_action`, `enum_field`,
`bool_field`, `proto_keyword`, etc.). v0.2 follows the same
convention everywhere except this block.

## Why this matters

A parser-generator (Lark, ANTLR, hand-rolled-with-validation, etc.)
fed the v0.2 grammar block as written either:

- Silently treats `upper letter` as two non-terminals, fails to
  resolve them (neither is defined), and emits a generic "undefined
  non-terminal" error pointing at the `cc_code` line. The error
  doesn't say what the user meant; the user has to read the prose
  to figure out it's a country code.

- Or accepts the spec as-is and parses every two-letter country code
  as four tokens, requiring the lexer to produce four `upper` /
  `letter` tokens per code. None of v0.2's lexical rules describe
  `upper` or `letter` as terminals.

Either way, an implementor who treats the EBNF block as
authoritative cannot reproduce the spec's intent without reading
the prose at `:60-63` ("Country-code tokens lexically tokenize as
`[A-Z][A-Z]`"). The grammar block is supposed to be the
parser-implementation contract; if it can't be implemented, it's
not serving that role.

This is the same shape as
`finding/2026-05-01-v02-spec-grammar-local-field-undefined-nonterminal`
(non-terminal with no production) but inverted — here the
non-terminal *has* a production, but the symbol name is illegal.

## Proposed fix

Change the symbol from `upper letter` (two tokens) to
`upper_letter` (one token), then anchor the production to the
prose:

```ebnf
geoip_call    = "geoip" "(" cc_code { "," cc_code } ")" ;
cc_code       = upper_letter , upper_letter ;       (* exactly two *)

ipv6          = (* RFC 5952 canonical form *) ;
upper_letter  = "A" | "B" | "C" | "D" | "E" | "F" | "G" | "H"
              | "I" | "J" | "K" | "L" | "M" | "N" | "O" | "P"
              | "Q" | "R" | "S" | "T" | "U" | "V" | "W" | "X"
              | "Y" | "Z" ;
```

(or replace the alternation with a comment `(* one ASCII A..Z *)`,
matching the `ipv6 = (* RFC 5952 canonical form *) ;` style used
two lines above).

This puts one identifier on each LHS, matches v0.1's snake_case
convention, and lets a parser-generator front-end consume the
production without hand-fixing.

## Class

Class B (spec-layer typo). No runtime divergence; the impact is
that the EBNF block as written is not implementable verbatim.
