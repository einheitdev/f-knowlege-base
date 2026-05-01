---
id: finding/2026-05-01-v02-emitter-tier2-count-action-keyerror-no-slot-allocation
type: finding
protocol: [tcp, udp, icmp]
builtins: []
severity: high
layer: emitter
pattern_tags: [tier2, count, counter-slot, keyerror, emitter-crash, dogfood, helper-skips-function-body]
status: fixed
source_file: fwl/fwl/emitter.py
created: 2026-05-01
fixed: 2026-05-01
pkt_path: corpus/from_hunt/2026-05-01_dogfood_hunt/tier2_count_action_keyerror.pkt
related: [finding/2026-05-01-v02-interpreter-tier2-v6-activation-helper-skips-function-body]
---

## Fix applied

`fwl/fwl/emitter.py:_allocate_counter_slots` now walks
`program.function.body` after Tier 1 rules via a new
`_collect_tier2_counter_slots` recursive helper. Tier 2 `count <n>`
statements get a stable per-CPU slot in source order, identical to
the Tier 1 path. Pinned regression at
`fwl/tests/corpus/10_tier2_functions/edge_count_action_emits_slot.pkt`.
Verified: 455/455 pytest, no new regressions.


# 2026-05-01-v02-emitter-tier2-count-action-keyerror-no-slot-allocation

## Summary

`_allocate_counter_slots(program)` in `fwl/fwl/emitter.py:1364-1380`
iterates `program.rules` only and never inspects `program.function`.
For a Tier 2 program that uses any `count <name>` action statement
(legal per FWL_V02_SPEC.md — "Actions: ... `count <n>` ... unchanged;
available as Tier 2 statements"), the emitter walks the function
body in `_emit_tier2_stmt`, hits the `count` branch, and looks up
`ctx.counter_slots[stmt.counter_name]` — which raises `KeyError`
because `_allocate_counter_slots` never inserted it.

The compiler crashes with an unhandled `KeyError` instead of either
emitting a proper counter slot or returning a clean compile error.
This is the same shape as the recently-fixed v6-activation gap in
the interpreter (`finding/.../tier2-v6-activation-helper-skips-function-body`)
and points to a class of bugs where v0.2 helpers that need to walk
the program forgot to descend into `program.function.body`.

## Hypothesis

A helper in the emitter walks `program.rules` but not
`program.function.body`. Specifically, the emitter's slot allocator
is the v0.1 implementation extended to v0.2 without v0.2 awareness.
A Tier 2 `count` statement triggers a KeyError downstream.

## Evidence

`corpus/from_hunt/2026-05-01_dogfood_hunt/tier2_count_action_keyerror.pkt`:

```yaml
name: "Tier 2: count action — counter slot allocation"
source_fw: |
  @xdp(eth0)
  def firewall(pkt):
    if pkt.proto == tcp:
      count tcp_seen
      allow
    drop

test_packet:
  builder: tcp(src_ip="1.2.3.4", dst_port=80)

expected:
  compiles: true
  bpf_action: allow
```

Run output (truncated traceback):

```
File "fwl/emitter.py", line 1188, in _emit_tier2_stmt
  slot = ctx.counter_slots[stmt.counter_name]
KeyError: 'tcp_seen'
```

The traceback confirms the call chain:
`emit()` → `_allocate_counter_slots()` returns `{}` (only walks
rules, finds none) → `_emit_tier2_body()` → `_emit_tier2_stmts()`
→ `_emit_tier2_stmt(ActionStmt(COUNT, counter_name='tcp_seen'))`
→ `ctx.counter_slots['tcp_seen']` → KeyError.

## Surface area

Every Tier 2 v0.2 program with a `count <name>` statement crashes
the emitter. The interpreter (`fwl/fwl/interpreter.py:114`) silently
ignores `count`/`log` statements (LOG and COUNT are non-terminal,
side effect omitted in test) so it doesn't crash. The analyzer
likewise accepts the program. So:

- `.pkt` runner: spec=pass, interpreter=pass, BPF=crash
- `fwl compile <prog.fw>`: stack trace, no output

The dogfood example as currently written (`fwl/examples/dogfood_v02.fw`)
does **not** use `count` so it doesn't trigger this. But the v0.2
spec advertises `count` as a Tier 2 action and the rate-limit
example in the spec body uses `count ssh_seen`, which means any
operator copying the spec's flagship pattern hits this immediately.

## Recommended fix

`fwl/fwl/emitter.py:1364-1380` — extend `_allocate_counter_slots` to
also walk `program.function.body` for Tier 2 `count` statements,
preserving source-order allocation.

Sketch:

```python
def _allocate_counter_slots(program: ast.Program) -> dict[str, int]:
  slots: dict[str, int] = {}
  for rule in program.rules:
    if rule.action == ast.Action.COUNT and rule.counter_name:
      if rule.counter_name not in slots:
        slots[rule.counter_name] = len(slots)
  if program.function is not None:
    _collect_tier2_counter_names(program.function.body, slots)
  return slots

def _collect_tier2_counter_names(stmts, slots):
  for s in stmts:
    if isinstance(s, ast.ActionStmt) and s.action == ast.Action.COUNT:
      if s.counter_name not in slots:
        slots[s.counter_name] = len(slots)
    elif isinstance(s, ast.IfStmt):
      _collect_tier2_counter_names(s.body, slots)
      for _, body in s.elif_branches:
        _collect_tier2_counter_names(body, slots)
      if s.else_body is not None:
        _collect_tier2_counter_names(s.else_body, slots)
```

Add the corpus case above to `fwl/tests/corpus/10_tier2_functions/`
to lock the regression.

## Class

Class A — interpreter accepts the program but the BPF emitter
crashes. While the interpreter happens to swallow LOG/COUNT during
testing, the spec defines Tier 2 `count` as well-formed; the BPF
oracle must produce a working program, not raise a Python
exception.
