---
id: finding/2026-05-01-v02-spec-grammar-operand-overpermissive-for-geoip-call
type: finding
protocol: []
builtins: [geoip]
severity: medium
layer: spec
pattern_tags: [spec-inconsistency, grammar, geoip]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: [finding/2026-05-01-v02-spec-geoip-quoted-string-contradicts-bare-identifier-rule]
---

# 2026-05-01-v02-spec-grammar-operand-overpermissive-for-geoip-call

## Summary

The grammar reference at `FWL_V02_SPEC.md:1011-1027` admits
`geoip_call` as a regular comparison `operand`, while the prose
twice asserts the parser rejects `geoip(...)` everywhere except the
RHS of `in`. The grammar production needs to be split (or the prose
softened to "the analyser rejects" rather than "the parser
rejects").

Class B (spec/grammar inconsistency).

## Locations

### Prose: parser rejects geoip outside RHS of `in`

- `FWL_V02_SPEC.md:305-309`:

  > A `geoip(...)` call may not appear as an argument to another
  > function: `geoip(geoip(RU))` and `f(geoip(RU))` are
  > **syntactically rejected** (the grammar limits the form to RHS
  > of `in` only).
  >
  > A `geoip(...)` call may not appear in any context other than RHS
  > of `in`. `pkt.src_ip == geoip(RU)` is a compile error.

- `FWL_V02_SPEC.md:1033-1035`:

  > `geoip_call` may appear only in the `set_or_range` production
  > (RHS of `in`); the parser rejects it elsewhere with the
  > "`geoip() is only valid as the right-hand side of 'in'`" error.

### Grammar: `geoip_call` is a regular operand

- `FWL_V02_SPEC.md:1015-1016`:

  ```ebnf
  operand       = integer | ipv4 | ipv6 | proto_keyword
                | geoip_call ;
  ```

`operand` is the right-hand side of equality (`==`, `!=`) — by the
v0.1 grammar (`comparison = field op operand`), this production
allows `pkt.src_ip == geoip(RU)`. The grammar therefore *does not*
limit `geoip_call` to the `set_or_range` production.

Identical issue with `proto_keyword` in the same operand
production: `pkt.src_port == tcp` is grammar-legal under this
production but semantically nonsense (proto keywords are only
valid against `pkt.proto`).

## Why this matters

A parser implementing the grammar exactly as written will accept
`pkt.src_ip == geoip(RU)`, then either:

1. Generate a parse-tree the analyser silently rejects as a
   compile error — fine, but contradicts the prose at line
   1033 ("the parser rejects it").
2. Or reject only at the analyser, in which case the error
   *layer* (parse vs analyser) is wrong relative to the spec's
   claim — and an implementor doing test classification can't
   tell whether the rejection should fire pre- or post-analysis.

`set_or_range` already contains `geoip_call` (`FWL_V02_SPEC.md:1020`).
If the intent is "parser-level rejection outside RHS of `in`", the
`operand` production must drop `geoip_call`; the only place
`geoip_call` should appear is `set_or_range`.

## Proposed fix

Edit `FWL_V02_SPEC.md:1015-1016`:

```ebnf
operand       = integer | ipv4 | ipv6 | proto_keyword ;
```

Drop `geoip_call` from `operand`. It already appears in
`set_or_range` (line 1020), which is the only context the prose
allows.

Also: `proto_keyword` is grammar-legal as a generic operand even
though it is only meaningful against `pkt.proto`. Either factor it
out into a `proto_comparison` non-terminal, or document that
field/operand type compatibility is checked by the analyser, not
the parser.
