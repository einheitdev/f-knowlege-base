---
id: finding/2026-05-01-v02-spec-tier2-example5-violates-grammar-and-dominator-rule
type: finding
protocol: [tcp, udp]
builtins: []
severity: high
layer: spec
pattern_tags: [spec-inconsistency, tier2, grammar, example-vs-rule, protocol-guards]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: [finding/2026-05-01-v02-spec-tier2-stmt-level-pkt-read-on-wrong-protocol]
---

# 2026-05-01-v02-spec-tier2-example5-violates-grammar-and-dominator-rule

## Summary

Tier 2 worked Example #5 — the *stack-budget stress-test* at
`FWL_V02_SPEC.md:778-797` — is presented as a valid program, yet
violates **two** of the spec's own rules at once:

1. It uses Python's **single-line** `if cond: stmt` form (six times).
   The v0.2 grammar at `FWL_V02_SPEC.md:1031-1034` has no such
   production: `if_stmt` requires NEWLINE after the colon, then an
   INDENT/DEDENT block. The single-line form does not parse.
2. It reads `pkt.src_port` and `pkt.dst_port` at the top of the
   function with no preceding `pkt.proto == tcp|udp` guard. Per the
   "proto-guard dominator rule" at `FWL_V02_SPEC.md:604-610` (and the
   matching Edge case at line 639) those reads are compile errors.

Either the example is wrong, or the grammar/dominator rule are wrong.
Class B (analyzer/spec contradiction) — and a *load-bearing* one,
because corpus authors will copy the example as a positive test case
and the analyser will reject it.

## Locations

### The example as written — `FWL_V02_SPEC.md:778-797`

```python
@xdp(eth0)

def firewall(pkt):
  a = pkt.src_ip
  b = pkt.dst_ip
  c = pkt.src_ip6
  d = pkt.dst_ip6
  e = pkt.src_port
  f = pkt.dst_port
  if a == 10.0.0.1: allow
  if b == 10.0.0.2: drop
  if c == 2001:db8::1: allow
  if d == 2001:db8::2: drop
  if e == 22: log
  if f == 22: count ssh
  drop
```

### Violation 1: grammar admits no single-line `if`

Grammar at `FWL_V02_SPEC.md:1031-1034`:

```ebnf
if_stmt       = "if" condition ":" NEWLINE
                INDENT statement { statement } DEDENT
                ...
```

The example's `if a == 10.0.0.1: allow` (and five sibling lines)
puts the action on the same physical line as the `if` header. There
is no production for that. By the grammar a parser will hit `allow`
where it expected `NEWLINE` and fail.

This is not isolated to Example 5. The Edge cases section uses the
same shorthand for `def`-bodies:

- `FWL_V02_SPEC.md:626-628` — `def f(pkt):\n    pass`
- `FWL_V02_SPEC.md:630-631` — `def f(pkt):\n    allow` (claimed
  "minimal valid Tier 2 program")
- `FWL_V02_SPEC.md:645-647` — `def f(pkt): allow` (note the *inline*
  body in narrative — single-line form again)
- `FWL_V02_SPEC.md:663-665` — `def f(pkt): drop` (inline body)

Half of the Edge cases narrate code in a syntax the grammar does not
accept. A corpus author who lifts those snippets into a `.pkt`
`source` field will get a parse error, not the documented behaviour.

### Violation 2: unguarded statement-level L4 reads

`FWL_V02_SPEC.md:604-610` (the strict resolution of the prior spec
hole `2026-05-01-v02-spec-tier2-stmt-level-pkt-read-on-wrong-protocol`):

> A read of `pkt.src_port`, `pkt.dst_port` is valid only at program
> points dominated by a `pkt.proto == tcp` or `pkt.proto == udp`
> guard … A read that fails the dominator check is a compile error:
> `error: '<field>' read on a path not guarded by 'pkt.proto ==
> <required>'`.

`FWL_V02_SPEC.md:639` reinforces the rule by example:

> *Unguarded statement-level L4 read.* `port = pkt.dst_port` at the
> top of the function (no preceding proto guard) is a compile error
> per the dominator rule above. … The dogfood example at the end of
> this spec respects this rule: every L4 read sits inside an `if
> pkt.proto == tcp` (or `udp`) block.

But Example 5 does the prohibited thing literally:

```python
def firewall(pkt):
  ...
  e = pkt.src_port      # top of body, no proto guard
  f = pkt.dst_port      # ditto
  ...
```

Both `e = …` and `f = …` are compile errors per the rule. The
example also says, in prose, "The estimator should report ~64 bytes
of stack" — implying the program is valid and reaches stack
estimation. It cannot reach stack estimation if the analyser rejects
it at the type/dominator pass.

## Why this matters

The example is *Construct 3 → Examples → #5* — the only example
specifically locking the stack-budget estimator. The corpus is
expected to encode it as a positive case (the analyser estimates,
the verifier confirms). If the example is rejected on entry, the
estimator's behaviour is never exercised by this case, and the
corpus author has to either:

- Edit the example into a different program (then the example loses
  its corpus-locking purpose), or
- Mark this case `expected.compiles: false` (then the spec's
  narrative "should report ~64 bytes" is unreachable nonsense).

Either way, the spec and the corpus drift. This is exactly the
class of spec-bug Phase 2's Gate 1 is supposed to catch before
landing compiler code.

## Proposed fix

Two independent edits, in either order:

### Fix the example (preferred)

Rewrite Example 5 so each `if` has a proper indented body, and the
two L4 reads sit inside a proto-guard block. One concrete rewrite:

```python
@xdp(eth0)

def firewall(pkt):
  a = pkt.src_ip
  b = pkt.dst_ip
  c = pkt.src_ip6
  d = pkt.dst_ip6
  if a == 10.0.0.1:
    allow
  if b == 10.0.0.2:
    drop
  if c == 2001:db8::1:
    allow
  if d == 2001:db8::2:
    drop
  if pkt.proto == tcp or pkt.proto == udp:
    e = pkt.src_port
    f = pkt.dst_port
    if e == 22:
      log
    if f == 22:
      count ssh
  drop
```

Same six locals → same ~64-byte stack estimate. Now grammar-legal
*and* dominator-rule-compliant. Update the prose to confirm.

### Fix the Edge cases narrations

Replace the inline `def f(pkt): allow` / `def f(pkt): drop` /
`def f(pkt): pass` snippets with the indented two-line form:

```python
def f(pkt):
    allow
```

(Same content, just one line further down so the grammar accepts it.)

Affected lines: 626-628, 630-631, 645-647, 663-665.

### Or: extend the grammar to accept Python's `if cond: stmt`

If the spec authors *want* `if cond: stmt` to be a real surface,
amend `if_stmt` (and analogously `function_def` for inline bodies)
to permit a single statement on the header line. Add the
corresponding compile-error condition (e.g. `else`/`elif` after a
single-line `if` is rejected). This is a bigger surface change and
should not be done quietly inside Examples.

## Class

Class B (spec/example contradiction with the spec's own grammar and
the spec's own dominator rule). Spec-layer.
