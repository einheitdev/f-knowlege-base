---
id: pattern/geoip-spec-cross-section-drift
type: pattern
protocol: []
builtins: [geoip]
severity: high
layer: [spec]
pattern_tags: [geoip, spec-inconsistency, spec-contradiction, bundle-format, manifest, cross-section, leftover-text, error-messages, type-rules]
status: confirmed
created: 2026-05-04
instance_count: 5
instances:
  - finding/2026-05-01-v02-spec-geoip-map-id-per-callsite-vs-per-callsite-family-pair
  - finding/2026-05-01-v02-spec-geoip-up-to-two-map-ids-leftover-contradicts-one-family-per-callsite
  - finding/2026-05-01-v02-spec-geoip-bundle-json-verbatim-vs-curated-subset
  - finding/2026-05-01-v02-spec-geoip-fd-refuse-error-strings-disagree
  - finding/2026-05-01-v02-spec-geoip-lhs-pkt-fields-only-vs-type-based-ambiguous
---

# geoip-spec-cross-section-drift

## Description

The `geoip(...)` feature is the largest single addition v0.2 makes
to the language: it spans roughly eight normative sub-sections of
`FWL_V02_SPEC.md` — Construct intro, Type rules, Operators table,
Semantics (compile-time / load-time / hot-reload), Compile errors,
Bundle additions, Manifest format example, and Load-time text.
The sub-sections were edited in separate commits to fix earlier
issues (e.g. "one-family-per-callsite" patched the map-id model)
without re-walking the surrounding paragraphs. As a result two,
three, even four normative paragraphs in the same document make
incompatible claims about the same concrete question.

The shape is uniform across the five confirmed instances:

- **Two readings, both grammatical.** Each instance has at least
  one paragraph saying "X" and at least one saying "not X" (or "Y
  ≠ X"), with no meta-rule for which paragraph wins.
- **Both readings are normative.** Neither paragraph is in an
  example, an aside, or a footnote — both define the runtime,
  analyser, manifest format, or error-string contract.
- **The grammar is silent or permissive.** The parser accepts the
  surface either way; the dispute is at the analyser, manifest
  emitter, daemon load-path, or error-message layer.
- **Two implementers diverge.** An implementer who reads only one
  paragraph ships an analyser/manifest emitter/daemon that fails
  every test built against the other paragraph.

The unifying root cause is **piecemeal evolution of a substantial
feature without a cross-section reconciliation pass**. Each fix to
the geoip section landed in the immediate paragraph; sister
paragraphs in adjacent sections kept the pre-fix language. The
"leftover-text" finding is the explicit smoking gun — a patch that
added the new model and *forgot* to delete the sentence describing
the old one.

This is distinct from existing patterns:

- `example-contradicts-rule` is example-vs-rule (a code block
  contradicts a stated rule). This pattern is rule-vs-rule (two
  prose paragraphs disagree).
- `grammar-vs-body-text-drift` is grammar-vs-prose (EBNF disagrees
  with body text). This pattern is prose-vs-prose.
- `strict-superset-erosion` is v0.1↔v0.2 (superset claim breaks).
  This pattern is v0.2-internal (two v0.2 paragraphs disagree).

This is the densest cross-section pattern in the v0.2 hunt, and it
is concentrated in one feature area (geoip). Bundle wire-format,
load-time error strings, and analyser pass shape all key off these
contradictions.

## Check Strategy

1. **Enumerate every `geoip` sub-section in `FWL_V02_SPEC.md`** —
   Construct intro (`:286-301`), Type rules (`:313-330`), Operators
   table (`:874-887`), Semantics (`:349-410`), Compile errors
   (`:445-466`), Bundle additions (`:817-862`), Manifest format
   example (`:359-372`, `:779-784`), Load-time text (`:392`,
   `:849-851`). For every concrete claim — map allocation rule,
   manifest cardinality, LHS legality, error-string wording, prefix
   budget — check that every other sub-section that mentions the
   same question agrees verbatim.
2. **Diff against any "fix" commit that touched geoip.** Each fix
   should have been applied across all sub-sections that mention
   the patched question. The "leftover text" finding is the
   archetype: the post-fix paragraph says one thing; the pre-fix
   paragraph two sections away still says the opposite. `git log
   -p docs/FWL_V02_SPEC.md -- '*geoip*'` to walk the patch history.
3. **For every concrete `.pkt`-corpus-ready claim in the geoip
   section**, write the minimum `.pkt` file that exercises it —
   manifest assertions, `expected.load_action: refuse` cases with
   the literal error string, `expected.compiles` for cross-family
   `in geoip(...)` against locals, etc. Drift surfaces immediately
   as cross-implementer disagreement.
4. **Audit the bundle-format reference (`:817-862`) as the canonical
   answer for everything wire-format.** For each non-bundle
   paragraph that mentions a wire-format detail (e.g. `map_id`
   cardinality, `family` field, error strings), confirm the wire
   reference matches verbatim.
5. **For every error string the spec promises to emit**, search the
   whole document for any other place that names the same
   condition. Two strings for the same condition (the `fd: refuse`
   finding's shape) is the sub-pattern.

## Known Instances

- `finding/2026-05-01-v02-spec-geoip-map-id-per-callsite-vs-per-callsite-family-pair`
  — three sections give three different map-allocation rules (one
  per call site, one per (call site, family), or hybrid logical
  IDs). Bundle wire format depends on which wins.
- `finding/2026-05-01-v02-spec-geoip-up-to-two-map-ids-leftover-contradicts-one-family-per-callsite`
  — explicit leftover-text instance: the patch that introduced
  "one family per call site" left "up to two map IDs per call
  site" intact two paragraphs above. Three of four paragraphs say
  one thing; one paragraph says the other.
- `finding/2026-05-01-v02-spec-geoip-bundle-json-verbatim-vs-curated-subset`
  — Semantics paragraph mandates verbatim copy; Bundle additions
  paragraph allows curated subset as "implementation choice". The
  determinism claim only holds under verbatim.
- `finding/2026-05-01-v02-spec-geoip-fd-refuse-error-strings-disagree`
  — two distinct error strings for "missing geoip.json on bundle
  attach", in two different paragraphs. Daemon implementor cannot
  satisfy both `.pkt` corpus expectations simultaneously.
- `finding/2026-05-01-v02-spec-geoip-lhs-pkt-fields-only-vs-type-based-ambiguous`
  — Construct intro and Operators say "the four pkt IP fields,
  full stop"; Type rules say "any operand whose type is `ipv4` or
  `ipv6`". Tier 2 locals are the surface where the two readings
  diverge.

## Where to Look Next

The geoip section's surface is far from fully audited; this
pattern's natural growth surface is **every geoip sub-section that
hasn't been cross-walked against every other geoip sub-section**.

- **Hot-reload behaviour** (`:357-361` Semantics, `:392`
  load-time, `:798-803` daemon refresh discipline). These three
  paragraphs each describe what happens when `geoip.json` changes
  on disk; they have not been cross-validated. Likely cross-drift
  on whether existing `bpf_map_create`'d maps are reused, replaced,
  or torn down + re-created.
- **The 65 536-prefix budget** (`:351-354`, `:398-403`, error
  table at `:446`). The wording is "across all call sites and
  across both families" in one place but Reading B of the
  map-id finding (rejected by the post-fix patch) implied
  per-family budgets. Audit budget arithmetic in any place the
  spec mentions counts; one of them may still reflect the old
  model.
- **`PKT_V02_SPEC.md` geoip-related paragraphs.** The `geoip_data`
  block on `PktCase`, the `expected.load_action: refuse` key, the
  `compile_error_pattern` semantics for geoip errors. Cross-walk
  with `FWL_V02_SPEC.md` geoip section; PKT spec was edited
  separately and may carry its own drift.
- **`country_code` / `cc_code` lexical and validation rules.**
  Multiple sections say the codes are bare uppercase identifiers;
  one example uses a quoted string (already a fixed instance,
  covered by `example-contradicts-rule`). Audit every place the
  spec mentions ISO-3166 alpha-2 — the validation pass, the
  manifest format, the error strings.
- **Cross-pattern: `helper-skips-tier2-function-body`.** Several
  geoip findings depend on the analyser walking Tier 2 function
  bodies for `geoip(...)` call sites. The geoip-call-site
  enumerator is one of the helpers explicitly named in that
  pattern's "where to look next" — a finding there would
  cross-reference here.
- **The `iso3166` validation list itself.** The spec defers the
  list to a runtime data file but doesn't say where it comes from
  or how it's versioned. A future change to the list is a hot-
  reload-equivalent mutation; the spec's hot-reload paragraph
  doesn't mention country-code list changes. Likely a sister
  finding once the v0.3 hot-reload semantics are nailed down.
- **The `geoip(...)` call-as-rvalue future surface.** The
  type-rules paragraph implies `bad_actors = geoip(RU, CN)` would
  be type-able as `set<ipv4|ipv6>`, but the grammar doesn't admit
  the form. A future v0.3 patch that adds this surface will need
  to thread updates through every sub-section listed above —
  this pattern's check strategy is the gating discipline for that
  patch.
