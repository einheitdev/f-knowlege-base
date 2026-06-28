---
id: finding/2026-05-01-v02-spec-geoip-bundle-json-verbatim-vs-curated-subset
type: finding
protocol: []
builtins: [geoip]
severity: medium
layer: spec
pattern_tags: [spec-inconsistency, geoip, bundle-format]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: [finding/2026-05-01-v02-spec-geoip-map-id-per-callsite-vs-per-callsite-family-pair]
---

# 2026-05-01-v02-spec-geoip-bundle-json-verbatim-vs-curated-subset

## Summary

`FWL_V02_SPEC.md` specifies the bundle's `geoip.json` two ways. The
`geoip()` Semantics section mandates a verbatim copy of the
`--geoip-source` file. The Bundle additions section calls the
verbatim-vs-curated choice an implementation choice. Both can't be
right: an implementation either has the freedom to subset or it
doesn't.

The two readings have different observable consequences (file size,
hot-reload behaviour, what the daemon must accept), so this is a
spec-level contradiction with downstream impact on the bundle wire
format.

Class B (spec contradiction; bundle wire-format ambiguity).

## Locations

### Reading A — verbatim copy required (Semantics)

`FWL_V02_SPEC.md:357-361`:

> *Bundle-time.* `fwl compile --bundle` reads a static `geoip.json`
> from the path passed via `--geoip-source` (default
> `<cwd>/geoip.json`). … Codes referenced by the program but absent
> from the `geoip.json` cause `fwl compile --bundle` to fail with
> `error: geoip code 'XX' not present in geoip.json`. **The bundle's
> `geoip.json` is copied into the bundle directory verbatim** so the
> daemon can re-read it during a hot reload.

"Verbatim" is an absolute term. There's no escape clause.

### Reading B — verbatim *or* curated subset, implementer's choice

`FWL_V02_SPEC.md:840-844`:

> 2. **`geoip.json`** in the bundle directory. **A verbatim copy of
>    the `--geoip-source` file (or a curated subset containing only
>    the codes referenced by the program — implementation choice;
>    the spec requires that the file be sufficient to populate every
>    referenced code).**

This explicitly grants the bundler a choice between verbatim and
curated subset.

## Why this matters

The two readings disagree on three observable behaviours:

| Behaviour | Reading A (verbatim) | Reading B (subset OK) |
|---|---|---|
| Bundle file size | Always the size of source `geoip.json` (e.g. 8 MB for the full IP2Location-LITE table). | Can be tiny — just the codes the program uses. |
| Daemon's tolerance for extra codes | Daemon must accept `geoip.json` keys not referenced by the manifest (silent drop). | Same — but the bundler may have already pruned them. |
| What `fd: refuse to attach: bundle's geoip.json is inconsistent with manifest` (`FWL_V02_SPEC.md:851`) actually means | Inconsistency = some manifest-required code missing from `geoip.json`. | Same direction; but a curated subset by definition matches manifest exactly, so the inconsistency window is narrower. |
| Reproducibility of bundle output | Bundle hash is deterministic w.r.t. `(source.fw, geoip.json)`. | Bundle hash is deterministic w.r.t. `(source.fw, geoip.json, bundler-curating-policy)` — implementer-defined, so different `fwl` builds can produce different bundles for the same inputs. |

Reading B undermines the spec's reproducibility claim at
`FWL_V02_SPEC.md:426-428`:

> *Bundle-time refresh between calls.* `fwl compile --bundle` is
> deterministic given the same source `.fw` and the same
> `geoip.json` — running it twice produces identical bundle output
> (modulo the `current` symlink rotation).

If the bundler may freely choose between verbatim and a curated
subset, then "the same `.fw` + the same `geoip.json`" can produce
two distinct bundle outputs across implementations (or even across
versions of the same implementation if the curating heuristic
changes). The determinism claim only holds under Reading A
(verbatim).

## How the rest of the spec leans

The semantics text at line 357-361 is in the *Semantics* paragraph
of the geoip section — the part that reasonable readers treat as
authoritative for how the construct works. The Bundle additions
section at line 819-855 is the bundle-format reference. When two
sections disagree, the Semantics text is the natural source of
truth, and the bundle reference should match.

The hot-reload language at line 360-361 ("so the daemon can re-read
it during a hot reload") only makes sense under Reading A: a curated
subset would break a future hot-reload that adds a country to the
program (the new code wouldn't be in the subset). That, plus the
deterministic-bundle claim, both point at Reading A as the intended
behaviour.

## Proposed fix

Drop the parenthetical at `FWL_V02_SPEC.md:842-844`:

> 2. **`geoip.json`** in the bundle directory. ~~A verbatim copy of
>    the `--geoip-source` file (or a curated subset containing only
>    the codes referenced by the program — implementation choice;
>    the spec requires that the file be sufficient to populate every
>    referenced code).~~

Replace with:

> 2. **`geoip.json`** in the bundle directory. A verbatim copy of
>    the `--geoip-source` file. (See the geoip section's *Bundle-time*
>    paragraph at `FWL_V02_SPEC.md:357-361` for the rationale: the
>    daemon must be able to re-read the file during a hot reload, and
>    bundle output must be deterministic given the source `.fw` and
>    the source `geoip.json`.)

Optionally — if the spec authors want to keep the curated-subset
escape hatch — flip it the other way: drop "verbatim" from line 360
and replace it with "may be a curated subset", and flag the
reproducibility hit as a known gap to v0.3.

## Class

Class B (spec contradiction). Spec-layer.
