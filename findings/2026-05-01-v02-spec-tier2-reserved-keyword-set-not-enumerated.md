---
id: finding/2026-05-01-v02-spec-tier2-reserved-keyword-set-not-enumerated
type: finding
protocol: []
builtins: [geoip, rate_limit]
severity: medium
layer: spec
pattern_tags: [spec-ambiguity, lexical, tier2, identifiers, reserved-words]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: []
---

# 2026-05-01-v02-spec-tier2-reserved-keyword-set-not-enumerated

## Summary

v0.2 introduces the first user-definable identifiers — Tier 2
function names and Tier 2 locals — but the spec never enumerates
the reserved-word set. The grammar uses literal terminals
(`"if"`, `"def"`, `"in"`, `"per"`, `"geoip"`, `"tcp"`, etc.); the
identifier production matches `[a-z_][a-z0-9_]*` (inherited from
v0.1 `FWL_V01_SPEC.md:39`); the only explicit reservation in the
v0.2 spec is `pkt` (`FWL_V02_SPEC.md:564, 703`). Whether `def`,
`if`, `geoip`, `count`, `tcp`, etc. may be used as a local name or
a function name is left to lexer guess-work.

Two conformant lexers — both implementing the v0.2 spec as written
— can disagree on whether `def def(pkt): allow` is a parse error
(stricter lexer: `def` is reserved) or a parse error for a *different*
reason (looser lexer: identifier-named-`def` collides with the keyword
`def` in the parser's lookahead). The same goes for `count` as a
counter name (`count count`), `geoip` as a local
(`geoip = pkt.src_ip in [...]`), `tcp` as a local
(`tcp = pkt.proto == tcp`), and so on.

Class B (analyzer/spec underspecification; spec layer).

## What the spec actually says

### Identifier production — inherited from v0.1

`FWL_V01_SPEC.md:39`:

> Identifiers match `[a-z_][a-z0-9_]*` — lowercase letters, digits
> (not in first position), and underscores. They appear as: built-in
> names (`rate_limit`, `tcp`, `udp`, `icmp`), field accessor segments
> (`pkt`, `proto`, `src_ip`, etc.), interface names in `@xdp(...)`,
> and counter names in `count <n>`. v0.1 has no user-defined
> identifiers in the sense of variables, function definitions, or
> labels.

v0.1 had no user-defined names at all, so the question never came
up. v0.2 adds two user-name-bearing surfaces:

- Tier 2 function names (`FWL_V02_SPEC.md:526-527`): "The function
  name is a bare identifier (`[a-z_][a-z0-9_]*`)."
- Tier 2 locals (`FWL_V02_SPEC.md:1045`): `assign_stmt = identifier
  "=" scalar_expr NEWLINE ;`.

### What's explicitly reserved

`FWL_V02_SPEC.md:564`:

> `pkt` is reserved and may not be used as a local name.

`FWL_V02_SPEC.md:565-568`:

> Action keywords (`allow`, `drop`, `log`) and counter names (after
> `count <n>`) live in disjoint name spaces and do not collide with
> locals. A counter named the same as a local is permitted but
> discouraged; the compiler emits a stylistic warning.

So:
- `pkt` is reserved (ban as local name).
- `allow`, `drop`, `log` are in a "disjoint name space" (so a
  local named `allow` is implicitly *permitted*?).
- `count <n>` — the `<n>` is a counter name in its own namespace.

### What's silent

Every other lowercase keyword the grammar uses as a literal
terminal: `def`, `if`, `elif`, `else`, `not`, `and`, `or`, `in`,
`per`, `count`, `default`, `limited`, `by`, `tcp`, `udp`, `icmp`,
`icmp6`, `geoip`, `rate_limit`, `src_ip`, `dst_ip`, `src_ip6`,
`dst_ip6`, `src_port`, `dst_port`, `proto`, `syn`, `ack`. None of
these appear in any "reserved" enumeration.

Existing prior-art miss
(`misses/2026-04-25-analyzer-and-emitter-edge-cases.md:61-68`)
flagged this for v0.1's `count tcp` case: "spec is silent on whether
keywords can be counter names — Class B candidate but not surfacing."
v0.2 *broadens* the surface (now we also have function names and
locals), so the v0.1 silence becomes a v0.2 multi-surface hole.

### Concrete examples the spec doesn't decide

| Program | Spec says | Two-implementation outcome |
|---|---|---|
| `def def(pkt): allow` | (silent) | One lexer rejects "`def` is the keyword"; another accepts and the parser blows up at the next `def`. |
| `def f(pkt): if = pkt.tcp.syn; if if: drop` | (silent) | `if` as a local — same ambiguity. |
| `def f(pkt): geoip = pkt.src_ip in [10.0.0.0/8]; if geoip: drop` | (silent — `geoip` is grammar-level only inside `geoip(...)`) | One lexer reserves `geoip` (since `geoip_call = "geoip" "(" ...)`); another permits it as an identifier and only refuses when followed by `(`. |
| `def f(pkt): count tcp` | "stylistic warning" if the local namespace overlaps; spec doesn't say if `tcp` (proto keyword) collides with the counter-name namespace. | Lexer A: `tcp` is a `proto_keyword` token, parser fails at `count <proto_keyword>`. Lexer B: counter names live in their own namespace per line 567-568, accept. (See miss `2026-04-25-analyzer-and-emitter-edge-cases:65`.) |
| `def f(pkt): tcp = pkt.proto; if tcp == tcp: drop` | (silent) | One lexer rejects the assignment (`tcp` is a `proto_keyword`); another accepts both occurrences and the analyser tries to type-check. |
| `def f(pkt): pkt = 1` | "`pkt` is reserved" → error | (the only case the spec is unambiguous about) |
| `def pkt(pkt): allow` | "`pkt` is reserved and may not be used as a local name." But the function name? Spec says only "a bare identifier". | One lexer rejects the function name; another accepts the function name and rejects only the local. |

### Why this is structurally a spec issue, not a parser issue

Two valid interpretations of the v0.2 spec produce different parse
behaviour. The spec must pick one. Either:

- **(a) All grammar terminals that look like identifiers are
  reserved**, including `def`, `if`, `elif`, `else`, `not`, `and`,
  `or`, `in`, `per`, `count`, `default`, `limited`, `by`, `tcp`,
  `udp`, `icmp`, `icmp6`, `geoip`, `rate_limit`, `pkt`, `proto`,
  `syn`, `ack`, `src_ip`, `dst_ip`, `src_ip6`, `dst_ip6`,
  `src_port`, `dst_port`. Plus the field accessor segments. None
  may be used as a Tier 2 function name, local, or counter name.

- **(b) Grammar terminals are tokenised contextually**: a
  lookahead-driven lexer recognises `geoip` as the `geoip_call`
  start only when followed by `(`; otherwise it's an `identifier`.
  Same for every other keyword. This is what most Pythonic
  parsers do, but it's a strong claim about the parser
  architecture.

- **(c) An explicit subset is reserved**: enumerate the small set
  the spec actually wants ban (e.g. `pkt`, `def`, `if`, `elif`,
  `else`, `not`, `and`, `or`, `in`, `default`, `for`, `while`,
  `pass`, `return`) and let the rest (`tcp`, `geoip`,
  `rate_limit`, `count`, ...) be valid identifiers in non-keyword
  positions.

(a) is the simplest to implement and verify. (b) is the most
permissive but adds parser complexity. (c) is what most modern
language specs do (Python, Go, Rust all enumerate a "keywords"
list).

## Why this matters

Two oracles wrote independent parsers. If oracle A picks (a) and
oracle B picks (b), every `count tcp`, `geoip = ...`, `tcp =
pkt.proto`, `def def(pkt):` becomes a cross-oracle disagreement —
visible at corpus time as a parse-error vs. parse-success drift.

The miss already flagged it once for v0.1 (`count tcp`) and
explicitly noted it as a "Class B candidate but not surfacing".
v0.2 introduces three new identifier surfaces (function names,
locals, locals' use in counter-vs-local-name overlap) and amplifies
the surface area. The Tier 2 dogfood example
(`FWL_V02_SPEC.md:953-986`) carefully avoids any reserved-word-shaped
identifier — but the corpus author writing fuzz cases will not.

## Proposed fix

Add a "Lexical Structure → Reserved words" subsection at the top of
the v0.2 spec (mirroring v0.1's "Lexical Structure" section). Pick
(c) for clarity:

> **Reserved words.** The following identifiers are reserved in v0.2
> and may not appear as Tier 2 function names, Tier 2 local names,
> or counter names. Lexically they tokenise as their respective
> keyword classes regardless of context.
>
> *Statement keywords:* `def`, `if`, `elif`, `else`, `for`, `while`,
> `pass`, `return`, `default`.
>
> *Boolean operators:* `not`, `and`, `or`.
>
> *Comparison/membership operators (word-form):* `in`.
>
> *Action keywords:* `allow`, `drop`, `log`, `count`.
>
> *Modifier keywords:* `limited`, `by`, `per`.
>
> *Built-in function names:* `rate_limit`, `geoip`.
>
> *Field-root reserved name:* `pkt`. (May not be used as a local or
> function name.)
>
> *Proto keywords:* `tcp`, `udp`, `icmp`, `icmp6`. (Reserved as
> tokens; may not appear as identifiers.)
>
> Field-segment names that follow `pkt.` (`proto`, `src_ip`,
> `dst_ip`, `src_ip6`, `dst_ip6`, `src_port`, `dst_port`, `tcp`,
> `syn`, `ack`) are recognised in field-accessor position only and
> are not globally reserved (a Tier 2 local named `proto` is
> permitted).
>
> Country codes inside `geoip(...)` are uppercase (`[A-Z][A-Z]`)
> and do not collide with the lowercase-only identifier production.

Then update the existing reservation prose at `FWL_V02_SPEC.md:564`
to point at the new section ("see Lexical Structure → Reserved
words for the full list").

## Class

Class B (lexical underspecification). Spec-layer.
