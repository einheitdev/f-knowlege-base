---
id: finding/2026-05-01-pkt-loader-accepts-noncanonical-ipv6-in-builder-fields
type: finding
protocol: [tcp, udp, icmp6]
builtins: []
severity: low
layer: loader
pattern_tags: [pkt-loader-validation, spec-conformance, ipv6, rfc-5952, silent-acceptance]
status: fixed
source_file: fwl/fwl/pkt.py
created: 2026-05-01
pkt_path: corpus/from_hunt/2026-05-01_v6_internal_hunt/v6_builder_noncanonical_ip.pkt
related: [finding/2026-04-25-pkt-loader-unknown-builder-fields]
---

# 2026-05-01-pkt-loader-accepts-noncanonical-ipv6-in-builder-fields

## Summary

`fwl/fwl/pkt.py` (`_ipv6_to_bytes`, `_build_v6_packet`) silently
accepts non-canonical IPv6 strings in `tcp6`/`udp6`/`icmp6` builder
fields. `PKT_V02_SPEC.md` line 303 mandates the load-time error:

> | IPv6 builder field with non-canonical RFC 5952 form | `error: builder <field>: '<value>' must be RFC 5952 canonical IPv6` |

The implementation passes the raw string to
`ipaddress.IPv6Address`, which accepts every valid RFC 4291 form
(uppercase hex, leading zeros, expanded `:0:0:` runs, embedded
hex `::ffff:0102:0304` for v4-mapped, etc.) without enforcing the
canonical form. The decoded `src_ip6`/`dst_ip6` keys carry the raw
string, which the interpreter then re-parses.

## Hypothesis

The FWL parser already enforces RFC 5952 canonicality on IPv6
literals in `.fw` source — `_parse_ipv6` in `parser.py` rejects
non-canonical input with the exact error string the spec
prescribes. The same canonicality discipline should apply to .pkt
builder fields per the PKT v0.2 error table; otherwise corpus
authors can write tests that load and pass against the
implementation while documenting non-existent canonical input.
The mismatch between FWL-source strictness and .pkt-builder
leniency is exactly the kind of silent loader gap the
2026-04-25 fix run hardened (see related: builder-fields finding).

## Evidence

`corpus/from_hunt/2026-05-01_v6_internal_hunt/v6_builder_noncanonical_ip.pkt`:

```yaml
test_packet:
  builder: tcp6(src_ip="FC00:0:0:0:0:0:0:1", dst_port=80)

expected:
  compiles: true
  bpf_action: drop
```

The string `FC00:0:0:0:0:0:0:1` is non-canonical on three RFC-5952
counts (uppercase hex, expanded zero runs, no `::` collapse). Per
spec the loader must reject the file. Run output:

```
PASS  v6_builder_noncanonical_ip.pkt  (...)
      spec         pass
      interpreter  pass
      bpf          skip  [...clang compile passed]
```

The case loads cleanly and the test runs to completion. To assert
the spec'd behaviour, the case would carry `expected.loads: false`
once the loader is fixed.

Cross-check: the same string used as a *program-source* literal
(`drop if pkt.src_ip6 == FC00:0:0:0:0:0:0:1`) is correctly rejected
by `parser._parse_ipv6` with the canonical-form error. So the
canonicality discipline is present in one half of the loader
(parser) and missing in the other (.pkt builder).

## Surface area

The bug applies to all three v6 builders' `src_ip` and `dst_ip`
fields:

| Builder | Field | Currently accepts |
|---|---|---|
| `tcp6` | `src_ip` | any RFC 4291-valid form |
| `tcp6` | `dst_ip` | any RFC 4291-valid form |
| `udp6` | `src_ip` | any RFC 4291-valid form |
| `udp6` | `dst_ip` | any RFC 4291-valid form |
| `icmp6` | `src_ip` | any RFC 4291-valid form |
| `icmp6` | `dst_ip` | any RFC 4291-valid form |

The non-canonical forms accepted are exactly those `parser.py`
rejects: uppercase hex, expanded zero hextets, missing `::`
collapse for runs ≥ 2 zeros, and the deprecated `::ffff:0102:0304`
hex form for IPv4-mapped addresses (canonical is the dotted-quad
form).

## Recommended fix

Add canonicality validation to `_ipv6_to_bytes` (or a new
helper called from `_build_v6_packet`) that mirrors
`parser._canonical_ipv6`:

```python
def _ipv6_to_bytes(addr: Any, field: str = "ip") -> bytes:
  if not isinstance(addr, str):
    raise ValueError(...)
  try:
    parsed = _ipaddress.IPv6Address(addr)
  except _ipaddress.AddressValueError as e:
    raise ValueError(f"{field}: not a valid IPv6 address: {addr!r}") from e
  expected = _canonical_ipv6(addr, parsed)  # mirror parser.py
  if addr != expected:
    raise ValueError(
      f"builder {field}: {addr!r} must be RFC 5952 canonical IPv6 "
      f"(expected {expected!r})"
    )
  return parsed.packed
```

The canonicalization helper can be lifted into a shared module so
parser and builder can both import it; today they have separate
copies of the same logic. (`parser._canonical_ipv6` is already
correct; only the builder is missing the call.)

## Why this matters

A test that documents `src_ip="FC00:0:0:0:0:0:0:1"` is misleading
to a reader — the .fw source it tests can't even contain that
literal. Failing the load early forces every corpus author to type
RFC-5952 canonical forms (`fc00::1`), which is also what the
.fw program will read. Catch the divergence at load, not at
interpret/run.
