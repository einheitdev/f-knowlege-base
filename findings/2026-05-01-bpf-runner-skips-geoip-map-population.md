---
id: finding/2026-05-01-bpf-runner-skips-geoip-map-population
type: finding
protocol: [tcp, udp, icmp, icmp6]
builtins: [geoip]
severity: high
layer: runner
pattern_tags: [oracle-skew, bpf-map-population, geoip, test-harness, silent-pass]
status: fixed
source_file: fwl/fwl/runner.py
created: 2026-05-01
fixed: 2026-05-01
pkt_path: corpus/from_hunt/2026-05-01_geoip_block_oracle_skew/
related: []
---

## Fix applied

Added `_build_geoip_map_init` in `fwl/fwl/runner.py`. Walks every
`geoip(...)` call in the program, pairs each call's `call_index` and
`family` with the case's `geoip_data`, and emits a
`{map_name: {key_bytes: value_bytes}}` block matching the LPM trie
key layout the emitter declares. The result is merged into the
existing `map_init` passed to `bpf_runner.run`. Verified by running
the whole corpus (`hone regress --corpus ../f/fwl/tests/corpus/`)
with `sudo` so BPF_PROG_TEST_RUN runs under root: 109/109 cases
pass with `bpf=pass` (was 99 pass + 10 BPF fails before the fix).
The two reproducer .pkts under
`corpus/from_hunt/2026-05-01_geoip_block_oracle_skew/` flip from
BPF fail → BPF pass.


# 2026-05-01-bpf-runner-skips-geoip-map-population

## Summary

The `.pkt` test runner (`fwl/fwl/runner.py:201-233`,
`_build_map_init`) only translates `state.rate_limit` blocks into
BPF map writes; it does not populate the geoip LPM trie maps
(`fwl_geoip_<call_index>`) that the emitter declares for every
`geoip(...)` call site. As a result, every BPF run sees empty
geoip tries — every `pkt.<ipfield> in geoip(...)` lookup returns
"miss", regardless of the `.pkt`'s `geoip_data:` block.

The interpreter consults `case.geoip_data` correctly
(`fwl/fwl/interpreter.py:35` and `_resolve_geoip_v4` /
`_resolve_geoip_v6`). The two oracles therefore disagree on every
program where a geoip lookup affects the action — interpreter
matches, BPF misses, and the action diverges.

This is a Class-A divergence (interpreter vs BPF on the same packet)
surfaced when the runner is executed with privileges that allow
BPF_PROG_TEST_RUN. Without privileges (the default for `fwl test`
in CI/dev), the BPF oracle is `skip`'d and the divergence is
hidden — the corpus appears green for the wrong reason.

## Hypothesis

For `fwl/examples/geoip_block.fw`, send a v4 packet from a Russian
prefix (5.8.1.1) to dst_port 80. Per the program:

  rule 0: `drop if pkt.src_ip in geoip(RU, CN, KP)` — matches RU
  rule 4: `allow if pkt.proto == tcp and pkt.dst_port in [80, 443]`

Interpreter resolves `geoip(RU)` via `case.geoip_data`, hits the
prefix `5.8.0.0/16`, fires rule 0 → `XDP_DROP`. BPF's
`fwl_geoip_0` LPM trie was never populated, so the lookup misses;
control falls through to rule 4, which matches → `XDP_PASS`.

Symmetrically for v6: a v6 packet from US (in the geoip(US) v6
list) to a port that no other allow rule covers (e.g. 22) should be
allowed by rule 3 (`allow if pkt.src_ip6 in geoip(US, DE)`).
Interpreter allows, BPF reaches `default drop` and drops.

## Test cases

Two `.pkt` cases under
`corpus/from_hunt/2026-05-01_geoip_block_oracle_skew/`:

| .pkt case | interpreter | BPF | spec-expected |
|---|---|---|---|
| `geoip_block_v4_ru_dst80.pkt` | drop | **pass** | drop |
| `geoip_block_v6_us_dst22.pkt` | pass | **drop** | pass |

Both fail the BPF oracle when run with `sudo -n fwl test
.../from_hunt/2026-05-01_geoip_block_oracle_skew`. They reproduce
the same divergence the existing `fwl/tests/corpus/09_geoip/`
suite shows under root: ten cases there fail BPF for the same
reason (e.g. `pass_v4_match_drops.pkt`,
`pass_v6_match_drops.pkt`, `pass_allowlist_two_countries.pkt`,
`edge_exact_prefix_boundary.pkt`,
`edge_duplicate_codes_dedup.pkt`, etc.). The divergence is
universal across the geoip corpus, not specific to one program.

## Root cause

`fwl/fwl/runner.py:201-233`:

```python
def _build_map_init(
  program: ast.Program,
  state: dict[int, dict[Any, int]],
) -> dict[str, dict[bytes, bytes]]:
  if not state:
    return {}
  result: dict[str, dict[bytes, bytes]] = {}
  ...
  for rule_idx, buckets in state.items():
    ...
    map_name = f"fwl_rl_map_{rule_idx}"
    ...
```

Two structural problems:

1. The early-out `if not state: return {}` short-circuits before
   geoip considers populating anything. Even when `state` is
   present, the loop body only ever writes `fwl_rl_map_<idx>`
   maps — never `fwl_geoip_<idx>`.
2. The function takes `state` (rate_limit only) as input; the
   caller (`_bpf_oracle`, line 144) never passes `case.geoip_data`
   into the BPF path at all. The geoip data flows only into the
   interpreter (line 105: `geoip_data=(case.geoip_data or None)`).

The BPF emitter declares the map correctly
(`fwl/fwl/emitter.py:777-849`, `_emit_geoip_maps_and_helpers`) —
the LPM trie exists in the loaded BPF object with the right
key/value sizes. The runner just never writes any prefixes to it.

## What this rules out

- Class A divergence in the parser, analyzer, interpreter, or
  emitter on geoip semantics themselves. Each component handles
  geoip correctly; the divergence is at the **harness** layer that
  wires the two oracles together.
- Class B in the analyzer. The analyzer correctly assigns
  `call_index` and `family` per spec.
- Spec ambiguity on `geoip()` runtime semantics. The spec is
  clear: the daemon populates the LPM trie at bundle attach time
  from `geoip.json`. The `.pkt` runner is the test analogue of
  the daemon and must do the equivalent.

## What this hunt did NOT cover

- Whether `fd` (the orchestrator daemon) correctly populates the
  trie at bundle attach. The .pkt runner is a separate path; the
  daemon path needs its own hunt.
- Whether any geoip case under `09_geoip/` carries
  `expected.bpf_action` values that *also* happen to be wrong for
  unrelated reasons (the BPF "miss" coincidentally matches the
  interpreter's verdict for some specific packet/program shapes —
  e.g. `geoip_block_v4_ru_drop.pkt` in the same hunt directory
  passes because both oracles end at the `default drop` rule, by
  coincidence).
- Whether the runner should skip the BPF oracle outright when
  geoip data is present (a softer fix that avoids the
  silent-passing-with-skips problem) instead of populating the
  map. The cleanest fix is to populate; a stop-gap is to skip.

## Severity

High. The corpus contains 10+ geoip cases that fail BPF when run
with privileges. In dev/CI without root the failures are masked
as `skip`; the corpus appears green and corpus authors believe
their geoip programs are working. Any user who later runs `fwl
test` with root (or attaches the program in production via the
daemon) will see the divergence — but the production path uses
`fd`, which has its own population logic, so the bug surfaces
mostly in the test harness. Still: the test harness lying about
agreement is the highest-impact class of test-infra bug.

The dogfood case for v0.2 (`fwl/examples/geoip_block.fw`) is
directly affected — that program's BPF behaviour cannot be
trusted by a `.pkt`-based test today.

## Fix

In `fwl/fwl/runner.py`:

1. Extend `_build_map_init` (or add a sibling `_build_geoip_map_init`)
   to walk every `geoip(...)` call site in the program, look up the
   call's `family` and `call_index`, and translate the
   `case.geoip_data[code]` CIDRs into `fwl_geoip_<call_index>` map
   entries.
2. The LPM trie key is `struct { __u32 prefixlen; <ip_bytes>; }`
   — see `fwl/fwl/emitter.py:794-797` (v4) and `:817-820` (v6).
   Pack each CIDR's prefix bytes (network byte order) and prefix
   length into a key, with value `(__u8)1`.
3. Wire the geoip map_init into the existing `map_init` dict
   passed to `bpf_runner.run`. The runner's `_populate_map`
   already does the BPF map update.

Stop-gap (less work, more brittle): make `_bpf_oracle` skip with
a clear reason when `case.geoip_data` is non-empty, mirroring the
existing skip path for unimplemented `truncate_to`. The corpus's
`09_geoip` directory would all show `skip` instead of `fail` — at
least no false green. Long-term the population is what we want.
