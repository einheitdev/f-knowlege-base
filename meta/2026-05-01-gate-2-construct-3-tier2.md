---
type: meta
title: Phase 2 Gate 2 — Construct 3 (Tier 2 programmable functions) sign-off
date: 2026-05-01
agent: f-phase2 implementation agent
related_findings: 3  # all hunt findings status:fixed
---

# Gate 2 sign-off — Construct 3 (Tier 2 programmable functions)

## Outcome

`def firewall(pkt):` body — `if/elif/else`, locals, `allow`/`drop`/`log`/`count <name>` statements, `rate_limit(...)` as an if-condition primary — all shipped. The grammar is indentation-aware via a custom `FwlIndenter` postlex that gates indent-tracking on the `def` keyword, so Tier 1 programs (line-oriented, `limited by ...` continuation tolerated) parse unchanged while Tier 2 emits proper `_INDENT`/`_DEDENT` for blocks.

## Per-construct sign-off checklist (CONSTRUCTS.md)

- [x] **Grammar** — `function_def`, `block`, `if_stmt`/`elif_clause`/`else_clause`, `assign_stmt`, `action_stmt` (allow/drop/log/count); `rate_limit_call` as a primary; `bool_primary` admits IDENTIFIER (bool local); `value_compare` extended to `lvalue/rvalue` for locals on either side; `bare_field` primary lifts `value_field` so `port = pkt.dst_port` parses as a scalar_expr. The Indenter postlex (`FwlIndenter`) gates indentation tracking on `DEF` so Tier 1 stays line-oriented.
- [x] **AST** — `LocalType` enum (bool/u16/u32/ipv4/ipv6/proto), `LocalRead`, `RateLimitCall`, `AssignStmt`, `ActionStmt`, `IfStmt`, `FunctionDef`. `Program.function: FunctionDef | None` co-exists with `rules: list[Rule]` and `default: DefaultRule | None`; mutual exclusion enforced at the analyzer.
- [x] **Parser** — Tier 2 transformers + the `bare_field` primary + `lvalue_field`/`lvalue_local`/`rvalue_operand`/`rvalue_field`/`rvalue_local`. Per-default-rule placement check moved to the program transformer (the new `body_item*` grammar permits any source order, but rules-after-default raise the spec's "default must be last" error).
- [x] **Analyzer** — Tier 1 vs Tier 2 mutual-exclusion check; Tier 2 sub-pass `_analyze_tier2` walks the function body with:
  - source-order first-assignment local type binding;
  - reassignment type-equality check ("bound as T1; cannot reassign as T2");
  - `_check_dominator` for bare-field reads (port/flag/v4/v6/proto) — polarity-aware via `_established_by`/`_comparison_guards`;
  - reachability check that errors on statements after a fully-terminating control-flow chain;
  - stack-budget estimator that errors at ≥450 B per spec;
  - `'pkt'` rejected as a local name;
  - `RateLimitCall` validated only inside if-conditions; rate_limit `per=` field dominator-checked.
- [x] **Interpreter** — `_exec_tier2`/`_exec_stmts`/`_eval_scalar`/`_eval_tier2_comparison` with locals dict, terminal-action propagation, if/elif/else short-circuiting. Field reads normalised on read (`proto` → `Proto` enum, IPv4 → int, IPv6 → int).
- [x] **Emitter** — Tier 2 body emission: per-local C variable declarations (with hi/lo split for IPv6 locals), `if/else if/else` chains, terminal actions emit `return XDP_PASS`/`return XDP_DROP`, log/count side-effects reuse the existing `_emit_log`/`_emit_count` helpers. Geoip and counter-slot allocation extended to walk `program.function.body`.
- [x] **Per-construct corpus** under `f/fwl/tests/corpus/10_tier2_functions/` — 24 cases (covering minimal, if/elif/else, locals, v6 + Tier 2, dominator-positive and dominator-negative, count action, type errors, mix-rejection). All 24 pass under `hone regress` (interpreter), 24/24 pass under `sudo hone regress` with BPF oracle live.
- [x] **`hone regress` whole f/fwl/tests/corpus** — **133/133** under sudo. The pre-existing 8 v0.1 loader-bug failures are not in this tree (they live under f-knowlege-base/corpus only).
- [x] **`hone fuzz boundary_probing`**: 0 divergent / 42 candidates.
- [x] **`hone fuzz oracle_divergence`** seed 0 (count=1000): 0 divergent.
- [x] **`hone hunt --target ../f/fwl/examples/dogfood_v02.fw`** rounds 1+2 (max-turns 80, opus-4-7): **3 high-severity findings, all status: fixed.**
- [x] **`hone mutate` + `hone transfer`** for every confirmed finding: 0 new divergences.
- [x] **`hone tune`**: weights healthy. boundary_probing=0.243, oracle_divergence=0.243, hunt=0.220, mutate=0.147, transfer=0.147 (no change since Construct 2's tune since hits stayed in `hunt`).
- [x] **Construct map**: `fwl/fwl/runner.py`, `fwl/fwl/interpreter.py`, `fwl/fwl/emitter.py` all gained the `tier2` tag. Diff-impact deferred — branch is uncommitted; commits are operator-driven.
- [x] **flake8 clean** on `f/fwl/fwl/`.
- [x] **pytest**: 457/457 green.

## Hunt findings (all fixed in this gate)

1. **`finding/2026-05-01-v02-interpreter-tier2-v6-activation-helper-skips-function-body`** (high; layer: interpreter). The interpreter's `_program_touches_v6_surface` walked Tier 1 rules only and never inspected `program.function.body`, so a Tier 2 program activating v6 via the function body falsely failed the gate. v6 packets on Tier 2 dogfood-shaped programs would hit the v0.1-style field strip and silently fall through. Fix: extended `_program_touches_v6_surface` to walk the function body via a new `_stmts_touch_v6`. `_condition_touches_v6` also gained `FieldRef` and `Ipv6Literal` branches so Tier 2 `addr = pkt.src_ip6` and `addr = ::1` shapes activate correctly. Pinned at `fwl/tests/corpus/10_tier2_functions/edge_v6_activation_via_function_body.pkt`.

2. **`finding/2026-05-01-v02-emitter-tier2-count-action-keyerror-no-slot-allocation`** (high; layer: emitter). `_allocate_counter_slots` walked `program.rules` only; a Tier 2 `count <name>` statement caused a `KeyError` in `_emit_tier2_stmt`. Fix: extended `_allocate_counter_slots` to also walk `program.function.body` via a new `_collect_tier2_counter_slots`. Pinned at `fwl/tests/corpus/10_tier2_functions/edge_count_action_emits_slot.pkt`.

3. **`finding/2026-05-01-v02-emitter-no-v4-parse-ok-gate-cross-family-disagreement`** (high; layer: emitter). The BPF emitter gated v6 comparisons on a `v6_ok` flag but had no equivalent `v4_ok`/`l4_ok` for v4 fields. `pkt.src_ip == 0.0.0.0`, `pkt.src_ip in [0.0.0.0/0]` (which short-circuited to literal `1`), `pkt.dst_port == 0` against ICMPv6, etc. spuriously matched in BPF on cross-family packets while the interpreter correctly returned false. **The spec's own Tier 2 dogfood example (`if pkt.src_ip in [0.0.0.0/0]:` as a v4-establishing guard) was bitten on every v6 packet that reached the guard.** Fix: declared `__u8 v4_ok = 0` (always) and `__u8 l4_ok = 0` (when L4 fields are referenced); set `v4_ok = 1` inside the IPv4 EtherType branch and `l4_ok = 1` inside the TCP/UDP guard blocks (both v4 and v6 paths). Gated `_emit_ip_compare`, `_emit_port_compare`, `_emit_proto_compare`, and `_emit_bool_field` on the corresponding flag. `v6_ok` is now always declared (zero in v0.1-shaped programs, where the OR `(v4_ok || v6_ok)` degrades to `v4_ok`). Pinned at `fwl/tests/corpus/10_tier2_functions/edge_v4_zero_cidr_v6_packet.pkt` and `edge_src_ip_eq_zero_v6_packet.pkt`.

## What landed in code

- `f/fwl/fwl/grammar.lark` — `function_def`, `block`, statement productions, `bare_field` primary, `lvalue/rvalue` extension, `proto_list`, `rate_limit_call`, `DEF` keyword token, `_NL`/`_INDENT`/`_DEDENT` declarations.
- `f/fwl/fwl/parser.py` — `FwlIndenter(Indenter)` with `_process` override that gates indentation on `DEF`, transformers for the new productions, mutually-exclusive Tier 1 + Tier 2 + default-placement enforcement at the program level.
- `f/fwl/fwl/ast.py` — `LocalType`, `LocalRead`, `RateLimitCall`, `AssignStmt`, `ActionStmt`, `IfStmt`, `FunctionDef`; `Program.function`.
- `f/fwl/fwl/analyzer.py` — `_analyze_tier2`, `_Tier2Ctx`, `_check_stmts`/`_check_assign`/`_check_condition`/`_check_tier2_comparison`/`_resolve_lvalue_type`/`_resolve_rvalue_type`/`_check_in_operand`/`_established_by`/`_comparison_guards`/`_check_dominator`/`_check_stack_budget`/`_assign_geoip_call_indices_tier2`/`_assign_rate_limit_call_indices_tier2` and various walk helpers.
- `f/fwl/fwl/interpreter.py` — `_exec_tier2`/`_exec_stmts`/`_eval_scalar`/`_eval_tier2_comparison`/`_ip_or_port_or_proto_in`/`_read_field`/`_PROTO_FROM_STRING`. `_program_touches_v6_surface` and `_condition_touches_v6` extended for Tier 2.
- `f/fwl/fwl/emitter.py` — `_emit_tier2_body`/`_emit_tier2_stmts`/`_emit_tier2_stmt`/`_emit_tier2_assign`/`_emit_tier2_if`/`_emit_scalar`/`_emit_tier2_comparison`/`_emit_field_read_scalar`/`_emit_ipv6_scalar`/`_emit_tier2_in`/`_collect_tier2_locals`/`_infer_assign_type`. `_referenced_fields` and `_is_v6_active` extended for Tier 2. `_allocate_counter_slots` extended for Tier 2 `count` statements. `_collect_geoip_calls` walks function body. New `v4_ok`/`l4_ok` BPF prelude flags + gating in `_emit_ip_compare`/`_emit_port_compare`/`_emit_proto_compare`/`_emit_bool_field`.
- `f/fwl/fwl/runner.py` — `_walk_geoip_in_tier2` so Tier 2 programs' geoip LPM trie maps get populated for `BPF_PROG_TEST_RUN`.
- `f/fwl/examples/dogfood_v02.fw` (new) — Phase 2 dogfood example combining IPv6, geoip(), and Tier 2.
- `f/fwl/tests/corpus/10_tier2_functions/` (new) — 24 .pkt cases.
- `f-hone/config/compiler_construct_map.yaml` — `tier2` tag added to interpreter, emitter, runner; `iso3166.py` already tagged.

## Sign-off

Gate 2 for Construct 3 (Tier 2 programmable functions) — **clear**. Phase 2 implementation is feature-complete; the remaining items in the Phase 2 DoD are the ≥48h soak, the abstract/critique passes, and the v0.2 release note.
