---
id: pattern/example-contradicts-rule
type: pattern
protocol: [tcp, udp, icmp, icmp6]
builtins: [geoip, rate_limit]
severity: high
layer: [spec]
pattern_tags: [example-vs-rule, spec-inconsistency, dogfood, worked-example, edge-case]
status: confirmed
created: 2026-05-01
instance_count: 9
instances:
  - finding/2026-05-01-v02-spec-rfc5952-rule4-rejects-bare-zero-and-loopback-canonical-examples
  - finding/2026-05-01-v02-spec-tier2-example1-rate-limit-per-src-ip-violates-dominator-rule
  - finding/2026-05-01-v02-spec-tier2-example5-violates-grammar-and-dominator-rule
  - finding/2026-05-01-v02-spec-tier2-stmts-after-terminal-example-violates-proto-guard
  - finding/2026-05-01-v02-spec-tier2-empty-body-example-uses-pass-which-fires-different-error
  - finding/2026-05-01-v02-spec-geoip-manifest-example-shows-unreachable-two-family-call-site
  - finding/2026-05-01-v02-spec-geoip-quoted-string-contradicts-bare-identifier-rule
  - finding/2026-05-01-v02-spec-tier2-locals-not-in-condition-grammar
  - finding/2026-04-25-rate-limit-spec-bullets-vs-example-still-inconsistent
---

# example-contradicts-rule

## Description

A worked example, edge-case snippet, or table illustration in the
spec violates the very rule it is supposed to demonstrate (or that
is stated elsewhere in the same section). The example may:

- Use syntax the spec's grammar does not admit (single-line `if
  cond: stmt`; locals on either side of a comparison; `def f(pkt):
  pass`).
- Demonstrate a positive case for one rule while silently violating
  another (the dominator rule, the canonicalization rule, the
  reserved-word rule).
- Show a JSON / wire-format shape the spec's other paragraphs prove
  unreachable (geoip manifest with two family entries under one
  `call_index`).
- Use a code/identifier form (quoted country code, expanded-zero
  IPv6) the spec's own lexical/grammar section forbids.

The downstream effect is **bimodal**: a strict implementer reading
the rule rejects the example and is forced to either edit the
example (defeating its corpus-locking purpose) or mark the case
`expected.compiles: false` (contradicting the spec's narrative). A
liberal implementer accepts the example and quietly diverges from
the rule on every other program of the same shape.

The two implementer choices land on different accept/reject
verdicts for the same program. Hone's interpreter-vs-BPF oracle
disagrees with no spec text to point at — the bug is one paragraph
above or below the rule the analyser is consulting.

This is the densest pattern in the Phase 2 hunt: 9 confirmed
instances across IPv6 canonicalization, Tier 2 grammar, Tier 2
dominator/protocol-guard, geoip manifest, and rate-limit
off-by-one. Many examples violate **multiple** rules at once
(Example #5 fails both the grammar and the dominator rule; the
empty-body example fails the reserved-word rule and the empty-body
rule).

## Check Strategy

1. **Lift every code block in `FWL_V02_SPEC.md` and `PKT_V02_SPEC.md`
   into a synthetic `.pkt` test with `expected.compiles: true`.**
   Any case that fails to compile is either a spec-rule bug or an
   example bug. Particularly target:
   - The numbered "worked examples" sections.
   - The "Edge cases" subsection of each construct.
   - JSON snippets in geoip / bundle-manifest sections.
   - Inline ` ```python ` blocks in narrative prose.
2. **For every spec rule of the form "X is a compile error" (or
   similar), grep the spec for code that exhibits X and confirm the
   example is annotated as a failing case, not a passing one.**
   Example: the spec lists `pass` as reserved → grep for `pass` in
   any positive code block.
3. **Cross-check tables against examples.** When a table enumerates
   "valid forms" or "canonical forms", confirm every entry round-trips
   under the rule the same section states. (Rule 4 of IPv6
   canonicalization should not have `::` and `::1` listed two
   paragraphs up as canonical.)
4. **Lift JSON wire-format examples and confirm they are
   reachable** — i.e. some legal source program produces them.
   Manifest examples in particular should be derived, not
   hand-written.
5. **For every "intent" comment in a worked program** (lines like
   `# Rate-limit new SSH SYNs per source`), trace through the spec's
   runtime semantics on a packet that exercises the intent. Mismatch
   is a Class C user-rule-bug-in-the-spec — see pattern
   `user-rule-intent-mismatch`.

## Known Instances

- `finding/2026-05-01-v02-spec-rfc5952-rule4-rejects-bare-zero-and-loopback-canonical-examples`
  — canonicalization rule 4 (added to fix the IPv4-mapped ambiguity)
  rejects `::` and `::1`, while the same section's tables list both
  as canonical-form examples and the worked examples use them.
- `finding/2026-05-01-v02-spec-tier2-example1-rate-limit-per-src-ip-violates-dominator-rule`
  — Tier 2 worked example #1 (the SSH brute-force flagship) puts
  `rate_limit(per=src_ip)` inside a `pkt.proto == tcp` guard. The
  spec's own dominator rule excludes `pkt.proto == tcp` from the
  v4-establishing guard list.
- `finding/2026-05-01-v02-spec-tier2-example5-violates-grammar-and-dominator-rule`
  — stack-budget stress-test uses single-line `if cond: stmt` form
  six times (grammar admits no such production) and reads
  `pkt.src_port`/`pkt.dst_port` at the top of the function with no
  proto guard (dominator rule rejects).
- `finding/2026-05-01-v02-spec-tier2-stmts-after-terminal-example-violates-proto-guard`
  — "Statements after a terminal" edge case offers `if pkt.tcp.syn:
  drop\nallow` as a valid program; `if pkt.tcp.syn:` violates the
  inherited v0.1 protocol-guard rule.
- `finding/2026-05-01-v02-spec-tier2-empty-body-example-uses-pass-which-fires-different-error`
  — the "empty function body" edge case uses `def f(pkt): pass` to
  demonstrate the empty-body error, but `pass` is itself reserved in
  v0.2 and fires the unsupported-keyword error first.
- `finding/2026-05-01-v02-spec-geoip-manifest-example-shows-unreachable-two-family-call-site`
  — geoip manifest example shows a single `call_index` with both
  v4 and v6 entries, a shape the language cannot generate (each
  textual `geoip(...)` is its own call site).
- `finding/2026-05-01-v02-spec-geoip-quoted-string-contradicts-bare-identifier-rule`
  — country codes are bare identifiers per lexical rule, but a
  worked example invokes `geoip("RU")` with a quoted code.
- `finding/2026-05-01-v02-spec-tier2-locals-not-in-condition-grammar`
  — every Tier 2 example uses locals inside `if`/`elif` conditions
  (`if internal:`, `if a == 10.0.0.1:`); the v0.2 grammar inherits
  `condition` from v0.1 unchanged, which admits no `identifier` in
  any condition position.
- `finding/2026-04-25-rate-limit-spec-bullets-vs-example-still-inconsistent`
  — even after the rate-limit-inversion fix, the formal bullets and
  the example explanation in `FWL_V01_SPEC.md` remain off-by-one
  (post-increment ≥ N vs. (N+1)-th packet).

## Where to Look Next

This pattern has high yield because the spec's prose-vs-rule density
is high. Surfaces still untouched:

- **Every Tier 2 worked example (#1 through #5) needs a positive
  `.pkt` test with `expected.compiles: true` checked into the
  corpus**. Three are confirmed broken. The remaining two (#2 and
  #4) have not been mechanically verified — if they still parse,
  they have not been re-checked since the dominator rule was added.
- **Tier 2 "Edge cases" subsection (`FWL_V02_SPEC.md:626-700`).** A
  dozen narrated snippets, several already shown to be broken. Each
  remaining edge case should be lifted into a synthetic `.pkt`.
- **PKT_V02_SPEC.md worked examples for new builders** (`tcp6`,
  `udp6`, `icmp6`, `geoip_data:` blocks). The geoip manifest finding
  was a PKT-spec finding; the same audit hasn't been run on the
  rest of `PKT_V02_SPEC.md`.
- **Tables that enumerate "valid forms"**: the proto-keyword table
  at `FWL_V02_SPEC.md:163-167`, the `set_or_range` table, the
  rate-limit `per=` field table. Each entry should be reachable from
  the grammar and consistent with every section's prose.
- **The dogfood example (`fwl/examples/dogfood_v02.fw`).** Already
  found one bug (the v6-bypass rate-limit user-intent finding). The
  example has v4-and-v6 mirror branches, multiple guard styles, and
  a count statement — every one of those is a candidate for an
  example-vs-rule contradiction that hasn't been checked against
  every spec section.
- **Compile-error tables in every section.** Each row is an
  implicit promise that the analyser fires the listed string for the
  listed input shape. Cross-reference with pattern
  `grammar-vs-body-text-drift`: many of these strings are
  unreachable because the parser preempts them.
