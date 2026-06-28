---
id: finding/2026-05-01-v02-spec-geoip-up-to-two-map-ids-leftover-contradicts-one-family-per-callsite
type: finding
protocol: []
builtins: [geoip]
severity: high
layer: spec
pattern_tags: [spec-contradiction, geoip, bundle-format, manifest, leftover-text]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: [finding/2026-05-01-v02-spec-geoip-map-id-per-callsite-vs-per-callsite-family-pair, finding/2026-05-01-v02-spec-geoip-manifest-example-shows-unreachable-two-family-call-site]
---

# 2026-05-01-v02-spec-geoip-up-to-two-map-ids-leftover-contradicts-one-family-per-callsite

## Summary

The geoip section's **Type rules** paragraph at
`FWL_V02_SPEC.md:322-330` states that "a call site may allocate up
to two map IDs" (one v4 trie, one v6 trie). The **Semantics**
paragraph immediately following at `FWL_V02_SPEC.md:349-374` states
that "each call site is bound to exactly one address family" and
"there is exactly one `map_id` per manifest entry — the v0.2 grammar
has no syntactic surface that can bind one call site to two
families." The two paragraphs **directly contradict** each other on
the central question of how many map IDs a `geoip(...)` call site
allocates.

The Semantics paragraph (line 349) is the post-fix wording from the
related
`finding/2026-05-01-v02-spec-geoip-map-id-per-callsite-vs-per-callsite-family-pair`
patch. The Type-rules paragraph (line 322-330) is leftover text from
the pre-patch model that was never updated. Two implementors of the
analyser, parser, manifest emitter, and daemon load path will read
the spec and disagree on:

- How many map IDs to allocate per textual `geoip(...)` occurrence.
- Whether the same call site can be referenced by both a v4 LHS
  and a v6 LHS (the leftover text says yes; the new text says no).
- Whether the manifest emits one entry or two per call site that
  could be queried by both families.

Class B (spec internal contradiction). Spec layer. Load-bearing —
the manifest format on disk and the daemon's load-time map creation
loop both key off this decision.

## Locations

### Leftover text — Type rules paragraph

`FWL_V02_SPEC.md:322-330`:

> *Map allocation is per (call site, family) pair.*
> `BPF_MAP_TYPE_LPM_TRIE` requires a fixed key size — 32 bits for
> IPv4, 128 bits for IPv6 — so the v4 and v6 indices for a single
> `geoip(...)` call are two distinct BPF maps with two distinct map
> IDs. **A call site may allocate up to two map IDs:** one each for
> the v4 and v6 tries that are referenced by at least one `in` site
> in the program. If only `pkt.src_ip in geoip(RU)` is used (no v6
> operand), only the v4 map ID is allocated for that call; the v6
> map is neither created nor populated.

The bolded sentence is unambiguous: a single call site can carry
up to two map IDs. The follow-on phrasing "the v4 and v6 tries that
are referenced by at least one `in` site" further implies that a
single call site can be referenced by **multiple** `in` sites with
different LHS families — i.e. one `geoip(...)` textual occurrence
serving both `pkt.src_ip in geoip(...)` and
`pkt.src_ip6 in geoip(...)`.

### Post-fix text — Semantics paragraph

`FWL_V02_SPEC.md:349`:

> *Compile time.* Each textual occurrence of `geoip(...)` is a
> distinct call site. By the grammar, a `geoip(...)` call appears
> only as the right-hand side of an `... in geoip(...)` comparison,
> and a comparison has exactly one left-hand operand — so **each
> call site is bound to exactly one address family** (IPv4 if the
> LHS is `pkt.src_ip`/`pkt.dst_ip`; IPv6 if the LHS is
> `pkt.src_ip6`/`pkt.dst_ip6`). The analyser allocates **one map
> ID per call site** and emits one `BPF_MAP_TYPE_LPM_TRIE`.

`FWL_V02_SPEC.md:374`:

> There is exactly **one `map_id` per manifest entry** — the v0.2
> grammar has no syntactic surface that can bind one call site to
> two families.

### Load-time text — also one map per entry

`FWL_V02_SPEC.md:392`:

> On bundle attach, `fd` parses the bundle's `geoip.json` and, for
> each `geoip` manifest entry, calls `bpf_map_create` **exactly
> once** (the entry's `family` decides the trie's key size — 32 bits
> for `ipv4`, 128 bits for `ipv6`).

### Bundle additions paragraph — also single map_id per entry

`FWL_V02_SPEC.md:847-862`:

> A list of objects, one per `geoip(...)` call site, in source order.
> Each entry binds **one call site to exactly one address family**
> (per the geoip Semantics: each call site has a single LHS family
> from its host comparison). … `family` is `"ipv4"` or `"ipv6"` and
> determines the LPM trie's key width. `map_id` is a **single
> integer** unique across the whole program.

### Score

Three of the four paragraphs that mention map allocation say
"one family per call site, one map per call site". One paragraph
(line 322-330) says "up to two map IDs per call site, one per
family". Three vs. one majority does not make a spec consistent.

## Why this matters

### The leftover text drives a different implementation

A parser/analyser implementor who reads only the Type-rules
paragraph (line 322-330) — natural reading order, since type rules
come before semantics — will write:

- A call-site → map-id table keyed by `(call_index, family)`.
- A manifest emitter that emits one entry per `(call_index, family)`
  with prefix counts split by family.
- A bundle loader that calls `bpf_map_create` **twice** per call
  site that's referenced by both families.

An implementor who reads only the Semantics paragraph (line 349 +
374) — or who reaches the Bundle Additions paragraph (line 847-862)
— will write:

- A call-site → map-id table keyed by `call_index` alone.
- A manifest emitter that emits one entry per textual occurrence,
  with a scalar `map_id` and scalar `family`.
- A bundle loader that calls `bpf_map_create` once per entry.

These two implementations produce **incompatible bundle formats**.
An analyser written to spec A produces a manifest the daemon written
to spec B cannot load (the `family` field is missing or has the
wrong cardinality), and vice versa.

### The "referenced by at least one `in` site" phrasing is more than wrong

The leftover paragraph's "the v4 and v6 tries that are referenced
by at least one `in` site in the program" implies a global
back-reference — the analyser should walk the program, find every
`in` site that uses a given call's `geoip(...)`, and aggregate the
families. This is *only* meaningful if a single textual `geoip(...)`
occurrence can be the RHS of multiple `in` comparisons (different
call sites by source order would each get their own occurrence).

But the v0.2 grammar makes each `geoip(...)` the RHS of **exactly
one** comparison (it appears inside one `set_or_range` slot of one
`comparison` production). There is no syntactic mechanism to share
a single `geoip(...)` literal across two `in` sites — no variable
binding, no `let`, no `=` of a `set_or_range` typed local
(`set_or_range`-typed locals are not in the type table, per the
related list-typed-locals finding). So "referenced by at least one
`in` site" is computing an aggregate over a set of size exactly one
— the same `in` site that hosts the call.

The leftover text is not just contradictory with the new rule; it
describes a programming model the v0.2 surface cannot express.

### The prefix-budget-is-per-program rule is consistent only with the post-fix model

`FWL_V02_SPEC.md:398-403` says the 65 536-prefix budget is
"across all call sites and across both families" — one shared
budget. That phrasing is consistent only when a "call site" is a
single (source occurrence, family) pair (the post-fix model). If
the leftover model wins, "across all call sites and across both
families" double-counts the v4 and v6 sub-tries of a single call
site, and the budget arithmetic the analyser does at line 400-403
would diverge from the budget the daemon enforces.

### The manifest example is unambiguous about the post-fix model

The example at `FWL_V02_SPEC.md:359-372` shows two manifest
entries for the program

```python
drop if pkt.src_ip  in geoip(RU, CN)
drop if pkt.src_ip6 in geoip(RU, CN)
default allow
```

— `call_index: 0, family: "ipv4", map_id: 1` and
`call_index: 1, family: "ipv6", map_id: 2`. Each entry is one
textual `geoip(...)` occurrence. This is the post-fix model.

But the leftover paragraph's "if only `pkt.src_ip in geoip(RU)`
is used (no v6 operand), only the v4 map ID is allocated for that
call" is talking about a single call's v4-vs-v6 reference state,
which only makes sense if a call could *otherwise* be referenced
by both families. Under the post-fix model the call can only ever
be referenced by one family (its own LHS), so the conditional is
vacuous.

## Concrete impact

A `.pkt` test using the construct that's most likely to surface
this:

```fwl
@xdp(eth0)

drop if pkt.src_ip  in geoip(RU)
drop if pkt.src_ip6 in geoip(RU)
default allow
```

This hits the **two-call-sites, two-families, two-codes-list-the-same**
shape. The post-fix model produces a 2-entry manifest with
`map_id: 1` (call 0, ipv4) and `map_id: 2` (call 1, ipv6). The
leftover model could produce either:

- Two entries with two map IDs each (4 maps total), or
- Two entries with `family: ["ipv4", "ipv6"]`-shaped values, or
- Some hybrid the daemon must invent.

The corpus author who copies the manifest example at line 359-372
into a `.pkt` `expected.bundle.manifest` clause will fail any
implementation written to the leftover model — and pass any
implementation written to the post-fix model. The reverse (corpus
written to the leftover model) will pass nothing.

## Proposed fix

Pick the post-fix model (it is the model the example, the
load-time text, and the bundle additions paragraph all use). Edit
line 322-330 to drop the "up to two map IDs per call site" model
entirely and replace it with the per-call-site-bound-to-one-family
model:

> *Map allocation is per call site.* `BPF_MAP_TYPE_LPM_TRIE` requires
> a fixed key size — 32 bits for IPv4, 128 bits for IPv6 — so each
> textual `geoip(...)` occurrence allocates one BPF map of the key
> size matching its single bound family (IPv4 if the LHS is
> `pkt.src_ip`/`pkt.dst_ip`; IPv6 if the LHS is
> `pkt.src_ip6`/`pkt.dst_ip6`). A program that needs both v4 and v6
> coverage of the same country list writes two `geoip(...)` calls,
> one per family — see the example under Semantics below.

Then drop the "If only `pkt.src_ip in geoip(RU)` is used (no v6
operand), only the v4 map ID is allocated for that call" sentence
entirely — under the post-fix model it has no scenario to describe
(every call has one bound family by construction).

Alternatively, if the leftover model is the intended one (the
analyser does try to share a single `geoip(...)` across families),
the spec must:

- Add a syntactic mechanism for sharing — e.g. assigning a
  `geoip(...)` to a local: `bad_actors = geoip(RU, CN)` — and
  re-spec the type table to include `set<ipv4|ipv6>` as a local
  type.
- Replace the manifest's scalar `map_id` and `family` fields with
  `{"ipv4": <id>, "ipv6": <id>}`-shaped objects.
- Re-spec the load-time `bpf_map_create` loop to call create one or
  two times per entry depending on which families are referenced.

Either resolution closes the contradiction. Leaving it as-is forces
implementors to choose silently and breaks bundle-format
compatibility across the boundary.

## Class

Class B (spec internal contradiction, leftover text vs.
post-patch text). Spec-layer. The contradiction was introduced by
the patch that fixed the related `geoip-map-id-per-callsite-vs-…`
finding — the patch added the "one family per call site" rule but
left the "up to two map IDs per call site" sentence in place.
