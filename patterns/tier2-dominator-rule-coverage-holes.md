---
id: pattern/tier2-dominator-rule-coverage-holes
type: pattern
protocol: [tcp, udp, icmp, icmp6]
builtins: [rate_limit]
severity: high
layer: [spec]
pattern_tags: [tier2, dominator-rule, protocol-guards, statement-position-read, ipv6, cross-family, soundness, undefined-behavior]
status: confirmed
created: 2026-05-01
instance_count: 13
instances:
  - finding/2026-05-01-v02-spec-tier2-pkt-proto-no-dominator-rule
  - finding/2026-05-01-v02-spec-tier2-stmt-level-pkt-read-on-wrong-protocol
  - finding/2026-05-01-v02-spec-tier2-stmt-level-ipv6-l3-read-undefined-value
  - finding/2026-05-01-v02-spec-tier2-rate-limit-implicit-pkt-read-not-dominator-checked
  - finding/2026-05-01-v02-spec-tier2-nested-if-condition-reads-not-covered-by-dominator-rule
  - finding/2026-05-01-v02-spec-tier2-dominator-rule-negation-and-else-not-addressed
  - finding/2026-05-01-v02-spec-tier2-condition-on-rhs-of-assign-dominator-vs-shortcircuit-conflict
  - finding/2026-05-01-v02-spec-tier2-icmp6-establishes-v6-guard-but-not-l3-guard
  - finding/2026-05-01-v02-spec-tier2-pkt-proto-icmp-as-v4-guard-vs-cross-family-icmp
  - finding/2026-05-01-v02-spec-tier2-pkt-proto-in-list-with-icmp6-vs-disjunction-rule
  - finding/2026-05-01-v02-spec-tier2-stmts-after-terminal-example-violates-proto-guard
  - finding/2026-05-01-v02-spec-tier2-example1-rate-limit-per-src-ip-violates-dominator-rule
  - finding/2026-05-01-v02-spec-tier2-example5-violates-grammar-and-dominator-rule
---

# tier2-dominator-rule-coverage-holes

## Description

v0.2 introduces Tier 2 programmable functions, which require a new
control-flow dominator analysis to keep statement-position reads of
protocol-specific `pkt` fields sound. The dominator rule was added
to fix the original "statement-level pkt read on the wrong protocol"
hole, but its enumeration is incomplete:

- **Some surfaces aren't enumerated.** The rule covers
  `pkt.src_port`/`pkt.dst_port`, `pkt.tcp.syn`/`pkt.tcp.ack`,
  `pkt.src_ip`/`pkt.dst_ip`, `pkt.src_ip6`/`pkt.dst_ip6`. It does
  **not** cover `pkt.proto` itself (an L3-derived field that needs
  L3 parse to succeed), the implicit `pkt.<per>` read inside
  `rate_limit(per=<field>)`, conditions inside nested `if`s where
  the proto guard is in an outer `if`, the RHS of an assignment that
  is itself a condition, `not`-wrapped guards, or `else`/`elif`
  branches where the guard's truth value is false.
- **Some "guards" aren't actually establishing.** The rule lists
  `pkt.proto == icmp` as a v4-establishing guard, justified by
  "icmp is v4-only". The proto-enum table contradicts: in v6-active
  programs `icmp` matches v6 frames whose `next_header == 1`. The
  guard is unsound.
- **Some "guards" are inconsistently establishing.** `pkt.proto ==
  icmp6` is listed as a v6-establishing guard but **not** as an
  L3-establishing guard (which it transitively is — v6 parse
  succeeded → L3 parse succeeded). Two adjacent bullets disagree on
  which inner reads are admissible.
- **Worked examples violate the rule.** Example #1 (the SSH
  brute-force flagship) and Example #5 (the stack-budget
  stress-test) both fail the dominator check. The dogfood example
  was patched to satisfy the rule explicitly with `if pkt.src_ip
  in [0.0.0.0/0]:`, demonstrating the spec author knew of the rule
  — but the worked examples weren't re-checked.

The unifying root cause is that v0.2 turned `pkt.proto`, the L4
fields, and the L3 fields into context-dependent reads (each
requiring a different dominator), but the enumeration of "what
counts as a guard" and "what positions need a guard" was made by
hand and missed several positions and several
guard-establishing/non-establishing distinctions. Each unhandled
position is its own Class A risk: interpreter and BPF runtime can
land on different answers on the unguarded surface, and the spec
provides no rule to adjudicate.

This is the densest pattern in the Tier 2 spec hunt: 13 confirmed
findings, several with cross-references to other patterns
(`example-contradicts-rule`, `strict-superset-erosion`).

## Check Strategy

1. **Enumerate every readable `pkt.<field>` surface in v0.2** —
   `pkt.proto`, `pkt.src_ip`, `pkt.dst_ip`, `pkt.src_ip6`,
   `pkt.dst_ip6`, `pkt.src_port`, `pkt.dst_port`, `pkt.tcp.syn`,
   `pkt.tcp.ack`, plus the implicit reads from
   `rate_limit(per=<field>)` and `geoip(...)`'s LHS. For each
   surface, check that the dominator rule covers every legal
   syntactic position: condition (top-level), inner condition (in a
   nested `if`), statement RHS, assignment RHS that is itself a
   condition, `not`-wrapped, `else`/`elif`, `rate_limit` implicit
   read, `geoip()` implicit read.
2. **Cross-validate guard-establishment claims against the
   proto-enum table.** Any keyword whose "Family allowed" cell is
   cross-family (currently `tcp`, `udp`, `icmp`) cannot be a v4-only
   guard in v6-active programs. The dominator rule must make this
   conditional or drop the guard.
3. **Audit guard transitivity.** A v6-establishing guard
   transitively establishes L3 (v6 parse ⇒ L3 parse). An
   L3-establishing guard does not transitively establish v4 or v6
   alone. The rule's enumeration of v6-/v4-/L3-guards must be
   consistent with this transitive closure.
4. **For every Tier 2 worked example, run a synthetic compile that
   only implements the dominator rule** (no other passes). Examples
   that fail to compile are spec contradictions.
5. **Adversarial fuzz: synthesize Tier 2 programs whose only `pkt`
   read sits in (a) an `else` branch, (b) a `not`-guarded body,
   (c) the RHS of an assignment that is itself a condition,
   (d) inside `rate_limit(per=...)`, (e) inside `geoip(...)`'s LHS
   on cross-family proto-equality.** Each of these positions is a
   separate finding-candidate.
6. **Enumerate the family/L3 truth-table** for each of the eight
   binary outcomes (v4 parse succeeded × v6 parse succeeded × L3
   parse succeeded × L4 parse succeeded, restricted to legal
   combinations) and confirm the dominator rule produces a defined
   answer for every legal pkt-read at every position.

## Known Instances

Surface omissions (the rule doesn't cover this position):

- `finding/2026-05-01-v02-spec-tier2-pkt-proto-no-dominator-rule` —
  `pkt.proto` itself in statement position.
- `finding/2026-05-01-v02-spec-tier2-stmt-level-pkt-read-on-wrong-protocol`
  — original spec hole (now fixed for L4 fields, kept here as the
  parent of the pattern).
- `finding/2026-05-01-v02-spec-tier2-stmt-level-ipv6-l3-read-undefined-value`
  — original spec hole for v6 L3 reads.
- `finding/2026-05-01-v02-spec-tier2-rate-limit-implicit-pkt-read-not-dominator-checked`
  — implicit read by `rate_limit(per=<field>)`.
- `finding/2026-05-01-v02-spec-tier2-nested-if-condition-reads-not-covered-by-dominator-rule`
  — inner `if` condition position.
- `finding/2026-05-01-v02-spec-tier2-dominator-rule-negation-and-else-not-addressed`
  — `not`/`else`/`elif` branches.
- `finding/2026-05-01-v02-spec-tier2-condition-on-rhs-of-assign-dominator-vs-shortcircuit-conflict`
  — RHS of assignment that is itself a condition.

Guard-establishment errors (the rule says X establishes Y, but doesn't):

- `finding/2026-05-01-v02-spec-tier2-pkt-proto-icmp-as-v4-guard-vs-cross-family-icmp`
  — `pkt.proto == icmp` claimed v4-only; cross-family in v6-active.
- `finding/2026-05-01-v02-spec-tier2-icmp6-establishes-v6-guard-but-not-l3-guard`
  — `pkt.proto == icmp6` blessed as v6-guard but excluded as
  L3-guard for an inner statement-position `pkt.proto` read.
- `finding/2026-05-01-v02-spec-tier2-pkt-proto-in-list-with-icmp6-vs-disjunction-rule`
  — `pkt.proto in [icmp6, ...]` ambiguous; depending on what fills
  `...` the disjunction may or may not establish v6.

Examples violate the rule:

- `finding/2026-05-01-v02-spec-tier2-example1-rate-limit-per-src-ip-violates-dominator-rule`
- `finding/2026-05-01-v02-spec-tier2-example5-violates-grammar-and-dominator-rule`
- `finding/2026-05-01-v02-spec-tier2-stmts-after-terminal-example-violates-proto-guard`

## Where to Look Next

- **The L3-establishing-guard list** is currently implicit. The spec
  enumerates v4-establishing and v6-establishing separately;
  L3-establishing should be defined as their union (plus the
  family-restricted proto-keyword guards), but the spec doesn't
  state this. Audit every `pkt.proto` read position for an explicit
  L3 guard.
- **`pkt.tcp.syn`/`pkt.tcp.ack` cross-family.** The proto-enum
  table makes `pkt.proto == tcp` cross-family. If a v6+TCP frame
  reaches a Tier 2 statement-position `pkt.tcp.syn` read, what does
  it read? The dominator rule covers L4 reads with a `pkt.proto ==
  tcp` guard — but `pkt.proto == tcp` is cross-family in v6-active
  programs, so the read is on v6+TCP, where the L4 offset is
  different. The rule may need a v4-or-v6 disambiguation that
  isn't currently spelled out.
- **`geoip(...)` LHS family-binding inside a Tier 2 condition.**
  The spec defers `pkt.src_ip6 in geoip(...)` family-aware lookups
  to v0.3, but the Tier 2 grammar admits the call. If a Tier 2
  function has `if pkt.<some_v6_field> in geoip(...)` without an
  L3-v6 guard above it, the analyser must reject — the rule
  doesn't currently say so.
- **Loop bodies (deferred to v0.3).** The reserved-word list keeps
  `for`/`while` as forward-compat. When v0.3 lands, the dominator
  rule's "control-flow dominator" must be redefined for cyclic
  control flow. Audit early — write the v0.3-blocking notes now.
- **Action statements as terminators.** `allow`/`drop` are
  terminators; `log`/`count` are not. The reachability analysis
  intersects with the dominator analysis (`stmts-after-terminal`
  finding). A program shape `if pkt.proto == tcp: allow` followed
  by `pkt.dst_port` read at function-end is dominator-clean only
  if the analyser knows `allow` is a terminator. Audit this
  intersection.
- **Bare-bool-local-as-condition + dominator transitivity.** If
  `internal = pkt.src_ip6 in fc00::/7` then `if internal: addr =
  pkt.src_ip6` — does the bool local "carry" the v6 guard
  transitively? The spec doesn't say. Hone fuzz can synthesize
  programs of this shape and demonstrate disagreement.
- **Cross-pattern: examples that violate the rule.** Cross-reference
  with pattern `example-contradicts-rule`. Every Tier 2 example not
  yet checked against the dominator rule is a candidate.
