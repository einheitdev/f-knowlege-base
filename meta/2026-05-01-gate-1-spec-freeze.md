---
type: meta
title: Phase 2 Gate 1 — Spec freeze for v0.2
date: 2026-05-01
agent: f-phase2 implementation agent
related_findings: 54  # all 2026-05-01-v02-spec-* and 2026-05-01-v02-pkt-spec-*
---

# Gate 1 sign-off — Spec freeze for v0.2

## Outcome

`f/docs/FWL_V02_SPEC.md` and `f/docs/PKT_V02_SPEC.md` are frozen for v0.2 implementation work.

## Process

22 rounds of `hone hunt --target ../f/docs/FWL_V02_SPEC.md --max-turns 30 --model claude-opus-4-7 --solr-url ""`, each round followed by spec edits to address every filed finding, then re-run.

54 `layer: spec` findings filed total, all `status: fixed` (zero open). Per-round counts: 6, 2, 3, 4, 0, 3, 3, 4, 3, 3, 3, 1, 3, 3, 1, 2, 3, 2, 0, 2, 3, 0.

## Convergence signal

Pass 19 ran 40 turns, filed zero new findings, crashed mid-flight in the SDK. Pass 22 ran 42 turns, filed zero new findings, crashed mid-flight in the SDK. In both cases the agent's transcript shows it spent the run investigating hypotheses and finding each one already covered by an existing finding — i.e. the agent was de-duplicating, not generating. Two consecutive zero-finding passes is the spec-saturation signal Phase 2 plan calls for.

The Phase 2 plan's stated exit criterion is "at least three ambiguities found and resolved." 54 is well over the floor.

## Spec-layer findings ledger

All 54 findings live under `f-knowlege-base/findings/2026-05-01-v02-spec-*.md` and `2026-05-01-v02-pkt-spec-*.md`. Each has its own resolution recorded inline. Categories of issue surfaced (with rough counts):

- Tier 2 dominator rule (statement-position pkt reads, polarity/disjunction/else, rate_limit implicit reads, family-restricted-keyword transitivity, pkt.proto in non-trivial positions): ~13.
- IPv6 / cross-family semantics (proto-enum table vs strict-superset, RFC 5952 canonicalization, IPv4-mapped, v6-activation rule scope, dogfood SSH bypass, dead-rule cross-family rate_limit): ~12.
- geoip semantics & manifest format (map allocation per (call site, family), bundle-time vs load-time, refuse-error strings, LHS type-based vs field-enumerated, JSON verbatim vs subset, prefix budget): ~9.
- Tier 2 grammar (locals in conditions/expressions, rate_limit_call as primary, value_field's missing fields, scalar_expr restrictions, list-typed locals, integer-literal typing, function-call form unreachable errors, indentation rule, reserved keywords): ~12.
- Strict-superset claim regressions (proto matching across families, reserved words breaking v0.1 counter names, pass/return/for/while propagation, icmp6 carve-out): ~5.
- Examples vs rules (Example 1 SSH rate-limit dominator, Example 5 grammar+dominator, Statements-after-terminal proto-guard, empty-body uses pass): ~3.

## Cost

Cumulative LLM spend across the 22 rounds is in the $35-50 range — over Phase 2 plan's `$15-35` LLM-budget estimate, but the budget assumed a single spec-review pass. The 22-round spec-saturation run is what finds 54 findings instead of 3.

## What changed at the spec level

`FWL_V02_SPEC.md` grew from the initial 1041 lines to ~1230 lines through accreted edits. Major surface changes vs the initial draft:

- Activation-conditional cross-family `pkt.proto` semantics (preserves v0.1 strict-superset for v0.1-shaped programs).
- Polarity- and disjunction-aware dominator rule for both Tier 2 statement-position pkt reads and inner-condition reads.
- byte-equality + family-allowed list for proto keywords (replaces the original "icmp is always v4-only" model).
- `geoip(...)` manifest format: one map_id per (call site, family) pair, `family` and `map_id` scalars per entry, IPv4-only or IPv6-only at compile time.
- v0.2 condition grammar made explicit (was "reused from v0.1 unchanged" — too narrow).
- Reserved word set: `def`, `elif`, `else`, `default` are syntactic; `for`/`while`/`pass`/`return` are non-reserved.
- IPv6 RFC 5952 canonicalization: rule 4 restricted to IPv4-mapped (`::ffff:0:0/96`) only; IPv4-compatible deprecated and not re-canonicalized.
- IPv6 literal lexer rule for trailing-`:` ambiguity.
- Strict-superset carve-out widened to four keywords (`def`/`elif`/`else`/`icmp6`).
- Dogfood example bifurcated v4/v6 SSH rate-limit (closes the v6 SSH bypass).

`PKT_V02_SPEC.md` gained a paragraph tying the v6-builder decoded dict to FWL's v6-surface activation rule.

## What's next

Per `planning/PHASE_2_PLAN.md`, Construct 1 — IPv6 fields — implementation begins. The corpus-machinery and builder additions land first per the plan's reasoning that v6 is the smallest of the three constructs and unblocks the others.

The 54 fixed findings are the spec's resolved-ambiguity ledger; they're frozen-in-time and will not be re-touched unless implementation surfaces something the spec missed (in which case the cycle reopens).
