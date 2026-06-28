---
id: pattern/strict-superset-erosion
type: pattern
protocol: [tcp, udp, icmp, icmp6]
builtins: []
severity: high
layer: [spec]
pattern_tags: [strict-superset, v01-compat, cross-family, ipv6, reserved-words, regression-hazard, activation-rule]
status: confirmed
created: 2026-05-01
instance_count: 7
instances:
  - finding/2026-05-01-v02-spec-strict-superset-vs-cross-family-pkt-proto
  - finding/2026-05-01-v02-spec-new-reserved-words-break-strict-superset-claim
  - finding/2026-05-01-v02-spec-icmp6-counter-name-breaks-strict-superset-carveout
  - finding/2026-05-01-v02-spec-v6-activation-rule-only-covers-equality-not-inequality-or-in-or-locals
  - finding/2026-05-01-v02-spec-proto-equality-table-vs-opaque-byte-equality
  - finding/2026-05-01-v02-spec-pass-return-loop-keywords-reserved-status-self-contradicts
  - finding/2026-05-01-v02-pkt-spec-tcp6-decoded-proto-tcp-vs-fwl-activation-rule
---

# strict-superset-erosion

## Description

The v0.2 spec opens with the strict-superset pledge — "every v0.1
program is a valid v0.2 program with identical semantics" — and
makes that pledge load-bearing for the methodology rule "the v0.1
corpus is the regression oracle for every v0.2 PR" (per
`planning/PHASE_2_PLAN.md`). Several v0.2 features were added without
reconciling them against the pledge:

- **Cross-family proto-keyword semantics.** The proto-enum table
  redefines `pkt.proto == tcp|udp|icmp` to match v6 packets when the
  program activates the v6 parse path. A v0.1 program that uses
  `pkt.proto == tcp` reads as v4-only under v0.1 and as v4-or-v6
  under v0.2, breaking identical-semantics on every v6 frame.
- **Lexical reservation expansion.** v0.2 reserves new keywords
  (`def`, `elif`, `else`, `icmp6`, plus an unstable list of
  `pass`/`return`/`for`/`while`) that v0.1 admits as identifiers
  (counter names, interface names). v0.1 programs using these as
  identifiers fail to compile under v0.2 with no migration warning.
- **Incomplete activation rule.** The v6 parse-path activation rule
  enumerates a small surface (`==` against `icmp6`, etc.) but omits
  `!=`, `in`-list, and Tier 2 hoists. A program that mentions a v6
  surface in any uncovered position silently fails to activate v6
  parse — the program "compiles" but the implementer who reads the
  rule literally produces a runtime that disagrees with the
  proto-enum table.
- **Cross-document drift.** `PKT_V02_SPEC.md` and `FWL_V02_SPEC.md`
  give different answers on the same question (e.g., what's in the
  decoded-fields dict for a `tcp6` builder).

The shape is uniform: a v0.2 feature was added with a clear
in-section motivation, but the strict-superset paragraph (and the
companion claims at `:46`, `:935-936`) was not narrowed to admit
the carve-out. Two paragraphs in the same document make
incompatible promises about a concrete v0.1 program. The
implementer who reads only the feature ships an implementation that
fails a v0.1 corpus case; the implementer who reads only the
strict-superset paragraph ships an implementation the spec
contradicts.

## Check Strategy

1. **Take every claim of the form "v0.1 program X compiles
   unchanged" and find a v0.1 program that exhibits the new
   v0.2 surface.** If the surface changes the program's behaviour,
   you've hit an instance.
2. **Audit the lexical-reservation list against
   `FWL_V01_SPEC.md:39`'s identifier production**. Every new
   reserved word that matches `[a-z_][a-z0-9_]*` and isn't already a
   v0.1 keyword is a strict-superset break unless the carve-out
   paragraph explicitly lists it.
3. **For every v6-related rule, audit which syntactic surfaces
   trigger it.** `==`, `!=`, `in`, `not in`, list-membership over
   each operand kind, Tier 2 hoist into a local. The activation
   rule is a single example of the more general "trigger surface
   enumeration" check; any rule that depends on which surface a
   program uses needs the same audit.
4. **Cross-walk `PKT_V02_SPEC.md` claims against `FWL_V02_SPEC.md`
   claims for every shared concept** — proto keyword, decoded-fields
   dict, builder field, geoip data layout. If they say different
   things, one of them is the canonical answer; the other is drift.
5. **Run the v0.1 corpus through the v0.2 toolchain on every commit
   that touches the spec or the lexer.** Regressions in a v0.1
   case are the high-signal fingerprint. (The methodology already
   says this; the pattern is which kinds of v0.2 changes most
   commonly trip it.)
6. **For every "Family allowed" cell in the proto-enum table that
   says "only when the program activates the v6 parse path", check
   that the in-effect rule is reachable from each of the v6
   activation triggers** — including the ones not listed in the
   activation rule.

## Known Instances

- `finding/2026-05-01-v02-spec-strict-superset-vs-cross-family-pkt-proto`
  — proto-enum table makes `pkt.proto == tcp|udp` cross-family;
  every v0.1 program using `tcp`/`udp` keywords flips behaviour on
  v6 packets in v0.2.
- `finding/2026-05-01-v02-spec-new-reserved-words-break-strict-superset-claim`
  — seven new reserved keywords break v0.1 programs using them as
  counter names; carve-out paragraph never narrowed.
- `finding/2026-05-01-v02-spec-icmp6-counter-name-breaks-strict-superset-carveout`
  — `icmp6` is the most likely operator-written v0.1 counter name
  and is the one the carve-out forgets.
- `finding/2026-05-01-v02-spec-v6-activation-rule-only-covers-equality-not-inequality-or-in-or-locals`
  — activation rule omits `!=`, `in`-list, Tier 2 hoist; programs
  using those surfaces silently lose v6 parse.
- `finding/2026-05-01-v02-spec-proto-equality-table-vs-opaque-byte-equality`
  — proto-keyword equality is described as "byte equality" in some
  paragraphs and "family-aware" in others; v0.1 programs that
  hoist the proto byte and compare it to the keyword get different
  answers under each reading.
- `finding/2026-05-01-v02-spec-pass-return-loop-keywords-reserved-status-self-contradicts`
  — three sections disagree on whether `pass`, `return`, `for`,
  `while` are reserved; the migration script for the v0.1 corpus
  cannot be written deterministically.
- `finding/2026-05-01-v02-pkt-spec-tcp6-decoded-proto-tcp-vs-fwl-activation-rule`
  — PKT spec says the `tcp6` builder's decoded dict has `proto:
  tcp`; FWL spec says `pkt.proto` is unreadable on v6 frames in
  v0.1-shaped programs. The two oracles read different fields.

## Where to Look Next

The pattern's natural growth surface is **anywhere v0.2 adds a
feature whose interaction with v0.1 was deferred or assumed
benign**. Concrete next-targets:

- **`pkt.tcp.syn`/`pkt.tcp.ack` on v6 frames.** The proto-enum
  table makes `tcp` cross-family in v6-active programs. v0.1
  programs read `pkt.tcp.syn` only on v4 frames; v0.2 v6-active
  programs may read it on v6+TCP frames. If any v0.1 corpus case
  has `pkt.tcp.syn` plus an unrelated v6 surface, the same
  expansion bites — same shape as the cross-family pkt.proto
  finding, different field.
- **`@xdp(<iface>)` decorator argument.** v0.1 admits any
  identifier as the interface name. If `icmp6` lexes as
  PROTO_KEYWORD in v0.2, `@xdp(icmp6)` fails to parse — a third
  surface for the icmp6-counter-name finding.
- **Whitespace/indentation lexer mode change.** Tier 2 needs
  indentation tracking; the `FwlIndenter` postlex is gated on the
  `def` keyword (see Session State decisions). Any future v0.3
  construct introduced after `def` retriggers the gating
  logic — a strict-superset hazard for programs that happen to use
  the new construct's keywords as identifiers.
- **`F_DEVELOPMENT_METHODOLOGY.md` invariants** that depend on the
  superset claim. The methodology document treats the v0.1 corpus
  as the regression oracle; if the strict-superset claim weakens to
  "near-superset modulo carve-out", the methodology may need a
  carve-out marker on each v0.1 corpus case (so a hone regress can
  distinguish "v0.1 case migrated" from "v0.1 case unchanged").
- **Cross-family `geoip()` LHS.** The spec defers v4-mapped-into-v6
  LPM-trie equivalence; the deferral is a strict-superset hazard
  the moment a v0.1 program with a `geoip()` v4 lookup is
  evaluated against a v4-mapped-v6 packet under v0.2 — the v0.1
  semantics had no v6 surface to consider.
- **Bundle-format/manifest backwards compatibility.** v0.1 bundles
  have no geoip section; v0.2 bundles do. A v0.1 program loaded
  through the v0.2 daemon should produce a manifest with no geoip
  section. If the manifest emitter unconditionally writes the
  geoip key (even empty), the daemon's bundle-load is sensitive to
  schema changes — a load-time strict-superset break.
