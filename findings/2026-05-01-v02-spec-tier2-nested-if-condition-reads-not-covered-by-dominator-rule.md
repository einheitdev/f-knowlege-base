---
id: finding/2026-05-01-v02-spec-tier2-nested-if-condition-reads-not-covered-by-dominator-rule
type: finding
protocol: [tcp, udp]
builtins: []
severity: high
layer: spec
pattern_tags: [spec-ambiguity, tier2, protocol-guards, nested-if, example-vs-rule]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: [finding/2026-05-01-v02-spec-tier2-stmt-level-pkt-read-on-wrong-protocol, finding/2026-05-01-v02-spec-tier2-pkt-proto-no-dominator-rule, finding/2026-05-01-v02-spec-tier2-stmts-after-terminal-example-violates-proto-guard]
---

# 2026-05-01-v02-spec-tier2-nested-if-condition-reads-not-covered-by-dominator-rule

## Summary

The Tier 2 protocol-guard rule has two halves:

- **Statement-position reads** (`port = pkt.dst_port`) — governed by
  the new control-flow **dominator rule** at
  `FWL_V02_SPEC.md:616-628` (an outer `if pkt.proto == tcp:`
  satisfies the guard for any read inside the then-branch).
- **Condition-position reads** (`if pkt.dst_port == 22:`) — governed
  by `FWL_V02_SPEC.md:615`:

  > All v0.1 short-circuit and protocol-guard rules apply
  > **unchanged** inside Tier 2 `if`/`elif` conditions.

The v0.1 protocol-guard rule (`FWL_V01_SPEC.md:148-176`) requires
the proto guard to appear "on the path" *inside the same predicate*
as the field access — that is, inside the same `if`'s condition.
v0.1 has no nested-`if` construct, so "the path" is necessarily
within one boolean expression.

In Tier 2 the condition-position reads in nested `if`s — where the
proto guard sits in an outer `if`'s condition and the field access
sits in an inner `if`'s condition — are not covered by either half:

- Not by the dominator rule, because the dominator rule explicitly
  applies to "fields read **outside** an `if` condition" (`:618`),
  i.e. statement-position reads.
- Not by "v0.1 rules apply unchanged", because v0.1's rule is
  intra-predicate; the inner `if`'s condition is a different
  predicate from the outer `if`'s condition.

The spec's own Tier 2 examples (1, 3, and the dogfood) all rely on
this exact pattern — proto guard in outer condition, field access
in inner condition — and depend on a rule the spec does not state.

Class B (spec underspecification; spec layer). Class A risk: an
implementer who reads `:615` literally rejects the spec's worked
examples; one who relaxes it on intuition produces an analyser the
spec doesn't describe.

## The hole

### What `:615` says

`FWL_V02_SPEC.md:615`:

> *Short-circuit and protocol guards in conditions.* All v0.1
> short-circuit and protocol-guard rules apply **unchanged** inside
> Tier 2 `if`/`elif` conditions. A Tier 2 `if pkt.proto == tcp and
> pkt.dst_port == 22:` reads `pkt.dst_port` only when the proto
> check passes, exactly as in Tier 1.

The example given (`pkt.proto == tcp and pkt.dst_port == 22`)
demonstrates the *same-predicate* case. v0.1's protocol-guard rule
is the same-predicate rule by construction: v0.1 has no other
control flow.

### What the dominator rule says

`FWL_V02_SPEC.md:616-625`:

> *Statement-level `pkt` reads — guard dominator rule.* Tier 2
> introduces the first context where a protocol-specific or
> family-specific `pkt` field can be read **outside an `if`
> condition** (in the right-hand side of a local assignment, e.g.
> `port = pkt.dst_port`). Such reads are governed by a control-flow
> dominator check ...

> "Dominated by" means: every control-flow path from the function
> entry to the read passes through an `if` whose condition has
> already established the required guard. Reads inside the
> then-branch of `if pkt.proto == tcp:` are guarded; reads after
> the `if` block are not...

Explicitly scoped to "outside an `if` condition". The dominator
analysis is presented as the answer to the question "what about
fields read outside conditions?", and the answer to "what about
fields read inside an inner if's condition, where the outer if
already establishes the guard?" is left implicit.

### What the spec's own examples demand

**Example 1 — SSH brute-force (`FWL_V02_SPEC.md:730-740`):**

```python
def firewall(pkt):
  if pkt.proto == tcp and pkt.dst_port == 22:
    if pkt.tcp.syn and not pkt.tcp.ack:
      if rate_limit(10, per=src_ip):
        drop
    allow
  allow
```

The inner `if pkt.tcp.syn and not pkt.tcp.ack:` reads
`pkt.tcp.syn` and `pkt.tcp.ack`. Per the v0.1 rule, both reads
require `pkt.proto == tcp` *in the same predicate*. The inner
predicate has neither — the proto guard sits in the **outer**
predicate.

**Example 3 — if/elif/else (`FWL_V02_SPEC.md:766-785`):**

```python
def firewall(pkt):
  if pkt.proto == tcp:
    if pkt.dst_port == 22:
      ...
    elif pkt.dst_port in [80, 443]:
      ...
```

The inner `if pkt.dst_port == 22:` and `elif pkt.dst_port in
[80, 443]:` both read `pkt.dst_port`. Per the v0.1 rule, the read
requires `pkt.proto == tcp` or `pkt.proto == udp` *in the same
predicate*. Neither inner predicate has it.

**Tier 2 dogfood (`FWL_V02_SPEC.md:968-1000`):**

```python
def firewall(pkt):
  ...
  if pkt.proto == tcp and pkt.dst_port == 22:
    if pkt.tcp.syn and not pkt.tcp.ack:
      if rate_limit(10, per=src_ip):
        drop
    count ssh_seen
    allow
```

Same shape as Example 1.

The relax-`:615`-to-allow-nested-guards reading is what the
examples assume. The literal-`:615` reading rejects all three.

### Compounding finding 2026-05-01-stmts-after-terminal-example

`finding/2026-05-01-v02-spec-tier2-stmts-after-terminal-example-violates-proto-guard`
(status: fixed) rewrote the unreachable-statement example to:

```python
if pkt.proto == tcp:
    if pkt.tcp.syn:
        drop
allow
```

The fix proposal asserts:

> "is fine — the `allow` is reachable when either the outer or
> the inner `if` does not match."

The fix's premise is that `if pkt.tcp.syn:` (no proto guard in this
predicate) is fine because the outer `if pkt.proto == tcp:`
dominates it. That premise is exactly the rule this finding says
the spec lacks. The earlier finding silently extended the
dominator rule from statement-position reads to condition-position
reads in nested ifs, but did not update `:615` (the rule it
contradicted) or add a paragraph stating the extension.

So: three Tier 2 examples and one prior finding's fix all rely on
"outer-`if` proto guard makes inner-`if` condition reads valid";
the spec's stated rules do not say that.

## Why this matters

Two implementers with the spec in hand and no contact with each
other:

| Implementer | Reading | Effect |
|---|---|---|
| (a) "v0.1 rules apply *unchanged*" → same-predicate only | rejects all three Tier 2 worked examples; rejects the dogfood | corpus is uncompilable; analyser disagrees with spec on its own positive cases |
| (b) "examples are authoritative; the dominator rule generalises to condition position" | accepts the worked examples; same intent the user reading the spec gets | matches intent but reads in rules the spec doesn't state |

A user writing `if pkt.tcp.syn:` at the top of a `def firewall(pkt):`
(no outer guard) hits a compile error in either reading; that case
is fine. The split is on the *nested* form.

A `.pkt` test that asserts Example 1 compiles (`expected.compiles:
true`) gets compile-success on (b) and compile-error on (a). Same
source, different verdicts, both spec-conformant.

## Three possible resolutions

| Resolution | What changes |
|---|---|
| (a) Generalise the dominator rule to condition-position reads in nested ifs | Edit the Statement-level dominator rule's header to "Statement- and condition-position `pkt` reads — guard dominator rule" and add: "An inner `if`'s condition reads a guarded field validly when an outer `if`'s condition has established the guard. The same dominator analysis used for statement-position reads applies." Then `:615` becomes a special case (single-predicate short-circuit is one form of dominance — the outer condition's guard dominates the rest of that condition). |
| (b) Keep `:615` strict, edit the examples | Rewrite Examples 1, 3, dogfood, and the prior finding's fix to put the proto guard in every inner predicate too: `if pkt.proto == tcp and pkt.tcp.syn and not pkt.tcp.ack:`. The examples become more verbose; the rule stays clean. |
| (c) Keep `:615` strict, but spec-state the nested-if exception | Add a separate bullet: "When an inner `if`'s condition would otherwise lack a required proto guard, an enclosing `if` whose condition already established the guard satisfies the requirement. The dominator analysis from the statement-position rule applies." Same effect as (a), narrower wording. |

(a) and (c) are the same outcome with different wording. (b) is
a non-trivial spec rewrite (every Tier 2 example needs editing)
and produces clumsier user-facing code. The cleanest fix is (a).

## Proposed fix

Edit `FWL_V02_SPEC.md:615`:

> *Short-circuit and protocol guards in conditions.* All v0.1
> short-circuit and protocol-guard rules apply **unchanged**
> inside a single Tier 2 `if`/`elif` condition: a condition like
> `pkt.proto == tcp and pkt.dst_port == 22` reads `pkt.dst_port`
> only when the proto check passes, exactly as in Tier 1.
> **Across nested `if`s, an enclosing `if` whose condition has
> already established a required guard satisfies the requirement
> for inner `if`/`elif` conditions and for any statement-position
> read inside the enclosing branch — see the dominator rule
> below.**

And update `:616` (the dominator-rule heading) from:

> *Statement-level `pkt` reads — guard dominator rule.* Tier 2
> introduces the first context where a protocol-specific or
> family-specific `pkt` field can be read outside an `if`
> condition...

to:

> *Statement- and inner-condition `pkt` reads — guard dominator
> rule.* Tier 2 introduces the first context where a
> protocol-specific or family-specific `pkt` field can be read in
> a position not covered by the same-predicate short-circuit
> rule — either outside an `if` condition (in the right-hand side
> of a local assignment, e.g. `port = pkt.dst_port`) or inside a
> nested `if`'s condition where the proto guard sits in an
> enclosing `if` rather than the same predicate. Both cases are
> governed by the same control-flow **dominator check**:

The rest of the bullet stays as-is: it already enumerates the
guards correctly for L3/L4/proto reads.

This change makes the dominator analysis a single rule covering
both surfaces — statement-position reads AND condition-position
reads in nested ifs. The same control-flow analysis the analyser
already needs for the statement-position rule covers the new
case for free.

The Compile-errors table at `:711-717` does not need new rows
— the existing rows ("'<field>' read on a path not guarded by
'pkt.proto == <required>'") fire equally for inner-condition reads
and for statement-position reads.

## Class

Class B — spec underspecification at the rule layer. The spec
defines the protocol-guard rule for two surfaces (single-predicate
short-circuit and statement-position dominator) but uses a third
surface (inner-`if` condition with outer-`if` guard) in three
worked examples without stating the rule. Implementations will
diverge on a high-traffic surface — the dogfood program itself.
