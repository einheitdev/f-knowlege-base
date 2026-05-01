---
id: finding/2026-05-01-v02-spec-grammar-set-or-range-includes-range-for-ipv6
type: finding
protocol: []
builtins: []
severity: medium
layer: spec
pattern_tags: [spec-inconsistency, grammar, ipv6]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: []
---

# 2026-05-01-v02-spec-grammar-set-or-range-includes-range-for-ipv6

## Summary

The v0.2 grammar reference admits `range` as a generic
`set_or_range` element, while the prose explicitly forbids IPv6
ranges. The grammar does not constrain `range` to IPv4-only
operands, so a parser implementing the grammar will accept
`pkt.src_ip6 in 2001:db8::1..2001:db8::ff` and only reject it in
the analyser. The prose at `FWL_V02_SPEC.md:71-72` and the deferral
list both promise parser-level rejection.

Class B (spec/grammar inconsistency).

## Locations

### Prose: no IPv6 ranges, parser rejects

- `FWL_V02_SPEC.md:71-72`:

  > There is no IPv6 range form (`a..b`) in v0.2 — ranges are
  > deferred to v0.3.

- `FWL_V02_SPEC.md:213-214`:

  | IPv6 range syntax `a..b` | `error: IPv6 ranges not supported in v0.2; use a CIDR or list` |

- `FWL_V02_SPEC.md:956-957` (deferral list):

  > **IPv6 ranges (`a..b`)** — use a CIDR or list.

### Grammar: range is generic, no family qualifier

- `FWL_V02_SPEC.md:1020-1024`:

  ```ebnf
  set_or_range  = list | range | cidr | cidr_list | geoip_call ;
  list          = "[" operand { "," operand } "]" ;
  cidr          = ipv4 "/" integer
                | ipv6 "/" integer ;
  cidr_list     = "[" cidr { "," cidr } "]" ;
  ```

The grammar inherits `range` from v0.1 (where it is `integer ".."
integer` for ports) but does not restate it here. v0.1's
`range` is port-only, but the v0.2 grammar adds `cidr` with both
IPv4 and IPv6 alternatives without similarly clarifying that
`range` stays port-only. A reader without v0.1 in front of them
cannot tell from the v0.2 grammar alone whether `2001:db8::1..2001:db8::ff`
parses.

The compile-error message at line 213 says "IPv6 ranges not
supported in v0.2" — strong implication that the surface
*accepts* the syntax and rejects semantically. But that is
inconsistent with `FWL_V02_SPEC.md:71-72` which describes IPv6
ranges as "no … form" — i.e. not a syntactic surface at all.

## Why this matters

If the analyser rejects "IPv6 range" with the error at line 213,
then the parser must accept *something* that gets passed to the
analyser. What is the parsed shape — `range_of(ipv6, ipv6)`? The
grammar doesn't define it, so the analyser cannot pattern-match on
it.

Compare to the `proto_keyword` issue (separate finding): this is
the same pattern — a syntactic surface the prose claims is rejected
at parse time but the grammar admits.

## Proposed fix

Either:

- **Resolution A: parser rejects.** Add an explicit `range = port
  ".." port ;` production in the v0.2 grammar block (or import the
  v0.1 `range` rule and restate `port = integer`). Drop the
  compile-error row at line 213-214 in favour of "syntax error:
  unexpected `..` after IPv6 literal".
- **Resolution B: parser accepts, analyser rejects.** Add an
  `ipv6_range = ipv6 ".." ipv6 ;` production to the grammar and
  keep the compile-error row at line 213-214. Both treatments are
  defensible but the spec must pick one.
