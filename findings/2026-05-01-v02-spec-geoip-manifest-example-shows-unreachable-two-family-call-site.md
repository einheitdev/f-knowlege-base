---
id: finding/2026-05-01-v02-spec-geoip-manifest-example-shows-unreachable-two-family-call-site
type: finding
protocol: []
builtins: [geoip]
severity: medium
layer: spec
pattern_tags: [spec-inconsistency, geoip, manifest, example-vs-rule, bundle-format]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: [finding/2026-05-01-v02-spec-geoip-map-id-per-callsite-vs-per-callsite-family-pair]
---

# 2026-05-01-v02-spec-geoip-manifest-example-shows-unreachable-two-family-call-site

## Summary

The geoip-section *Semantics* paragraph illustrates the manifest
schema with a worked example that shows a single `call_index` whose
`maps` field carries **both** an `ipv4` and an `ipv6` entry. Per the
rest of the spec — specifically the rule at
`FWL_V02_SPEC.md:421-425` that each textual occurrence of
`geoip(...)` is its own call site, plus the syntactic constraint
that each `... in geoip(...)` comparison binds the call site to
exactly one LHS operand of a single family — that worked example is
unreachable. No v0.2 program can produce a manifest entry of that
shape.

The example will mislead implementers building the analyser/manifest
emitter and the daemon's load path: both will write code paths for a
state that the language cannot generate.

Class B (spec example contradicts spec rule). Spec-layer.

## The unreachable example

`FWL_V02_SPEC.md:330-338`:

```json
"geoip": [
  {"call_index": 0,
   "codes": ["RU", "CN"],
   "maps": {"ipv4": 1, "ipv6": 2},
   "prefix_count": {"ipv4": 12345, "ipv6": 2345}},
  ...
]
```

`call_index: 0`, `maps: {"ipv4": 1, "ipv6": 2}` — one call site,
two physical maps, one per family.

The same shape recurs in the Bundle additions reference at
`FWL_V02_SPEC.md:825-833`. Both spots present a two-family entry
under one `call_index` as a normal, expected manifest shape.

## Why no v0.2 program can produce this entry

### Rule 1 — call_index is per textual occurrence

`FWL_V02_SPEC.md:340-341`:

> `call_index` is the zero-based source-order index of the
> `geoip(...)` call.

`FWL_V02_SPEC.md:421-425`:

> *Same code referenced twice in one program.* `if pkt.src_ip in
> geoip(RU)` and `if pkt.dst_ip in geoip(RU)` — the analyser
> allocates **two** map IDs (one per call site). The daemon
> populates both tries with the same prefix data. This is wasteful
> but correct; per-call-site maps are simpler than a global cache
> and avoid cross-rule lifetime issues. v0.3 may de-dupe.

Two textual occurrences of `geoip(RU)` → two `call_index` entries.
The analyser does **not** deduplicate by argument list.

### Rule 2 — each call site has exactly one LHS operand

The grammar at `FWL_V02_SPEC.md:1140-1152` forces every
`geoip_call` to appear as the RHS of one `... in ...` comparison:

```ebnf
set_or_range  = list | range | cidr | cidr_list | geoip_call ;
geoip_call    = "geoip" "(" cc_code { "," cc_code } ")" ;
```

`comparison = lvalue "in" set_or_range ;` (`FWL_V02_SPEC.md:1089-1090`).

A `comparison` has one `lvalue`. The `lvalue` chooses the family
(`pkt.src_ip` / `pkt.dst_ip` ⇒ ipv4; `pkt.src_ip6` / `pkt.dst_ip6`
⇒ ipv6). So each call site is bound to exactly one family.

There is no way to share a single textual `geoip(...)` between two
`in` expressions:

- `geoip(...)` is not assignable (`expression` admits neither
  `geoip_call` nor an opaque-set type — `FWL_V02_SPEC.md:1110-1115`).
- `geoip(...)` is not nestable (`FWL_V02_SPEC.md:309-311`).
- The grammar admits `geoip_call` only inside `set_or_range`, and a
  `set_or_range` is the RHS of exactly one comparison.

### Conjunction → unreachable

Combine the two rules:

- A single call site is one textual occurrence of `geoip(...)`.
- One textual occurrence is the RHS of exactly one comparison.
- That one comparison has exactly one LHS operand of one family.
- ∴ Each call site has exactly one family.
- ∴ No call site can have both `ipv4` and `ipv6` keys under `maps`.

So `{"call_index": 0, "maps": {"ipv4": 1, "ipv6": 2}, ...}` is a
manifest entry no analyser can ever emit.

## Why this matters

The earlier finding
`finding/2026-05-01-v02-spec-geoip-map-id-per-callsite-vs-per-callsite-family-pair`
(status: fixed) resolved a three-way ambiguity by picking
"per (call site × family)" as the allocation rule. That fix is
correct in spirit — `BPF_MAP_TYPE_LPM_TRIE` does require separate
maps for v4 and v6 keys — but the resulting spec text was written
as if the language allowed one call site to span two families. It
doesn't.

The downstream cost:

1. **Analyser implementer** writes a `dict[Family, MapId]` per
   call site. They populate both keys for the example at
   line 330-338, then realise no input program ever exercises that
   path. Wasted code, dead branches, brittle tests.

2. **Daemon implementer** writes the load loop at line 362-371:

   > calls `bpf_map_create` once per allocated family **(zero, one,
   > or two calls per entry depending on which `maps` keys are
   > present)** …

   "Zero" applies when the warning at line 326-327 fires (call
   site never queried — orphan). "Two" never applies. The "two"
   branch is dead code unless v0.3 introduces argument-list dedup.

3. **`.pkt` test author** writing a corpus case for "geoip with
   both v4 and v6 sites" naturally writes:

   ```
   drop if pkt.src_ip  in geoip(RU)
   drop if pkt.src_ip6 in geoip(RU)
   ```

   and expects manifest `geoip[0].maps == {"ipv4": ..., "ipv6": ...}`.
   The actual analyser correctly emits *two* entries (one per
   family). The test fails. The test author re-reads the spec,
   sees the example at line 330-338, and concludes the analyser is
   buggy when it isn't.

4. The Cross-family geoip example at `FWL_V02_SPEC.md:481-487`:

   ```python
   drop if pkt.src_ip  in geoip(RU, CN)
   drop if pkt.src_ip6 in geoip(RU, CN)
   default allow
   ```

   produces a manifest with **two** entries:

   ```json
   "geoip": [
     {"call_index": 0, "codes": ["RU","CN"], "maps": {"ipv4": 1}, ...},
     {"call_index": 1, "codes": ["RU","CN"], "maps": {"ipv6": 2}, ...}
   ]
   ```

   not one entry with both keys.

## Proposed fix

Two consistent options:

### Option A — keep call-site = textual-occurrence; fix the example

Replace `FWL_V02_SPEC.md:330-338` with a manifest example that
matches what the analyser actually emits. The cross-family
program at `FWL_V02_SPEC.md:481-487` is the natural source:

```json
"geoip": [
  {"call_index": 0,
   "codes": ["RU", "CN"],
   "maps": {"ipv4": 1},
   "prefix_count": {"ipv4": 12345}},
  {"call_index": 1,
   "codes": ["RU", "CN"],
   "maps": {"ipv6": 2},
   "prefix_count": {"ipv6": 2345}}
]
```

Then revise the prose at `FWL_V02_SPEC.md:341-344`:

> `call_index` is the zero-based source-order index of the
> `geoip(...)` call. **Each call site is bound to exactly one
> family by its host comparison's LHS operand**, so `maps` always
> has exactly one entry — `ipv4` for `pkt.src_ip`/`pkt.dst_ip` LHS,
> `ipv6` for `pkt.src_ip6`/`pkt.dst_ip6` LHS. Map IDs are unique
> across the whole program.

And drop "zero, one, or two calls per entry" at line 362-365 in
favour of "exactly one `bpf_map_create` per manifest entry".

### Option B — introduce argument-list dedup

Spec a rule at the analyser pass: "geoip(...) calls with the same
sorted argument list collapse into one call site". Then the
example at line 330-338 becomes reachable: the cross-family
program above produces one entry with both family keys.

Cost: line 421-425 must be reversed (no longer "two map IDs per
call site for the same code"), and the wasted-prefix-budget
behaviour the spec currently accepts ("wasteful but correct")
goes away — at the cost of more implementation complexity in the
analyser. v0.3 was already slated to do this.

### Recommendation

Option A. v0.2 ships sooner, the dead-code branches go away, and
v0.3 can pick up Option B as planned without a manifest format
change (a reader can collapse two entries with identical sorted
codes into one logical call site at parse-time if dedup ever
matters).

## Class

Class B (spec example contradicts spec rule). Spec-layer.
