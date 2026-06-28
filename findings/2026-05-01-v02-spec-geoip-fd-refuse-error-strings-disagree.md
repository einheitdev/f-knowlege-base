---
id: finding/2026-05-01-v02-spec-geoip-fd-refuse-error-strings-disagree
type: finding
protocol: []
builtins: [geoip]
severity: low
layer: spec
pattern_tags: [spec-inconsistency, geoip, error-messages, bundle-format]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: [finding/2026-05-01-v02-spec-geoip-bundle-json-verbatim-vs-curated-subset]
---

# 2026-05-01-v02-spec-geoip-fd-refuse-error-strings-disagree

## Summary

The spec specifies two **different** runtime error strings that `fd`
must emit on bundle attach when the bundle's `geoip.json` is
missing. The Edge cases paragraph at `FWL_V02_SPEC.md:412-413`
gives one wording. The Bundle additions paragraph at
`FWL_V02_SPEC.md:849-851` gives another. A daemon implementor who
hard-codes one cannot satisfy a corpus test that expects the other.

Class B (spec contradiction; runtime-error-string layer).

## Locations

### Wording A — `bundle missing required geoip.json`

`FWL_V02_SPEC.md:412-416`:

> *Missing `geoip.json` at load time.* `fd` refuses to attach the
> bundle: `fd: refuse to attach: bundle missing required
> geoip.json`. This is a runtime test marked with
> `expected.compiles: true` and `expected.load_action: refuse` (a
> v0.2 addition to the `.pkt` spec — see PKT_V02_SPEC.md).

This is in the geoip section's Edge cases list. The wording is
specific: "bundle missing required geoip.json".

### Wording B — `bundle's geoip.json is inconsistent with manifest`

`FWL_V02_SPEC.md:849-851`:

> A bundle whose `geoip.json` is **missing** or whose codes do not
> cover the program's references is refused: `fd: refuse to attach:
> bundle's geoip.json is inconsistent with manifest`.

The same condition (geoip.json missing) yields the *other* wording.
The conjunction "missing or whose codes do not cover" puts the two
distinct failure modes (file absent; file present but incomplete)
behind one error string.

## Why this matters

`expected.load_action: refuse` is described as a v0.2 addition to
the `.pkt` spec (line 415-416). A `.pkt` test that exercises
"missing geoip.json" needs to assert against the *exact* error
string the daemon emits. With two specs in play:

- A test author copying from line 412 expects
  `bundle missing required geoip.json`.
- A test author copying from line 851 expects
  `bundle's geoip.json is inconsistent with manifest`.

The actual daemon implementation can emit only one string. So one
of the two wordings will produce false-positive corpus failures.

Worse, the line 851 wording conflates two different failure modes
(missing vs. partial) under one string. The diagnostic is
strictly less informative than the line 412 wording for the
"missing" case, and the analyser/runtime cannot distinguish the
two from the error string alone — which matters for users
debugging bundles.

Compare with the analyser's compile-time error for a similar
condition at `FWL_V02_SPEC.md:446`:

> | Bundle source missing required code | `error: geoip code '<XX>' not present in geoip.json` |

That is *not* the same as a missing `geoip.json` file. It's the
"file present, code absent" case. Three failure modes total, only
two of which the spec gives error strings for, and the runtime
side has the cross-cutting wording B that doesn't distinguish
them.

## Proposed fix

Pick **wording A** as canonical. It is more specific and matches
the `expected.load_action: refuse` test contract that the Edge
cases section bills as authoritative. Then split line 849-851
into two distinct error strings:

> A bundle whose `geoip.json` is missing is refused with
> `fd: refuse to attach: bundle missing required geoip.json`.
> A bundle whose `geoip.json` is present but does not contain
> every code the manifest requires is refused with
> `fd: refuse to attach: bundle's geoip.json is missing required
> codes: <XX>, <YY>`.

This gives the daemon implementor and the corpus author one
string per failure mode, makes the diagnostic actionable
("which codes are missing?"), and aligns the two paragraphs.

Alternatively, if "inconsistent" is preferred for both modes,
delete the wording-A row at line 412-413 and pin the spec to one
generic string. That degrades the diagnostic for users but
removes the contradiction.

## Class

Class B (spec contradiction). Spec-layer.
