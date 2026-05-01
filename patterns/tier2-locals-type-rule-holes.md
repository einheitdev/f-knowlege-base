---
id: pattern/tier2-locals-type-rule-holes
type: pattern
protocol: []
builtins: []
severity: high
layer: [spec]
pattern_tags: [tier2, locals, type-rules, operators, undefined-behavior, ordered-comparison, integer-literals]
status: confirmed
created: 2026-05-01
instance_count: 6
instances:
  - finding/2026-05-01-v02-spec-tier2-non-bool-local-as-bare-condition-undefined
  - finding/2026-05-01-v02-spec-tier2-ordered-comparison-on-u16-u32-locals-undefined
  - finding/2026-05-01-v02-spec-tier2-integer-literal-typing-ambiguous-and-first-assignment-undefined
  - finding/2026-05-01-v02-spec-tier2-list-typed-locals-grammar-permits-but-no-type
  - finding/2026-05-01-v02-spec-tier2-scalar-expr-missing-proto-keyword
  - finding/2026-05-01-v02-spec-tier2-proto-local-vs-local-comparison-disallowed-but-no-error
---

# tier2-locals-type-rule-holes

## Description

Tier 2 introduces six local-variable types: `bool`, `u16`, `u32`,
`ipv4`, `ipv6`, `proto`. The grammar admits locals in arbitrary
positions (assignment LHS/RHS, comparison LHS/RHS, bare condition,
inside lists, etc.). The Tier 2 type-rules table at
`FWL_V02_SPEC.md:557-572` enumerates **some** legal combinations
but not all; the compile-error table fills in **some** of the gaps
but not all. The result is a sparse type system whose holes the
spec doesn't acknowledge:

- **Operator × type holes.** Ordered comparisons (`<`, `<=`, `>`,
  `>=`) are restricted in one paragraph to "integer port fields and
  integer literals", which excludes `u16`/`u32` *locals* from
  ordered comparisons by a strict reading. Two paragraphs over, the
  type table treats `u16`/`u32` locals as first-class numeric types
  with no operator-class restriction. The sparse-rule set produces
  two analyser verdicts on `if my_port < 1024`.
- **Position × type holes.** A bare local identifier as a `primary`
  inside a condition (`if my_local:`) is well-defined for `bool` and
  undefined for the other five types. Truthy-coercion, type error,
  silent zero-comparison: each is conformant under "no rule".
- **First-assignment integer-literal typing.** `x = 22` —
  is `x` typed `u16` or `u32`? The spec says "integer literal type
  is determined by context", but a first assignment has no context.
  Two analysers will infer different types; downstream comparisons
  with packet fields then succeed or fail accordingly.
- **List-shaped locals.** The grammar admits `ranges = [10.0.0.0/8,
  192.168.0.0/16]`. The type table has no list-shaped local type.
  No compile error covers it. Implementer must invent a behaviour.
- **Missing RHS productions.** `proto`-typed locals can be assigned
  *from* `pkt.proto` but cannot appear *in* a comparison RHS — the
  grammar's `rvalue` admits no `proto_keyword`. The type table
  treats `proto`-typed locals as comparable to keywords; the grammar
  contradicts.
- **Disallowed operations without a compile-error row.** "A
  comparison between two `proto`-typed locals is not permitted" —
  but the compile-errors table has no row for it. Two implementers
  produce two different error strings (or accept the comparison).

The unifying root cause is that the Tier 2 type system was added by
*augmenting* the v0.1 type rules rather than re-deriving them.
v0.1's type rules were complete because v0.1 had no locals — every
operand was a packet field with a fixed type at a fixed position.
Adding six new local types and admitting locals at arbitrary
positions multiplies the surface by ~36 (six types × six positions)
and the type-rules table only nailed down a subset.

## Check Strategy

1. **Build a (operator × LHS-type × RHS-type) truth table** for
   every legal v0.2 operator and every pair of types from the set
   {literal, packet-field, local-of-each-type}. ~600 cells. For each
   cell, locate the spec rule that says it is well-typed, ill-typed,
   or undefined. Cells with no rule are findings.
2. **Build a (position × type) truth table** for the eight
   syntactic positions where a local can appear (LHS of assignment,
   RHS of assignment, LHS of comparison, RHS of comparison, bare
   condition primary, inside list literal, inside `set_or_range`,
   inside `not`). For each cell, locate the rule.
3. **For each type, locate its first-assignment context-typing
   rule.** v0.2 needs at least: `bool` from a comparison or
   `pkt.tcp.syn`; `u16` from `pkt.src_port`/`pkt.dst_port`; `u32`
   from comparison context; `ipv4` from `pkt.src_ip` or v4 literal;
   `ipv6` from `pkt.src_ip6` or v6 literal; `proto` from
   `pkt.proto`. If any type can be inferred from a literal alone (no
   field context), the rule needs a fallback type.
4. **Run the `fwl-test-agent` synthetic-corpus generator** to
   produce one program per (operator × type-pair) cell, run through
   the analyser, and confirm verdict matches the documented rule.
   Cells with no rule must be added to the type-rules table or
   compile-error table.
5. **Audit the compile-error table for completeness.** Every "X is
   not permitted" sentence in the prose should have a corresponding
   error string in the table. Missing strings are reachable via
   adversarial fuzz.

## Known Instances

- `finding/2026-05-01-v02-spec-tier2-non-bool-local-as-bare-condition-undefined`
  — `if my_u16:` and analogous for `u32`/`ipv4`/`ipv6`/`proto`.
- `finding/2026-05-01-v02-spec-tier2-ordered-comparison-on-u16-u32-locals-undefined`
  — `if my_port < 1024` with `my_port` a `u16` local. Operators
  section forbids; type rules permit.
- `finding/2026-05-01-v02-spec-tier2-integer-literal-typing-ambiguous-and-first-assignment-undefined`
  — `x = 22` has no context to disambiguate `u16` vs `u32`.
- `finding/2026-05-01-v02-spec-tier2-list-typed-locals-grammar-permits-but-no-type`
  — `ranges = [10.0.0.0/8, ...]` is grammar-legal, has no type.
- `finding/2026-05-01-v02-spec-tier2-scalar-expr-missing-proto-keyword`
  — `proto`-typed locals can't appear in a comparison RHS via
  the grammar; type-rules contradict.
- `finding/2026-05-01-v02-spec-tier2-proto-local-vs-local-comparison-disallowed-but-no-error`
  — "two proto-locals cannot be compared" has no error string.

## Where to Look Next

- **`bool` locals on the RHS of comparisons against non-`bool`
  fields.** `dst_port == my_bool` is described as a type error in
  the spec's prose; the grammar admits it; the compile-error table
  doesn't list it. Confirm the prose and add the row.
- **`ipv6` literals as comparison RHS against `pkt.src_ip` (v4
  field).** The grammar's `rvalue` admits `ipv6`; the type rules
  forbid v4-vs-v6 comparison. Audit the compile-error table.
- **CIDR vs literal type compatibility.** `pkt.src_ip == 10.0.0.0/8`
  vs `pkt.src_ip in [10.0.0.0/8]`. The first is a v4 literal
  comparison (10.0.0.0/8 is a CIDR, not an `ipv4` operand); the
  second is well-typed. The type rules need to spell this out.
- **Implicit widening between `u16` and `u32` locals.** The spec
  forbids it; no error row.
- **`bool`-typed expression in non-bool position.** `count <n> if
  my_u16` — does `my_u16` truthy-coerce, error, or silently
  zero-compare? The spec's prose covers the bool case ("`count <n>
  if my_bool` — fine") but not the non-bool case.
- **`proto`-typed local as `pkt.proto in [<list>]` LHS.** Tier 2
  `p = pkt.proto; if p in [tcp, udp]:` — is this well-typed? The
  proto-enum table treats `pkt.proto in [...]` as well-typed; the
  Tier 2 type rules treat `proto`-typed locals as comparable only
  via `==` and `!=`. The grammar admits the surface; the rules
  conflict.
- **Negation operators on each type.** `if not my_bool:` is
  well-defined. `if not my_u16:` — does it truthy-negate? `if not
  (my_ip == 10.0.0.1):` is a comparison-negation; cleanly defined.
  But `if not my_proto:` is in no-man's land.
- **Type narrowing across assignments.** `x = pkt.src_port; x = 22;
  x = pkt.dst_port` — does the second assignment refine `x`'s type
  from `u16` to a narrower literal type? v0.2 says "locals are
  single-assignment" in some paragraphs; if multi-assignment is
  ever permitted, type narrowing rules will be needed.
- **`set_or_range` LHS = local.** `if my_ip in [10.0.0.0/8]:` —
  grammar admits `lvalue = identifier`, and the prose accepts
  locals on the LHS of `in`. Cross-check the geoip-call surface:
  `if my_ip in geoip(RU)` — does the family-binding rule consider
  the local's type? The spec defers to "the LHS field's type"; for
  a local, that's the local's type, not a field's type.
