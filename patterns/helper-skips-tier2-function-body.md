---
id: pattern/helper-skips-tier2-function-body
type: pattern
protocol: [tcp, udp, icmp, icmp6]
builtins: [geoip, count, rate_limit]
severity: high
layer: [emitter, interpreter, runner]
pattern_tags: [helper-skips-function-body, tier2, oracle-disagreement, v01-to-v02-extension, dogfood]
status: hypothesis
created: 2026-05-01
instance_count: 4
instances:
  - finding/2026-05-01-v02-interpreter-missing-v6-surface-activation-gate
  - finding/2026-05-01-v02-interpreter-tier2-v6-activation-helper-skips-function-body
  - finding/2026-05-01-v02-emitter-tier2-count-action-keyerror-no-slot-allocation
  - finding/2026-05-01-bpf-runner-skips-geoip-map-population
---

# helper-skips-tier2-function-body

## Description

A v0.1-era helper that walks the program (or its inputs) for some
purpose — counter-slot allocation, v6-surface activation, BPF map
population — was extended to v0.2 by adding new node types or new
features, but the helper's traversal was **not** extended to descend
into the new v0.2 input surface. The helper silently returns a
v0.1-shaped answer for v0.2-shaped programs.

The new surface in every confirmed instance is one of:

- `program.function.body` — the Tier 2 function-body AST (interpreter
  and emitter helpers had only ever walked `program.rules`).
- `case.geoip_data` — the per-`.pkt` geoip prefix table (the runner's
  `_build_map_init` had only ever consumed `state.rate_limit`).

Because the helper returns the v0.1 answer, downstream code paths
that depend on it silently misbehave: a Tier 2 v6-active program is
gated as if it were v0.1-shaped (interpreter strips v6 fields), a
Tier 2 `count` statement looks up an unallocated slot and crashes
the emitter with `KeyError`, the BPF runtime sees an empty geoip
trie regardless of the test case's data.

The defining symptom is **interpreter ↔ BPF disagreement on Tier 2
programs that exercise the new v0.2 surface in question** — Class A
in the hunting taxonomy. Three of the four confirmed instances were
discovered during the same dogfood-round-2 hunt because the dogfood
example is the first realistic Tier 2 program; v0.1-era corpus
shapes never reached the missing helpers.

## Check Strategy

1. **Grep for traversal helpers in the compiler/runtime that take
   `program: ast.Program` and iterate `program.rules`.** For every
   such helper, confirm it also walks `program.function` (when not
   `None`) using a Tier 2-aware recursive descender:
   - `IfStmt.body`, `IfStmt.elif_branches`, `IfStmt.else_body`
   - `AssignStmt.rhs`
   - Action statements (`allow`/`drop`/`log`/`count`)
   - Inner `rate_limit_call` and `geoip_call` operands
2. **Grep for runner/harness helpers that take `case: PktCase`** and
   confirm every v0.2-introduced field on `PktCase` (currently
   `geoip_data`; future `bundle_manifest`, etc.) flows into both
   oracle paths, not just the interpreter path.
3. **Diff each interpreter helper against its emitter twin.** The
   oracle-independence rule says they cannot share code, but they
   must produce the same answer on the same program. If
   `interpreter._program_touches_v6_surface` and
   `emitter._is_v6_active` walk different sets of nodes, one of them
   is a bug.
4. **Run a Tier 2 program through `fwl test` with every legal v0.2
   action statement (`allow`/`drop`/`log`/`count <n>`) and every
   typed local kind under each `if`/`elif`/`else` branch.** Any
   helper that doesn't traverse the body throws or silently
   degrades.

## Known Instances

- `finding/2026-05-01-v02-interpreter-missing-v6-surface-activation-gate`
  — interpreter had no v6-activation gate at all; `_v6_packet_gate`
  unconditionally stripped v0.1-style fields from v6-builder
  packets, so a v6-active Tier 2 program received a stripped packet.
- `finding/2026-05-01-v02-interpreter-tier2-v6-activation-helper-skips-function-body`
  — second-order: the gate helper was added but only walks
  `program.rules`, missing every Tier 2 v6 activation that lives in
  the function body. Direct cause of the dogfood interpreter ↔ BPF
  disagreement.
- `finding/2026-05-01-v02-emitter-tier2-count-action-keyerror-no-slot-allocation`
  — `_allocate_counter_slots(program)` walks `program.rules` only
  and never sees Tier 2 `count <name>` statements; emitter crashes
  with `KeyError` on every Tier 2 program that uses `count` (the
  spec's own SSH-brute-force pattern).
- `finding/2026-05-01-bpf-runner-skips-geoip-map-population` — the
  `.pkt` runner's `_build_map_init` only translates
  `state.rate_limit` into BPF map writes; `case.geoip_data` is
  consumed by the interpreter but never reaches the BPF runtime, so
  every BPF run sees an empty geoip trie regardless of the test
  case's prefix table.

## Where to Look Next

These are the v0.2 helpers that have not yet been audited for
Tier 2-body and `case.geoip_data` traversal. Each is a candidate for
a finding by the same shape:

- **Analyzer passes that walk `program.rules`** — `fwl/fwl/analyzer.py`.
  The dominator-analysis pass in particular: if it walks rules-only
  it accepts unsound Tier 2 programs (statement-position pkt reads
  on the wrong protocol). Cross-reference with pattern
  `tier2-dominator-rule-coverage-holes`.
- **`_collect_rate_limit_call_sites` / equivalents in the emitter**.
  v0.2 adds a new syntactic position for `rate_limit_call`: as a
  bool-valued primary inside a Tier 2 `if` condition. If any helper
  collects rate-limit call sites by walking only rule modifiers, it
  misses Tier 2 ones — same shape as the counter-slot bug.
- **The `geoip` analyzer's call-site enumerator.** Tier 2 admits
  `geoip(...)` inside a Tier 2 condition (per
  `FWL_V02_SPEC.md:1095-1099`); a v0.1-shaped enumerator that walks
  rule conditions only would silently miss Tier 2 geoip calls.
  Symptom: missing maps in the manifest; the BPF object loads but
  every Tier 2 geoip lookup misses.
- **The `fd` daemon's bundle-attach path.** The `.pkt` runner is the
  test analogue of `fd`'s bundle-attach. The runner finding showed
  the test path missed geoip; the daemon path has its own
  population logic that's never been hunted.
- **Stack-budget estimator.** Tier 2 introduces locals with stack
  cost; the v0.1 estimator counted only spilled-rule cost. If any
  helper sums up stack from rules-only it under-reports for Tier 2
  programs and the verifier rejects programs the spec's stack
  budget claims are within budget.
- **Counter-name reservation enumerator.** Same shape as the
  counter-slot allocator — anywhere the compiler enumerates user
  counter names (for the manifest, for `fctl counters`, for the
  ring-buffer log schema), if it walks `program.rules` only it
  silently drops Tier 2 counters.
- **Symmetric: harness-layer extensions for any future v0.2/v0.3
  `PktCase` field.** The runner's pattern of "interpreter consumes
  field X, BPF path silently ignores it" is mechanical. Any new
  oracle input added to `PktCase` should be wired into both oracles
  in the same PR; otherwise the test harness lies about agreement.
