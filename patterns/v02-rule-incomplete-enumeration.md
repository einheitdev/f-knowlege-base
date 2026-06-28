---
id: pattern/v02-rule-incomplete-enumeration
type: pattern
protocol: [tcp, udp, icmp, icmp6]
builtins: [rate_limit]
severity: medium
layer: [spec]
pattern_tags: [tier2, lexical, reachability, indentation, reserved-words, dead-rule, rate-limit, spec-ambiguity, spec-incompleteness, undefined-behavior]
status: hypothesis
created: 2026-05-04
instance_count: 4
instances:
  - finding/2026-05-01-v02-spec-tier2-indentation-rule-enumerated-list-contradicts-consistency-rule
  - finding/2026-05-01-v02-spec-tier2-reserved-keyword-set-not-enumerated
  - finding/2026-05-01-v02-spec-tier2-unreachable-after-fully-terminating-if-else
  - finding/2026-05-01-v02-spec-rate-limit-per-src-ip-with-v6-only-predicate-dead-rule
---

# v02-rule-incomplete-enumeration

## Description

v0.2 introduces several new constructs (Tier 2 functions and
locals, IPv6 fields, cross-family rate-limit combinations) that
require new analyser rules. The spec writes the rules for the
common cases but leaves enumeration gaps the surface admits — and
the gap is **not acknowledged** as deferred or undefined. Two
implementers reading the spec land on different verdicts on the
same source, and the `.pkt` corpus has no spec text to assert
either verdict against.

This pattern is the sister of `tier2-dominator-rule-coverage-holes`
for **non-dominator** rules. The dominator-rule pattern catalogues
how the protocol-guard enumeration misses surfaces; this pattern
catalogues the same shape applied to:

- **Lexical rules.** Indentation step sizes (rule says "consistent"
  + "2/4/tab" — two halves of one sentence contradict). Reserved
  keyword set (no enumeration at all — the spec uses literal
  terminals in the grammar but never says which identifiers are
  banned in user-name positions).
- **Reachability rules.** Statement-after-terminal covers the
  bare `drop\nallow` case but the "per branch" hint is unformalized
  for the if/else-where-every-branch-terminates case.
- **Rule-composition rules.** A rate-limit rule whose predicate
  is v6-only and whose `per=` is v4-only is provably dead, but the
  spec neither rejects it nor warns. Two correct rules combined
  produce an undiagnosed class.

The shape is uniform:

- **The construct's surface admits more cases than the rule
  enumerates.** Either the grammar is broader than the rule
  ("identifiers" admits any lowercase identifier, but reserved
  words aren't enumerated), or the construct's combinatorics
  exceed the rule's worked examples (if/else with every branch
  terminating; v6-predicate + v4-per combo).
- **The spec doesn't say "deferred to v0.3" or "implementer
  defined".** The gap is silent — neither rejected nor accepted
  nor explicitly left open. An implementer must invent behaviour.
- **The Compile-errors table is silent on the gap.** No row covers
  the case the rule doesn't handle, so even an implementer who
  decides to reject has no spec-blessed string to emit.
- **Two `.pkt` test authors disagree.** One writes
  `expected.compiles: false` against an invented error message;
  the other writes `expected.compiles: true` and asserts a
  runtime action. The corpus splits.

The downstream effect is **silent oracle drift**: interpreter and
BPF emitter, written by different authors at different times,
land on different verdicts for the gap-shaped programs. Because
the gap isn't acknowledged, neither author knows they're each
filling it differently.

This is a hypothesis pattern (N=4) — the shape is mechanically
clear and each instance points to a missing rule clause, but four
instances across four different rules (indentation, reserved
words, reachability, rate-limit-composition) is a sparse signal.
A v0.3 audit of every Tier 2 / IPv6 / cross-family rule's
enumeration would either confirm this pattern or split it into
narrower per-rule sub-patterns.

## Check Strategy

1. **Enumerate every analyser rule the v0.2 spec adds** (Tier 2
   indentation, reserved-keyword set, reachability, dominator,
   stack-budget, type rules for new locals, cross-family
   rate-limit combinations, v6 activation rule, geoip call-site
   identity rule). For each, list the surface positions the
   construct admits and confirm the rule enumerates a verdict for
   every position. Positions with no verdict are findings of this
   shape.
2. **For every Compile-errors table row, locate the rule it
   enforces.** A row exists only when its rule is explicit. The
   reverse — every rule has a table row — is what this pattern
   checks. Rules with no table row are candidates: either the
   table needs a row, or the rule needs to declare the case
   undefined / accepted.
3. **For every "consistent" or "uniform" word in the spec**,
   check that the consistency criterion is fully specified. The
   indentation finding shows the failure mode: "consistent" is
   under-defined when the same sentence also enumerates a closed
   set of accepted forms.
4. **For every "intent" claim about a construct combination**,
   walk the truth table of legal combinations and confirm each
   cell has a defined verdict. The rate-limit finding shows the
   failure mode: predicate-family × per-field-family is a
   well-defined cross-product, but the spec only treats the
   "matching family" diagonal explicitly.
5. **For every grammar terminal that overlaps with the identifier
   production** (lowercase keyword), check the spec enumerates
   whether it is reserved in user-name positions. The
   reserved-keyword finding shows the failure mode: the spec uses
   `def`, `if`, `geoip`, `tcp`, etc. as literal terminals but
   never says they are reserved as Tier 2 function names or
   locals.
6. **For every "per branch" / "per X" hint**, locate the rule it
   implements. The unreachable finding shows the failure mode:
   "Reachability is computed per branch" appears as commentary
   without a corresponding rule clause; the textual rule that
   actually runs covers a narrower case.

## Known Instances

- `finding/2026-05-01-v02-spec-tier2-indentation-rule-enumerated-list-contradicts-consistency-rule`
  — single sentence has two halves: "as long as it is consistent"
  and "2-space, 4-space, or 1-tab". 3-space-uniform programs are
  accepted by one half and rejected by the other.
- `finding/2026-05-01-v02-spec-tier2-reserved-keyword-set-not-enumerated`
  — the spec reserves only `pkt` explicitly; the status of `def`,
  `if`, `geoip`, `tcp`, etc. as Tier 2 local / function-name
  identifiers is silent. Two lexers can produce different parse
  results for the same source.
- `finding/2026-05-01-v02-spec-tier2-unreachable-after-fully-terminating-if-else`
  — the unreachable rule covers `drop\nallow` (single-statement
  predecessor) but not `if X: drop else: drop\nallow` (every-arm
  terminates). The "per branch" hint at `:682` is unformalized.
- `finding/2026-05-01-v02-spec-rate-limit-per-src-ip-with-v6-only-predicate-dead-rule`
  — a v6-only predicate with a v4-only `per=` field is a class of
  dead Tier 1 rate-limited rules with no diagnostic. Composition
  of two correct rules creates an undefined class.

## Where to Look Next

This pattern's growth surface is **any v0.2 rule whose enumeration
hasn't been mechanically walked against the construct's full
surface**. Each Tier 2 / v0.2 rule should be re-examined the way
`tier2-dominator-rule-coverage-holes` examined the dominator rule.

- **Tier 2 stack-budget rule.** The spec gives a rough budget
  (~512 bytes per local-set; `FWL_V02_SPEC.md:557-572`). The
  stack-budget estimator's enumeration of "what counts as stack
  cost" should cover every Tier 2 local kind, every Tier 2
  expression form, and every implicit pkt-read-into-temporary that
  the emitter may produce. If any case is uncounted, the analyser
  may bless a program the BPF verifier rejects (or vice versa).
- **v0.2 `count <name>` rule under Tier 2.** Counter names live in
  a "disjoint namespace" per `:565-568` but are still subject to
  reserved-keyword constraints (the v0.1 prior-art miss noted
  `count tcp` is silent in the spec). Two unresolved holes:
  (a) does `count <reserved-keyword>` parse, (b) does `count
  <local-shadowing-name>` produce the warning the spec mentions?
- **`@xdp(<iface>)` decorator argument lexical rule.** v0.1
  admits any lowercase identifier; v0.2 reserves more keywords
  (`icmp6`, `geoip`). Whether `@xdp(icmp6)` parses depends on
  the reserved-keyword resolution and is unenumerated.
- **Tier 2 first-assignment integer-literal typing.** The
  type-rules-holes pattern already catches `x = 22` ambiguity for
  `u16` vs `u32`. The complementary "first assignment with a
  field literal but no field context" case (e.g. `addr =
  10.0.0.1` — is it `ipv4`? a CIDR? a u32?) hasn't been audited.
- **Action sequence after `count`/`log` without proto guard.**
  `count <n>` and `log` are not terminators; programs of the
  shape `count fired; allow` need a reachability verdict for the
  `allow` against the dominator rule (since `count` may
  side-effect even on packets the dominator says are
  guard-failing). Cross-pattern with both this pattern and
  `tier2-dominator-rule-coverage-holes`.
- **Cross-family rate-limit `per=` combinations beyond the
  v6-predicate + v4-per case.** Symmetric forms once `per=src_ip6`
  lands in v0.3 (v4-predicate + v6-per — provably dead). ICMP /
  ICMPv6 with `per=src_port` (no L4 ports — provably dead-on-this-
  proto). Each cross-product cell is a candidate for the same
  shape.
- **Reachability under `if/elif` chain without `else`.** The
  unreachable finding addresses if/else where every arm
  terminates; the if/elif-without-else case ("the implicit
  fall-through, then a trailing terminal") needs the same
  truth-table audit. So does the nested case where one arm's
  inner if/else fully terminates.
- **`limited by rate_limit(...)` modifier on Tier 2 rules.** The
  spec only spells out `limited by` for Tier 1; the Tier 2 grammar
  admits `rate_limit(...)` as a primary inside `if` conditions
  but not as a Tier 2 modifier on an action statement. Whether
  `drop limited by rate_limit(...)` parses inside a `def` body is
  an enumeration gap.
- **`pkt.tcp.syn`/`pkt.tcp.ack` cross-family in Tier 1.** The
  v0.1-cross-family-pkt-proto finding (in `strict-superset-erosion`)
  flagged `pkt.proto == tcp` cross-family in v6-active programs.
  Whether `pkt.tcp.syn` reads on a v6+TCP frame is similarly
  unenumerated; same shape as the rate-limit-per dead-rule
  finding (composition of v0.2 cross-family rule and v0.1 L4 read).
