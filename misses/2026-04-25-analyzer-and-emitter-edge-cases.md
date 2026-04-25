---
id: miss/2026-04-25-analyzer-and-emitter-edge-cases
type: miss
protocol: [tcp, udp, icmp]
builtins: [rate_limit, count, log]
severity: n/a
layer: compiler
pattern_tags: [analyzer-guards, port-boundaries, cidr-boundaries, or-precedence, type-mismatch, short-circuit, counter-slot]
status: open
created: 2026-04-25
---

# 2026-04-25-analyzer-and-emitter-edge-cases

## Summary

Hunted for compiler bugs across analyzer protocol-guard logic,
type-mismatch detection, port and CIDR boundary handling,
boolean-composition short-circuit, OR precedence, counter slot
allocation, and rate_limit modifier interactions. Wrote 22+ targeted
`.pkt` cases. No new Class A or compiler-level Class B bugs surfaced
in this round; the interpreter and the emitter agreed on every
matching/non-matching case, the analyzer rejected every spec-invalid
program with a clear error, and clang accepted every emitted C
source.

The hunt did surface three unrelated findings at the loader / spec
layer (separate documents under `findings/`), but the FWL compiler
core (parser → analyzer → interpreter / emitter) was clean across
everything tested here.

## Hypotheses tested

### Analyzer protocol-guard logic

| # | Hypothesis | Result |
|---|---|---|
| 1 | `drop if not pkt.tcp.syn` (no proto guard) → rejected | confirmed |
| 13 | `pkt.proto != tcp and pkt.dst_port == 80` (negated proto doesn't establish guard) → rejected | confirmed |
| 14 | `(pkt.proto == tcp and pkt.tcp.syn) or (pkt.proto == udp and pkt.dst_port == 80)` (per-branch guards union) → accepted, fires correctly | confirmed |
| 18 | `pkt.proto in [tcp, udp]` (grammar restricts proto to ==/!= with PROTO_KEYWORD) → syntax error | confirmed |

### Type and range checks

| # | Hypothesis | Result |
|---|---|---|
| 2 | `pkt.dst_port == 65535` (boundary, valid) → accepted, matches | confirmed |
| 3 | `pkt.dst_port == 65536` (out of range, decimal) → rejected | confirmed |
| 4 | `pkt.dst_port == 0x10000` (out of range, hex) → rejected | confirmed |
| 5 | `pkt.src_ip > 1.2.3.4` (ordered compare on IP) → rejected | confirmed |
| 10 | `pkt.dst_port in 100..50` (lo > hi) → rejected | confirmed |
| 11 | `pkt.src_ip == 256.0.0.1` (octet > 255) → rejected | confirmed |

### CIDR semantics

| # | Hypothesis | Result |
|---|---|---|
| 8 | `pkt.src_ip in 0.0.0.0/0` matches every IP (mask=0 special case → emits literal `1`) | confirmed |
| 12 | CIDR /32 single-host match → matches only the exact IP | confirmed |

### Identifier / keyword interactions

| # | Hypothesis | Result |
|---|---|---|
| 6 | `count tcp` (counter name = reserved proto keyword) → rejected | confirmed (lexer refuses; spec is silent on whether keywords can be counter names — Class B candidate but not surfacing) |
| 7 | `@xdp(eth0)\ndefault drop` (zero rules + default) → accepted, drops all | confirmed |

### Boolean composition / OR precedence

| # | Hypothesis | Result |
|---|---|---|
| 19a | `tcp or (udp and dst_port==443)`, TCP packet → matches first branch | confirmed |
| 19b | same condition, UDP/443 → matches second branch | confirmed |
| 19c | same condition, UDP/53 → no match | confirmed |
| 19d | same condition, ICMP → no match | confirmed |
| 20a | `drop if X` then `count Y` after (terminal action stops eval) | confirmed |
| 20b | `count` then `drop if X` (non-terminal continues) | confirmed |
| 20c | OR short-circuit on first true branch | confirmed |

### Hex literals and integer ranges

| # | Hypothesis | Result |
|---|---|---|
| 9 | `pkt.dst_port in 80..80` (single-value range) → matches | confirmed |
| 15 | hex port literal `0x50 == 80` → matches | confirmed |

## Compiler-surface checks (static)

For each of the above, inspected the emitted BPF C and the
interpreter's evaluation path:

- **CIDR /32 boundary**: emitter generates
  `((src_ip & 0xFFFFFFFFu) == 0xC0A80105u)` — the `0xFFFFFFFFu`
  literal is well-formed for clang. Interpreter computes
  `((1 << 32) - 1) << 0 = 0xFFFFFFFF` correctly in Python's
  arbitrary-precision int.
- **CIDR /0 special-case**: emitter returns literal `"1"` (no
  parens). When wrapped in `if (1) { ... }` clang accepts cleanly.
  Interpreter `_cidr_match` returns `True` directly for `bits == 0`.
- **Range with hex bounds (`0x50..0x100`)**: parser interprets both
  as integer tokens, range carries `lo=80, hi=256`. Both interpreter
  and emitter use these as decimals. Match.
- **Counter slot reuse**: `_allocate_counter_slots` allocates one
  slot per unique name; both rules with `count many` increment slot
  0. The runtime has the same slot table either way.
- **Rate-limit gate code share**: `_emit_rate_limit_gate` and
  `_emit_rate_limit_side_effect` use the same `cur >= threshold`
  pre-increment predicate. Interpreter `_rate_limit_allows` uses
  the same predicate. Modes agree on the gate.
- **`_referenced_fields`**: when the only port reference comes via
  a rate_limit `per=src_port`, `needs_l4` is correctly true; the
  port is parsed in the prelude.
- **OR-precedence parse tree**: `tcp or udp and dst_port == 443`
  produces an `OrOp` whose second operand is an `AndOp` carrying
  the udp/port comparison; analyzer correctly establishes the per-
  branch guards.

## What this rules out for the compiler

- Off-by-one at port boundaries (0, 65535, 65536, 0x10000).
- Off-by-one or sign errors on octet bounds (256.0.0.1).
- IP CIDR mask off-by-one at /0 and /32.
- Type-mismatch in IP `<`/`>` comparisons.
- Missing-guard escape via NotOp on a bool field.
- Missing-guard escape via `!=` on `pkt.proto`.
- Missing-guard escape via `pkt.proto in [...]` (grammar refuses).
- OR-branch guard union: each branch contributes its own narrowed
  scope, the union is computed correctly even with TCP-only and
  UDP-only branches.
- Reserved-keyword counter names are refused at lex time (spec is
  silent but the behavior is consistent).
- Empty (rules-only-default or rules-zero) programs compile and
  execute as the spec describes.
- Counter slot allocation: stable across rules with the same name;
  no double-allocation.
- Rate-limit gate emission: identical pre-increment predicate
  across interpreter and emitter, confirmed by both code review
  and parallel `.pkt` tests.

## What this hunt did NOT cover

- Class A runtime divergence between BPF_PROG_TEST_RUN and the AST
  interpreter — the environment lacks unprivileged BPF; only
  static inspection plus clang compile pass were possible.
- Counter delta or log-event payload assertions (runner doesn't
  yet drain those — `counter_changes` and `log_events` are
  spec-only per `PKT_V01_SPEC.md:144,176`).
- `truncate_to` truncated-packet bounds-check fall-through paths
  (loader doesn't yet implement, per `PKT_V01_SPEC.md:81`).
- Multi-packet / cross-window rate_limit accumulation. The
  interpreter doesn't model time; the runner pre-loads the bucket
  ts so it doesn't expire mid-test.

## Corpus added

22 `.pkt` cases under
`corpus/from_hunt/2026-04-25_round2/` (h01–h15) and
`/tmp/fwl_h17`–`/tmp/fwl_h20` (kept off-corpus, hypothesis-only).
The h01–h15 set is suitable as regression coverage for the
analyzer's guard-rejection paths and for the boundary cases on
ports/CIDRs.
