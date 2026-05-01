---
id: finding/2026-05-01-v02-spec-tier2-stmts-after-terminal-example-violates-proto-guard
type: finding
protocol: [tcp]
builtins: []
severity: medium
layer: spec
pattern_tags: [spec-inconsistency, tier2, protocol-guards, example-vs-rule]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: [finding/2026-05-01-v02-spec-tier2-stmt-level-pkt-read-on-wrong-protocol, finding/2026-05-01-v02-spec-tier2-example5-violates-grammar-and-dominator-rule]
---

# 2026-05-01-v02-spec-tier2-stmts-after-terminal-example-violates-proto-guard

## Summary

The "Statements after a terminal" Tier 2 edge case at
`FWL_V02_SPEC.md:643-647` claims that the program

```
if pkt.tcp.syn:
    drop
allow
```

"is fine — the `allow` is reachable when the `if` does not match."

But `if pkt.tcp.syn:` is a compile error per the proto-guard rule
inherited from v0.1 (and explicitly affirmed for Tier 2 conditions
at `FWL_V02_SPEC.md:606`). The spec's own rule rejects the very
program it offers as the positive example for the
unreachable-statement check.

Class B (analyzer/spec inconsistency, example contradicts the
spec's own rule).

## Locations

### The example as written

`FWL_V02_SPEC.md:643-647`:

> *Statements after a terminal.* `if pkt.tcp.syn:\n    drop\nallow`
> is fine — the `allow` is reachable when the `if` does not match.
> But `drop\nallow` (the second statement is unconditionally dead)
> is a compile error: `error: unreachable statement after terminal
> action 'drop'`. Reachability is computed per branch.

### v0.2 affirms the v0.1 proto-guard rule for Tier 2 conditions

`FWL_V02_SPEC.md:606`:

> *Short-circuit and protocol guards in conditions.* All v0.1
> short-circuit and protocol-guard rules apply unchanged inside
> Tier 2 `if`/`elif` conditions. A Tier 2 `if pkt.proto == tcp and
> pkt.dst_port == 22:` reads `pkt.dst_port` only when the proto
> check passes, exactly as in Tier 1.

### v0.1 rule explicitly rejects the example's form

`FWL_V01_SPEC.md:171-172`:

```python
# error: pkt.tcp.syn requires pkt.proto == tcp guard
drop if pkt.tcp.syn
```

Translated to Tier 2 form: `if pkt.tcp.syn:\n    drop` is a compile
error for exactly the same reason. The proto guard is missing.

## Why this matters

The example is the only positive demonstration of the
"reachability is computed per branch" rule for Tier 2. A corpus
author who lifts it into a `.pkt` file expecting `compiles: true`
gets a compile error citing the proto-guard rule, not the
reachability semantics the case is meant to test.

The unreachable-statement rule itself is fine — it is the chosen
example that fails. The fix is to wrap the inner `if` in a
`pkt.proto == tcp` guard:

```
if pkt.proto == tcp:
    if pkt.tcp.syn:
        drop
allow
```

Now the inner `pkt.tcp.syn` read is dominated by `pkt.proto == tcp`,
the program parses, and the trailing `allow` is reachable when the
inner `if` does not match. The reachability rule is exercised; no
proto-guard rule is violated.

## Proposed fix

Edit `FWL_V02_SPEC.md:643-647` to:

> *Statements after a terminal.*
>
> ```
> if pkt.proto == tcp:
>     if pkt.tcp.syn:
>         drop
> allow
> ```
>
> is fine — the `allow` is reachable when either the outer or the
> inner `if` does not match. But
>
> ```
> drop
> allow
> ```
>
> (the second statement is unconditionally dead) is a compile
> error: `error: unreachable statement after terminal action
> 'drop'`. Reachability is computed per branch.

Same content, but the example now respects the proto-guard rule
and stays purely about reachability.

## Class

Class B — spec/example contradiction. The example's prose claims a
program is "fine" that the spec's own protocol-guard rule rejects.
Spec layer.
