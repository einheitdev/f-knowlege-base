---
id: finding/2026-05-01-v02-spec-grammar-local-field-undefined-nonterminal
type: finding
protocol: []
builtins: []
severity: low
layer: spec
pattern_tags: [spec-typo, grammar, tier2]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: []
---

# 2026-05-01-v02-spec-grammar-local-field-undefined-nonterminal

## Summary

The grammar reference's expression production references a
non-terminal `local_field` that is never defined anywhere in the
spec. `local_field` appears to be a typo for `local | field` (or
similar). Without a definition, an implementor cannot tell whether
locals and packet fields are read through one production or two.

Class B (spec typo / undefined non-terminal).

## Location

`FWL_V02_SPEC.md:1007-1010`:

```ebnf
(* condition: same as v0.1; see FWL_V01_SPEC.md.                 *)
(* expression: condition | local_field | int | ipv4 | ipv6 |     *)
(*   ipv4_cidr | ipv6_cidr | list                                *)
```

`local_field` does not appear elsewhere in the v0.2 grammar block,
and the v0.1 grammar (which v0.2 imports for `condition`) does not
define it either. The closest defined symbols are `value_field`
(`FWL_V02_SPEC.md:1011-1013`) — packet fields only — and `identifier`
(`assign_stmt`'s LHS at `FWL_V02_SPEC.md:994`) — locals only.

## Why this matters

Tier 2 introduces the first context where a *local* can be read in
an expression position. The spec implies this is supported — the
example at `FWL_V02_SPEC.md:735-744` reads `internal` after
assigning it — but the grammar production for the expression that
permits it is undefined.

An implementor of the v0.2 parser must guess one of:

| Guess | Consequence |
|---|---|
| `local_field = identifier ;` (read-only) | Locals readable; can't read packet fields except via existing v0.1 productions inside `condition`. The example at line 735-744 then doesn't parse — `pkt.src_ip` is not in the `expression` alternatives. |
| `local_field = identifier | value_field ;` | Locals and `pkt.<field>` both readable. Probably the intent, since the example assigns `pkt.src_ip in [...]` to a local, which requires an `in` expression on the RHS — meaning `expression` must subsume `condition` (which it does, listed first). |
| `local_field = identifier ;` AND `expression` grows packet-field readers via `condition` only | `internal = pkt.src_ip in [...]` parses (the RHS is a `condition`). `port = pkt.dst_port` does **not** parse (an integer comparison is not a `condition`; a bare field read is not defined). |

The third reading is the most surface-restrictive but is fragile:
adding `port = pkt.dst_port` (read a `u16` into a `u16` local) needs
a separate `field_read` production — which the spec hasn't
provided.

## Proposed fix

Replace the comment on `FWL_V02_SPEC.md:1007-1010` with explicit
productions:

```ebnf
expression    = condition
              | identifier
              | value_field
              | integer | ipv4 | ipv6
              | cidr | list
              | cidr_list
              | geoip_call ;            (* only inside an `in` RHS *)

(* `condition` is reused from v0.1; see FWL_V01_SPEC.md. *)
```

Then resolve the `geoip_call`-in-`operand` issue separately
(see `finding/2026-05-01-v02-spec-grammar-operand-overpermissive-for-geoip-call`).
