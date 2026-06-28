---
type: meta
title: Phase 2 Gate 2 — Construct 2 (geoip() built-in) sign-off
date: 2026-05-01
agent: f-phase2 implementation agent
related_findings: 1  # hunt finding status:fixed
---

# Gate 2 sign-off — Construct 2 (geoip() built-in)

## Outcome

`geoip(<ALPHA2>, ...)` variadic built-in, RHS of `in` only, valid against both `pkt.src_ip`/`pkt.dst_ip` (v4) and `pkt.src_ip6`/`pkt.dst_ip6` (v6). Family is bound at analysis time from the LHS field type. Each call site gets its own LPM trie + `__always_inline` lookup helper in BPF; the interpreter resolves call sites against `case.geoip_data` and memoises the per-call prefix list.

## Per-construct sign-off checklist (CONSTRUCTS.md)

- [x] **iso3166.py**: 249-entry alpha-2 frozenset; analyzer rejects unknown codes.
- [x] **Grammar / AST / parser / analyzer / interpreter / emitter / pkt / runner**: all updated. The runner update was to populate the LPM trie maps before `BPF_PROG_TEST_RUN` (see hunt finding below). `GeoIp` is the one non-frozen AST node — analyzer fills `call_index` and `family` post-parse.
- [x] **Per-construct corpus** under `f/fwl/tests/corpus/09_geoip/` — 24 cases (8 pass, 4 fail, 6 edge, 6 interaction). All 24 pass under `hone regress` (interpreter), 24/24 pass under `sudo hone regress` (interpreter + BPF).
- [x] **`hone regress` (no root, BPF=skip)** green on the full f/fwl/tests/corpus tree: **109/109**. **`sudo hone regress`** with BPF oracle live: **109/109** with `bpf=pass` on every case. Whole-base kb corpus under sudo: 310/318 — the 8 failures are the same pre-existing v0.1 loader-bug findings carried over from earlier gates; none are Construct 2 regressions.
- [x] **`hone fuzz boundary_probing`** (count=1000, seed=0): 0 divergent / 42 candidates / 0 compile_failed.
- [x] **`hone fuzz oracle_divergence`** seeds 0+1 (count=1000 each): **0 divergent** / 2000 candidates / 0 compile_failed. Combined runner_error count is from the strategy generating non-`@xdp` snippets the BPF runner can't load — expected, not a Construct 2 issue.
- [x] **`hone hunt --target ../f/fwl/examples/geoip_block.fw --max-turns 80 --model claude-opus-4-7`** (115 turns, $8.04): **1 high-severity finding, status: fixed.**
- [x] **`hone mutate` + `hone transfer`** for the confirmed finding: mutate 13/13 agreed (0 divergent), transfer 1/1 agreed (0 divergent).
- [x] **`hone tune`**: weights healthy. boundary_probing=0.243, oracle_divergence=0.243, hunt=0.220, mutate=0.147, transfer=0.147. Hunt is 5/5 hits (every hunt round produced findings); other strategies have 0 hits this gate but stay at the 5% floor via UCB1.
- [x] **Construct map**: `f-hone/config/compiler_construct_map.yaml` updated. Added `geoip` tag to `interpreter.py`, `emitter.py`, `pkt.py`, `runner.py`; new entry `fwl/fwl/iso3166.py: [geoip]`; `PKT_V02_SPEC.md` now also tagged with `geoip`. Diff-impact verification deferred — the FWL repo's v0.2 work is uncommitted; commits are operator-driven.

## Hunt finding (fixed in this gate)

**`finding/2026-05-01-bpf-runner-skips-geoip-map-population`** (high; layer: runner). The `.pkt` test runner only populated `fwl_rl_map_<idx>` maps from `case.state` blocks; it never wrote the geoip LPM trie maps (`fwl_geoip_<call_index>`) the emitter declared. Result: every BPF run saw empty geoip tries, every `pkt.<ipfield> in geoip(...)` lookup returned "miss", and the BPF oracle disagreed with the interpreter on every program where a geoip lookup drives the action. Without root the BPF oracle skipped, masking the divergence; with root the entire `09_geoip/` corpus failed `bpf`.

Fix: added `_build_geoip_map_init` in `f/fwl/fwl/runner.py:286-336`. Walks every `geoip(...)` call site in the program, pairs each call's `call_index` and `family` with the case's `geoip_data`, and emits a `{map_name: {key_bytes: value_bytes}}` block matching the LPM trie key layout the emitter declares (v4: `<I` prefixlen + 4 NBO bytes; v6: `<I` prefixlen + 16 NBO bytes; value: single membership byte). The result is merged into the existing `map_init` passed to `bpf_runner.run`. Verified: `sudo hone regress --corpus ../f/fwl/tests/corpus/` flips from 99 pass + 10 BPF fails → 109/109 with `bpf=pass`. The two reproducer .pkts under `corpus/from_hunt/2026-05-01_geoip_block_oracle_skew/` go BPF-fail → BPF-pass.

## Hunt miss (ruled out, recorded for future hunts)

**`miss/2026-05-01-geoip-block-fw-program-semantics`** — covers H1–H5 hypotheses against `geoip_block.fw`. H5 (cross-family `pkt.src_ip == 0.0.0.0` matching v6 frames where `src_ip` defaults to 0) is a real cross-family pseudobug at the emitter prelude layer, but the program shape doesn't surface it (no v4 IP equality rules; geoip data does not contain `0.0.0.0/0`). Filed as a separate hypothesis for the Construct 3 (Tier 2) hunt — Tier 2 statement-position reads have a more general dominator rule that may interact with this.

## What landed in code

- `f/fwl/fwl/iso3166.py` (new) — 249 ALPHA2_CODES frozenset.
- `f/fwl/fwl/grammar.lark` — `GEOIP` keyword, `CC_CODE` token, `geoip_call` production.
- `f/fwl/fwl/ast.py` — `GeoIp` AST node (mutable; `codes`, `call_index`, `family`, `span`).
- `f/fwl/fwl/parser.py` — `geoip_call` transformer producing `GeoIp(call_index=-1)`.
- `f/fwl/fwl/analyzer.py` — `_assign_geoip_call_indices` source-order pre-walk + `_bind_geoip` (validates against ALPHA2_CODES, dedupes, stamps family from LHS).
- `f/fwl/fwl/interpreter.py` — `_Ctx`, `_resolve_geoip_v4/v6` (memoised), `_geoip_match_v4/v6` (LPM-style scan).
- `f/fwl/fwl/emitter.py` — `_collect_geoip_calls`, `_emit_geoip_maps_and_helpers` (BPF_MAP_TYPE_LPM_TRIE + `__always_inline` helpers per call site, distinct v4/v6 key structs).
- `f/fwl/fwl/pkt.py` — `geoip_data` field on `PktCase`, parsed from the `geoip_data:` block in YAML.
- `f/fwl/fwl/runner.py` — `_build_geoip_map_init` (added in this gate's hunt-fix step).
- `f/fwl/examples/geoip_block.fw` (new) — Construct 2 dogfood example. v4+v6 blocklist + allowlist + service allow + `default drop`. 4 LPM trie maps + 4 helpers in emitted C.
- `f/fwl/tests/corpus/09_geoip/` (new) — 24 .pkt cases.
- `f-hone/config/compiler_construct_map.yaml` — geoip tags added; `iso3166.py` entry created.

## Sign-off

Gate 2 for Construct 2 (geoip()) — **clear**. Construct 3 (Tier 2 programmable functions) is unblocked.
