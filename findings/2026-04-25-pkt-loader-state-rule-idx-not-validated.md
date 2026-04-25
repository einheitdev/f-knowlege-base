---
id: finding/2026-04-25-pkt-loader-state-rule-idx-not-validated
type: finding
protocol: [tcp, udp, icmp]
builtins: [rate_limit]
severity: low
layer: loader
pattern_tags: [pkt-loader-validation, spec-conformance, silent-failure]
status: fixed
source_file: fwl/fwl/pkt.py
created: 2026-04-25
pkt_path: corpus/from_hunt/2026-04-25_loader_state_invalid_idx/
related: [finding/2026-04-25-pkt-loader-unknown-builder-fields]
---

# 2026-04-25-pkt-loader-state-rule-idx-not-validated

## Summary

`fwl/fwl/pkt.py` and `fwl/fwl/runner.py` silently accept `.pkt`
files whose `state.rate_limit` block references rule indices that
either don't exist in the program or point at a rule with no
`rate_limit` modifier. `PKT_V01_SPEC.md:256` mandates the load-time
error:

> state.rate_limit references rule index without a modifier —
> error: rule <idx> has no rate_limit modifier

Implementation: `pkt.load` (`fwl/fwl/pkt.py:268-289`) reads the
state block as raw `{int: {key: int}}` without consulting the
program's rule list. The runner's `_build_map_init`
(`fwl/fwl/runner.py:216-220`) `continue`s past invalid indices
silently, and the interpreter's `_rate_limit_allows`
(`fwl/fwl/interpreter.py:89`) reads `state.get(rule_idx, {})` with
no awareness of whether `rule_idx` is meaningful.

## Hypothesis

A corpus author who fat-fingers a rule index (writes `0:` for the
modifier on the second rule, or carries `9:` over from a different
program) gets a green test that exercises *zero* state. The bucket
counts they wrote are silently discarded, and the failing case they
were trying to demonstrate becomes a passing case for the wrong
reason.

## Test cases

Two `.pkt` cases under
`corpus/from_hunt/2026-04-25_loader_state_invalid_idx/`:

| .pkt case | result | what it shows |
|---|---|---|
| `state_for_rule_without_modifier.pkt` | PASS | Program has one rule (`allow if pkt.proto == tcp`) with no modifier. State block writes `rate_limit.0."1.2.3.4": 999`. Per spec the loader must error with `error: rule 0 has no rate_limit modifier`. Implementation silently accepts and the test passes for the wrong reason — the bucket count of 999 is never consulted. |
| `state_for_nonexistent_rule_idx.pkt` | PASS | Program has one rule. State block writes `rate_limit.99."1.2.3.4": 5`. Same shape — silently accepted; runner's `rule_idx >= len(program.rules)` branch quietly continues, no error surfaces to the corpus author. |

Both cases "pass" the runner. The pass IS the bug: per spec the
loader should reject the file, not run it.

## Root cause

`fwl/fwl/pkt.py:277-280`:

```python
raw_state = (doc.get("state") or {}).get("rate_limit", {})
state: dict[int, dict[Any, int]] = {}
for rule_idx, buckets in raw_state.items():
  state[int(rule_idx)] = dict(buckets)
```

No reference to the parsed program (which `load` doesn't see anyway
— it returns a `PktCase` carrying raw `source_fw` text). The
loader's separation from the analyzer is the friction here: spec
table validation requires the parsed program, but `load` only
parses the YAML, not the embedded `source_fw`.

`fwl/fwl/runner.py:216-220` does see the program, but its
"silently skip" behavior is in service of robustness across runner
internals, not user-facing validation.

## What this rules out

- Class A divergence on the rate_limit modifier itself. The
  modifier is correctly emitted and consumed; the bug is upstream
  in *which* rule's state to populate.
- Class B in the analyzer. Analyzer doesn't see `state` at all.
- Spec ambiguity. `PKT_V01_SPEC.md:256` is unambiguous about the
  error and its wording.

## What this hunt did NOT cover

- Behavior when state references a valid rule index but a bucket
  *value* is malformed (e.g., negative count, non-integer count).
  Spec is silent on per-bucket-value validation.
- Behavior when `state.rate_limit` itself isn't a mapping (e.g.,
  a list). The loader would crash with an opaque AttributeError
  rather than a clean error message — separately worth fixing.

## Severity

Low. The bug is in test infrastructure and surfaces as
"author-side mistake silently swallowed." No production impact;
no Class A on packet-action behavior. But it reduces confidence in
the corpus: a `.pkt` that *appears* to test a rate-limit edge case
may in fact be testing the bare program with no state at all.

## Fix

Validate at load time, after parsing `source_fw`:

```python
# in pkt.load, after yaml.safe_load:
program = parser.parse(doc["source_fw"])  # raises FwlException as today
for rule_idx in raw_state:
  rule_idx = int(rule_idx)
  if rule_idx >= len(program.rules):
    raise ValueError(f"rule {rule_idx} has no rate_limit modifier")
  if program.rules[rule_idx].modifier is None:
    raise ValueError(f"rule {rule_idx} has no rate_limit modifier")
```

This couples the loader to the parser, which is acceptable: the
runner already does both, and the cost of one extra parse at load
time is negligible relative to the BPF clang invocation.

## Resolution

Applied. `pkt.load` (`fwl/fwl/pkt.py`) now invokes
`_rate_limit_rule_indices(source_fw)` (lazy import of
`fwl.parser` + `fwl.analyzer` to avoid a hard cycle) and rejects
any state.rate_limit key whose index is missing or doesn't carry a
RateLimit modifier with `ValueError("state.rate_limit references
rule index N without a rate_limit modifier")`. The two PoC cases
under `corpus/from_hunt/2026-04-25_loader_state_invalid_idx/`
declare `expected.loads: false` and serve as anti-regression
guards.
