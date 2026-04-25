---
id: finding/2026-04-25-pkt-loader-unknown-builder-fields
type: finding
protocol: [icmp, tcp, udp]
builtins: [rate_limit]
severity: medium
layer: loader
pattern_tags: [pkt-loader-validation, oracle-skew, builder-fields, spec-conformance]
status: fixed
source_file: fwl/fwl/pkt.py
created: 2026-04-25
pkt_path: corpus/from_hunt/2026-04-25_loader_unknown_field/
---

# 2026-04-25-pkt-loader-unknown-builder-fields

## Summary

`fwl/fwl/pkt.py` silently accepts unknown builder argument names, in
violation of the explicit `.pkt` v0.1 contract at
`docs/PKT_V01_SPEC.md:253`:

> Builder field not in the constructor's table — error: unknown
> <proto> field '<name>'

Implementation: `parse_builder` (`fwl/fwl/pkt.py:123-135`) accepts any
`name=value` pair from `_split_args` without checking the `name`
against the constructor's field table. `build_packet` then copies
the unfiltered `fields` dict into `decoded` for every constructor
(`tcp`/`udp`/`icmp` branches at lines 216, 231, 240).

For `tcp(...)` and `udp(...)` the bug is mostly cosmetic — typos
(`tcp(syn=1, syn_typo=true)`) silently no-op rather than catch the
mistake. For `icmp(...)` the bug is operationally severe: an ICMP
packet's decoded fields can carry `src_port`/`dst_port`/`syn`/`ack`
attributes that the wire bytes don't, creating a Class A divergence
between the AST interpreter (which consumes decoded fields) and the
BPF runtime (which parses real bytes).

This is Class B at the loader, latent Class A at the methodology
level when state is involved.

## Hypothesis

Per `PKT_V01_SPEC.md:222-228` the `icmp(...)` constructor's field
table lists exactly two fields: `src_ip` and `dst_ip`. Anything else
must be rejected.

The implementation:

```python
# fwl/fwl/pkt.py:123-135
def parse_builder(text):
  match = _BUILDER_RE.match(text)
  ...
  fields = {"proto": match.group("proto")}
  for name, raw_value in _split_args(match.group("args")):
    fields[name] = _parse_value(raw_value)  # no field-table check
  return fields
```

```python
# fwl/fwl/pkt.py:236-242 (icmp branch)
elif proto == "icmp":
  proto_num = _IPPROTO_ICMP
  l4 = struct.pack(">BBHHH", 8, 0, 0, 0, 0)
  decoded = dict(fields)        # KEEPS arbitrary user keys
  decoded.setdefault("src_ip", src_ip)
  decoded.setdefault("dst_ip", dst_ip)
```

`builder: icmp(src_port=12345)` therefore produces a Packet where
`raw` is a real ICMP frame (no L4 ports) but `fields` carries
`src_port: 12345`.

The interpreter's `_rate_limit_allows` reads
`packet.get(mod.per_field)` for the bucket key
(`fwl/fwl/interpreter.py:84`), which returns 12345. The BPF
emitter's gate uses `(__u32)src_port`
(`fwl/fwl/emitter.py:464,491`), where `src_port` is initialized to
0 and only overwritten in the TCP/UDP parse branches — so for an
ICMP frame, `src_port == 0` at the gate.

The two oracles look up *different* buckets for the same `.pkt`
case. With empty state both buckets are absent and both oracles
agree (0 >= threshold = false → allow). With state pre-populated
at bucket 0 (which is what the BPF will read), the interpreter
still misses the state entirely — it looks up bucket 12345 — and
returns "below threshold," while BPF reads cur=count and fires
the rule.

## Test cases

Two `.pkt` cases under `corpus/from_hunt/2026-04-25_loader_unknown_field/`:

| .pkt case | result | what it shows |
|---|---|---|
| `icmp_with_unknown_src_port_field.pkt` | PASS | The loader silently accepts `icmp(src_port=12345)`. Both oracles produce `XDP_PASS` because empty state masks the bucket-key divergence. The test "passing" is itself the bug — per spec it should fail to load. |
| `icmp_unknown_field_state_skew.pkt` | FAIL (interpreter says allow, BPF would say drop) | State pre-populated at bucket 0 with count=5 against threshold=3. BPF runtime (unavailable here) would look up bucket 0, find 5 ≥ 3, drop. AST interpreter looks up bucket 12345, finds nothing, falls through to "0 ≥ 3" = false, allow. Class A divergence by construction. |

The state-bearing case demonstrates the divergence is reachable
from a single corpus author's mistake — write `icmp(src_port=...)`
in the builder, set state on bucket 0, and the two oracles
disagree on a packet that, on the wire, is unambiguous.

## Root cause

Loader bug, two surfaces:

1. **`parse_builder` doesn't validate field names.** `fwl/fwl/pkt.py:133`
   accepts any `name=value` from the split args.
2. **Builder branches copy the full user dict into decoded.** Lines 216,
   231, 240. For `tcp`/`udp` this is mostly fine because every
   builder field is a valid packet field. For `icmp` it leaks
   non-ICMP attributes into the interpreter's view.

Both surfaces need to change. A field-table lookup at parse time
(`if name not in TCP_FIELDS: raise ValueError(...)`) is the simplest
fix and matches the spec's error wording verbatim.

## Resolution

Applied per the Fix section. `parse_builder` (`fwl/fwl/pkt.py`) now
consults `_BUILDER_FIELD_TABLE` and raises
`ValueError("unknown <proto> field '<name>'")` for any name outside
each constructor's documented set. Companion runner change in
`fwl/fwl/runner.py` adds a `loads:` expected key to the `.pkt`
schema so the two PoC cases (`icmp_with_unknown_src_port_field.pkt`
and `icmp_unknown_field_state_skew.pkt`) now declare `loads: false`
and serve as anti-regression guards. 284/284 corpus cases pass.

## What this rules out

- Compiler bug in the rate_limit gate predicate. The interpreter and
  emitter both compute `cur >= threshold` correctly; the divergence
  is upstream, in what `cur` is keyed by.
- Spec ambiguity on what an ICMP builder accepts. The table at
  `PKT_V01_SPEC.md:222-228` is complete and explicit (only `src_ip`,
  `dst_ip`).
- Class A in valid programs that obey the analyzer's protocol
  guards: the analyzer prevents rules from reading port/flag fields
  for ICMP packets. The divergence here is in `rate_limit` keying,
  which is *not* gated by analyzer guards.

## What this hunt did NOT cover

- BPF_PROG_TEST_RUN at kernel level — the environment lacks
  unprivileged BPF, so the divergence is provable only by reading
  both oracles' code. With BPF runtime enabled,
  `icmp_unknown_field_state_skew.pkt` would visibly fail in the
  bpf oracle while the interpreter silently allowed.
- Other "weird" builder field combinations: `tcp(garbage=1)`,
  `udp(syn=true)`, `icmp(syn=true)` — all silently accepted. A
  `tcp(syn=true, syn=false)` duplicate also takes the LAST value,
  with no warning. None of these were file-promoted; the icmp case
  is the most operationally surfacing one.
- Confirmed: `_ipv4_to_bytes` does correctly reject string types
  for an int IP — a partial validation already exists at line 154,
  it just isn't applied to the field name itself.

## Severity

Medium. The bug is in test infrastructure, not in the FWL compiler
or the BPF runtime; production traffic isn't affected. But the
verification methodology *requires* the three oracles to agree on
the same packet, and the methodology document
(`F_DEVELOPMENT_METHODOLOGY.md`) treats the .pkt loader as part of
the trusted base. A loader bug that lets the oracles see different
packets erodes that trust and risks future regressions hiding in
"agreement on the wrong packet" or "disagreement on the right
packet."

## Fix

Two-line change in `parse_builder`:

```python
_FIELD_TABLE = {
  "tcp": {"src_ip", "dst_ip", "src_port", "dst_port", "syn", "ack"},
  "udp": {"src_ip", "dst_ip", "src_port", "dst_port"},
  "icmp": {"src_ip", "dst_ip"},
}

def parse_builder(text):
  match = _BUILDER_RE.match(text)
  if not match:
    raise ValueError(f"unrecognized builder expression: {text!r}")
  proto = match.group("proto")
  allowed = _FIELD_TABLE[proto]
  fields = {"proto": proto}
  for name, raw_value in _split_args(match.group("args")):
    if name not in allowed:
      raise ValueError(f"unknown {proto} field {name!r}")
    fields[name] = _parse_value(raw_value)
  return fields
```

The companion `state` validation is its own finding (see
`finding/2026-04-25-pkt-loader-state-rule-idx-not-validated`).
