---
id: finding/2026-05-01-v02-interpreter-tier2-v6-activation-helper-skips-function-body
type: finding
protocol: [tcp, udp, icmp, icmp6]
builtins: []
severity: high
layer: interpreter
pattern_tags: [v6-activation, tier2, oracle-disagreement, ipv6, gate-helper, dogfood, cross-family]
status: fixed
source_file: fwl/fwl/interpreter.py
created: 2026-05-01
fixed: 2026-05-01
pkt_path: corpus/from_hunt/2026-05-01_dogfood_hunt/tier2_v6_activation_via_function_only.pkt
related: [finding/2026-05-01-v02-interpreter-missing-v6-surface-activation-gate]
---

## Fix applied

`fwl/fwl/interpreter.py:_program_touches_v6_surface` now walks
`program.function.body` via a new `_stmts_touch_v6` helper that
mirrors the emitter's `_stmts_activate_v6` traversal (kept as a
private interpreter helper per the oracle-independence rule).
`_condition_touches_v6` also gained branches for bare `FieldRef`
and `Ipv6Literal` nodes so Tier 2 `addr = pkt.src_ip6` and
`addr = ::1` shapes activate v6 correctly.

Reproducer pinned at
`fwl/tests/corpus/10_tier2_functions/edge_v6_activation_via_function_body.pkt`.
Verified: 454/454 pytest cases, 130/130 corpus regress under sudo
(BPF live), no other regressions.


# 2026-05-01-v02-interpreter-tier2-v6-activation-helper-skips-function-body

## Summary

The interpreter's `_program_touches_v6_surface(program)`
(`fwl/fwl/interpreter.py:340-351`) walks `program.rules` only and
never inspects `program.function`. For a Tier 2 program that
activates the v6 parse path **only via its function body** (the
common case — Tier 1 rules and Tier 2 are mutually exclusive in
v0.2), the helper falsely returns `False`.

Consequence: when such a program runs against a v6-builder packet,
`_gate_v6_packet_for_v01_program` strips the v0.1-style fields
(`proto`, `src_port`, `dst_port`, `syn`, `ack`, `src_ip`, `dst_ip`)
from the packet dict before Tier 2 evaluation. Subsequent reads of
`pkt.proto`, `pkt.dst_port`, etc. inside the Tier 2 body return
`None`, and any condition that depends on them silently fails.

The BPF emitter is unaffected — `_is_v6_active` in
`fwl/fwl/emitter.py:226-244` correctly walks `program.function.body`
via `_stmts_activate_v6` and emits the v6 parse path. So the
interpreter and BPF runtime disagree on every Tier 2 v6-active
program whose body reads v0.1-style fields against a v6 frame.

This is the canonical Class A bug (oracle disagreement on the same
packet) and it directly affects the Phase 2 dogfood example
(`fwl/examples/dogfood_v02.fw`), where the same combination is the
norm: `pkt.src_ip6 in fc00::/7` activates v6, then later branches
read `pkt.proto == tcp and pkt.dst_port in [80, 443]`. Every v6
non-internal HTTP/HTTPS/DNS packet would be allowed by BPF and
dropped by the interpreter.

## Hypothesis

The interpreter has two helpers that walk the program for v6
activation; one (the gate) ignores Tier 2; the other (the AST
walks Tier 1 rules) is for a different purpose. The emitter's
counterpart correctly walks both.

`fwl/fwl/interpreter.py:340-351`:

```python
def _program_touches_v6_surface(program: ast.Program) -> bool:
  """True when the program activates the v6 parse path.

  Mirrors emitter._is_v6_active. ...
  """
  for rule in program.rules:
    if _condition_touches_v6(rule.condition):
      return True
  return False
```

The docstring claims "Mirrors emitter._is_v6_active" but the
emitter's helper additionally walks `program.function.body` via
`_stmts_activate_v6` (`fwl/fwl/emitter.py:241-243`):

```python
if program.function is not None:
  if _stmts_activate_v6(program.function.body):
    return True
```

## Evidence

`corpus/from_hunt/2026-05-01_dogfood_hunt/tier2_v6_activation_via_function_only.pkt`:

```yaml
name: "Tier 2: v6 active via function body — dst_port read on v6 packet"
source_fw: |
  @xdp(eth0)
  def firewall(pkt):
    if pkt.src_ip6 in fc00::/7:
      allow
    if pkt.proto == tcp and pkt.dst_port == 80:
      allow
    drop

test_packet:
  builder: tcp6(src_ip="2001:db8::dead", dst_ip="2001:db8::cafe", dst_port=80)

expected:
  compiles: true
  bpf_action: allow
```

Run output:

```
FAIL  tier2_v6_activation_via_function_only.pkt  (...)
      spec         pass
      interpreter  fail  -- expected XDP_PASS, got XDP_DROP
      bpf          skip  [BPF_PROG_RUN unavailable; clang compile passed]
```

The emitted BPF C confirms the runtime would allow:

```c
} else if (eth->h_proto == bpf_htons(ETH_P_IPV6)) {
  ...
  if (proto == IPPROTO_TCP) { ... }  // populates dst_port
}
if ((v6_ok && ((src_ip6_hi & 0xFE00000000000000ull)
               == 0xFC00000000000000ull))) {
  return XDP_PASS;
}
if ((((proto == IPPROTO_TCP)) && ((dst_port == 80)))) {
  return XDP_PASS;
}
return XDP_DROP;
```

For `2001:db8::dead` (not in `fc00::/7`), the second `if` fires →
`XDP_PASS`. The interpreter strips `proto`/`dst_port` from the
gated packet, so both Tier 2 conditions evaluate to false → the
final `drop` statement returns → `XDP_DROP`.

## Surface area

Every Tier 2 v0.2 program that activates v6 only via the function
body (i.e., every realistic Tier 2 v6 program — including the
dogfood example) and reads any of `proto`, `src_port`, `dst_port`,
`syn`, `ack`, `src_ip`, `dst_ip` against a v6 frame:

- `fwl/examples/dogfood_v02.fw` — the Phase 2 soak target.
  HTTP/HTTPS/DNS allow rules silently drop in the interpreter for
  every non-internal v6 packet.
- Any `f/fwl/tests/corpus/10_tier2_functions/*.pkt` that combines
  v6 activation in the body with v0.1-style L4 reads (latent — none
  of the existing cases happen to exercise this combination, which
  is why it slipped through).

The BPF runtime is correct; the verification authority (the
interpreter side of the three-oracle loop) is wrong.

## Recommended fix

`fwl/fwl/interpreter.py:340-351` — extend `_program_touches_v6_surface`
to walk the function body using a Tier 2-aware traversal. Reusing
the existing `_condition_touches_v6` helper requires a wrapper that
also descends into `IfStmt.body`, `IfStmt.elif_branches`,
`IfStmt.else_body`, and `AssignStmt.rhs`. The emitter's
`_stmts_activate_v6` (`fwl/fwl/emitter.py:268-288`) is the shape to
mirror — but the interpreter must keep its own copy per the oracle-
independence rule (F_DEVELOPMENT_METHODOLOGY.md:307-311).

Sketch:

```python
def _program_touches_v6_surface(program: ast.Program) -> bool:
  for rule in program.rules:
    if _condition_touches_v6(rule.condition):
      return True
  if program.function is not None:
    if _stmts_touch_v6(program.function.body):
      return True
  return False

def _stmts_touch_v6(stmts) -> bool:
  for s in stmts:
    if isinstance(s, ast.IfStmt):
      if _condition_touches_v6(s.cond):
        return True
      if _stmts_touch_v6(s.body):
        return True
      for cond, body in s.elif_branches:
        if _condition_touches_v6(cond):
          return True
        if _stmts_touch_v6(body):
          return True
      if s.else_body is not None and _stmts_touch_v6(s.else_body):
        return True
    elif isinstance(s, ast.AssignStmt):
      # Tier 2 RHS may be a Comparison / IP6 literal / etc.
      if _condition_touches_v6(s.rhs):
        return True
  return False
```

Add a corpus regression case (the `.pkt` above) under
`fwl/tests/corpus/10_tier2_functions/` so the gap can't reopen.

## Class

Class A — interpreter and BPF runtime disagree on the same packet
for the same program. The bug is in the interpreter; the BPF
emitter is correct.
