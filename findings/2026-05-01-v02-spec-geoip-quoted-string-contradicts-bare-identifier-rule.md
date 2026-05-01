---
id: finding/2026-05-01-v02-spec-geoip-quoted-string-contradicts-bare-identifier-rule
type: finding
protocol: []
builtins: [geoip]
severity: low
layer: spec
pattern_tags: [spec-inconsistency, geoip, example-vs-rule]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: []
---

# 2026-05-01-v02-spec-geoip-quoted-string-contradicts-bare-identifier-rule

## Summary

`docs/FWL_V02_SPEC.md` declares country codes are bare identifiers
(not strings) in two places, but a worked example inside the same
section invokes `geoip(...)` with a *quoted* code. The example
contradicts the lexical rule it lives under.

Class B (analyzer/spec contradiction).

## Locations

- **Bare-identifier rule** — `FWL_V02_SPEC.md:265-281`:

  > `<cc>` is an unquoted (bare) two-letter ISO 3166-1 alpha-2 country
  > code in **uppercase** (e.g. `RU`, `CN`, `US`, `DE`).
  >
  > Codes are bare identifiers, not strings. v0.1 has no string
  > literals, so this stays consistent with the language's lexical
  > rules — country codes are tokens of the form `[A-Z][A-Z]` and
  > only valid inside a `geoip(...)` argument list.

- **Grammar** — `FWL_V02_SPEC.md:1026-1027`:

  > `geoip_call    = "geoip" "(" cc_code { "," cc_code } ")" ;`
  > `cc_code       = upper letter , upper letter ;       (* exactly two *)`

  No string production, no quotes.

- **Contradicting worked example** — `FWL_V02_SPEC.md:292-296`:

  > One `geoip(...)` call backs **both** an IPv4 trie and an IPv6
  > trie. The daemon populates whichever tries are referenced by at
  > least one `in` site in the program. If `pkt.src_ip in
  > geoip("RU")` is the only use, the v6 trie for that call is empty
  > and not loaded; the bundle's `manifest.json` records this.

  `geoip("RU")` uses a *quoted* string. Per the lexical rule, this
  is a syntax error.

## Why this matters

Spec readers seed expectations from worked examples more than from
bullet text. An implementor who copies `geoip("RU")` into the
parser/analyzer test suite gets a different parse path than the
grammar requires: either the implementation must accept quoted
strings (contradicting the rule and silently leaking string
literals into v0.2), or this example fails to compile and the spec
is wrong by example.

Compile-error table at `FWL_V02_SPEC.md:408-418` does not list
"quoted string instead of bare code" as an error class either, so
the diagnostic the user sees is undefined.

## Proposed fix

Change `geoip("RU")` to `geoip(RU)` at `FWL_V02_SPEC.md:295`. Add
a row to the compile-error table:

| `geoip("RU")` (string instead of bare code) | `error: geoip codes are bare identifiers, not strings; got '"RU"'` |
