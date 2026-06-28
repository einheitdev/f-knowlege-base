---
id: finding/2026-05-01-v02-spec-geoip-map-id-per-callsite-vs-per-callsite-family-pair
type: finding
protocol: []
builtins: [geoip]
severity: high
layer: spec
pattern_tags: [spec-ambiguity, geoip, bundle-format, manifest]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: []
---

# 2026-05-01-v02-spec-geoip-map-id-per-callsite-vs-per-callsite-family-pair

## Summary

`FWL_V02_SPEC.md` specifies the `geoip()` map allocation rule three
times, and the three statements are mutually inconsistent. The
analyzer can read the spec to mean three different things, and the
manifest format the daemon consumes is correspondingly under-defined.

Class B (analyzer/spec inconsistency, with downstream impact on the
bundle wire format).

## Locations and the three readings

### Reading A — one map ID per call site (`FWL_V02_SPEC.md:315-316`)

> *Compile time.* Every `geoip(...)` call site is assigned a unique
> map ID by the analyser.

This says **one** map ID per call site, regardless of family.

### Reading B — one BPF map per (call site × family) (`FWL_V02_SPEC.md:316-318`)

> The compiler emits one `BPF_MAP_TYPE_LPM_TRIE` per (call site,
> family) pair that is actually queried.

This says **up to two** physical maps per call site (one v4 trie,
one v6 trie), since `BPF_MAP_TYPE_LPM_TRIE` requires a fixed
key size — 32 bits for IPv4, 128 bits for IPv6 — so the v4 trie and
v6 trie cannot share a single BPF map object.

### Reading C — manifest example shows one `map_id` listing two families (`FWL_V02_SPEC.md:779-784`)

```json
"geoip": [
  {"map_id": 1,
   "codes": ["RU", "CN"],
   "families": ["ipv4", "ipv6"],
   "prefix_count": {"ipv4": 12345, "ipv6": 2345}}
]
```

The manifest entry has a single scalar `map_id: 1` covering **both**
`ipv4` and `ipv6`. Combined with Reading B, this implies map_id `1`
is a *logical* identifier that the daemon expands into two physical
maps. But that scheme is never spelled out, and `FWL_V02_SPEC.md:794-795`
then says:

> `fd` reads the manifest, then for each entry calls `bpf_map_create`
> with the map ID's reserved type (`BPF_MAP_TYPE_LPM_TRIE`)…

— singular `bpf_map_create` per entry, contradicting the
"two physical maps under one logical ID" reading.

## Possible resolutions, each requiring a different impl

| Resolution | What changes |
|---|---|
| Reading A wins: one map per call site, regardless of family | The (call site, family) sentence is wrong; need a different LPM scheme (e.g. v4-mapped-into-v6 keys), with all the cross-family equivalence questions that v0.2 explicitly defers (`FWL_V02_SPEC.md:185-189`, `957-961`). |
| Reading B wins: two physical maps per call site | Manifest must carry **two** `map_id`s per entry, or the scalar `map_id` must be replaced by `{"ipv4": <id>, "ipv6": <id>}`. `bpf_map_create` is called twice per entry. |
| Hybrid: one logical `map_id`, daemon expands to two physical maps | Spec must add a section explaining the logical-to-physical mapping, and how the BPF program references the two physical maps (BPF object code can't dispatch on a logical ID at runtime). |

Until the spec picks one, two implementors of the bundle format
will disagree, and the manifest the analyser writes will not be
loadable by the daemon (or vice-versa).

## Why this matters

The compile-error table at `FWL_V02_SPEC.md:408-418` includes
`error: geoip prefix budget exceeded: <N>; v0.2 limit is 65536`.
Whether the cap counts v4 and v6 prefixes as one budget or two
depends on which reading wins. The spec at `FWL_V02_SPEC.md:351-354`
says "across all call sites and across both families" — so one
budget. That reading is consistent only with Reading A or with the
hybrid; it is *inconsistent* with strict Reading B if Reading B
expects each (call site × family) map to have its own budget.

This is a load-bearing ambiguity: the bundle format, the analyser's
manifest emission, and the daemon's load path all key off this
decision.

## Proposed fix

Pick Reading B + manifest-format adjustment (cleanest mapping to
BPF map semantics). Then:

- Reword `FWL_V02_SPEC.md:315-316` to say "Every (call site, family)
  pair that is actually queried is assigned a unique map ID by the
  analyser."
- Replace the scalar `map_id` in the manifest example with
  `{"ipv4": <id>, "ipv6": <id>}` (with either field omitted when
  the family is not queried).
- Spell out the bundle-time and load-time consequence: two
  `bpf_map_create` calls per entry; manifest carries two map IDs
  when both families are queried.
