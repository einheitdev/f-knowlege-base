---
id: finding/2026-05-01-v02-spec-proto-equality-table-vs-opaque-byte-equality
type: finding
protocol: [icmp, icmp6, tcp, udp]
builtins: []
severity: high
layer: spec
pattern_tags: [spec-contradiction, ipv6, cross-family, pkt-proto, tier2, locals, opaque-type, regression-hazard]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: [finding/2026-05-01-v02-spec-strict-superset-vs-cross-family-pkt-proto, finding/2026-05-01-v02-spec-tier2-proto-local-vs-local-comparison-disallowed-but-no-error]
---

# 2026-05-01-v02-spec-proto-equality-table-vs-opaque-byte-equality

## Summary

The v0.2 spec defines `pkt.proto == <proto_keyword>` semantics in two
places that disagree on what the comparison evaluates to for a v6 packet
whose `next_header` byte happens to equal a v4-only proto keyword's
IPPROTO (or vice-versa).

1. **Proto-enum table — family-aware.** `FWL_V02_SPEC.md:161-167`:

   | Keyword | Matches when packet is | IPPROTO |
   |---|---|---|
   | `tcp`   | IPv4 with `proto == 6` always; IPv6 with `next_header == 6` (no ext hdrs) **only when v6 path activated** | 6 |
   | `udp`   | IPv4 with `proto == 17` always; IPv6 with `next_header == 17` (no ext hdrs) **only when v6 path activated** | 17 |
   | `icmp`  | IPv4 with `proto == 1` | 1 |
   | `icmp6` | IPv6 with `next_header == 58` (no ext hdrs) | 58 |

   Reinforced at `FWL_V02_SPEC.md:168`:
   > `icmp` is IPv4-only and `icmp6` is IPv6-only.

   Under this reading, `pkt.proto == icmp` is *false* on a v6 packet
   even when `next_header == 1`, and `pkt.proto == icmp6` is *false*
   on a v4 packet even when `proto == 58`. The keyword carries hidden
   family info, so the equality is structured (family AND byte), not
   pure byte equality.

2. **Tier 2 type-rules — opaque byte equality.** `FWL_V02_SPEC.md:559`:

   > The `proto` type is opaque. Locals of type `proto` may participate
   > in equality and inequality comparisons (`==`, `!=`) against any
   > other `proto`-typed value — a `proto_keyword` token (`tcp`, `udp`,
   > `icmp`, `icmp6`), the `pkt.proto` field, or another `proto`-typed
   > local. … Equality between two opaque `proto` values has the
   > obvious meaning, so it is permitted (matches the bool-vs-bool rule
   > below).

   Combined with `FWL_V02_SPEC.md:157-158`:

   > The `pkt.proto` field reads the IPv6 fixed header's `next_header`
   > byte when the packet is IPv6.

   The "obvious meaning" of equality on two 8-bit opaque values stored
   in 8-bit slots is byte equality. Under this reading, `pkt.proto ==
   icmp` is *true* on a v6 packet with `next_header == 1`, because the
   stored byte (1) equals `icmp`'s IPPROTO (1).

The two readings give different answers on the same source line for the
same packet. This is the same shape as
`finding/2026-05-01-v02-spec-strict-superset-vs-cross-family-pkt-proto`
(which the spec resolved by making `tcp`/`udp` cross-family conditional)
but applied to `icmp`/`icmp6` and to the local-extraction case — neither
of which the earlier finding addressed.

Class A. The interpreter and BPF emitter will land on different answers
because the spec gives them no rule that resolves the conflict.

## The two concrete divergences

### Divergence 1 — `pkt.proto == icmp` on a v6 packet with next_header == 1

A program that touches a v6 surface (so the v6 parse path is active):

```python
@xdp(eth0)

drop if pkt.proto == icmp
default allow
# program also has, e.g., a separate `allow if pkt.src_ip6 in ...`
```

A `.pkt` test that fires an IPv6 frame with `next_header == 1`
(stuffed by hand — real ICMP-over-v6 uses proto 58, but the wire is
not validated):

| Implementer reading | Verdict |
|---|---|
| Reading 1 (table — family-aware) | `XDP_PASS` (default allow; `pkt.proto == icmp` is false on v6) |
| Reading 2 (opaque byte equality) | `XDP_DROP` (byte 1 == 1) |

The strict-superset finding's resolution chose Reading 1 for `tcp`/`udp`
but did not fold `icmp`/`icmp6` into the same rule. The table at
`:161-167` left `icmp` as flat "IPv4 with proto == 1" — which by the
opaque-byte-equality rule on line 559 must *also* match v6 packets with
`next_header == 1`, but the table refuses to.

### Divergence 2 — extracting `pkt.proto` to a local changes the meaning

```python
@xdp(eth0)

def firewall(pkt):
  if pkt.src_ip6 in ::/0:        # establish v6 guard
    p = pkt.proto                 # p :: proto, holds next_header byte
    if p == icmp:                 # "obvious meaning" => byte == 1
      drop
  allow
```

For an IPv6 frame with `next_header == 1`, `p` holds the byte value
`1`. By line 559's opaque-byte-equality rule, `p == icmp` is *true*
(both proto values, both byte 1). Yet the equivalent in-condition form
`if pkt.proto == icmp:` would be *false* per the table at `:161-167`
(`icmp` is "IPv4-only").

Two source-line transformations the user expects to be semantically
equivalent are not:

| Source                              | Verdict on v6 + next_header=1 |
|-------------------------------------|-------------------------------|
| `if pkt.proto == icmp: drop`        | `XDP_PASS` (table; v6 ≠ v4)   |
| `p = pkt.proto; if p == icmp: drop` | `XDP_DROP` (byte eq via line 559) |

This silently breaks an obvious refactor: hoisting `pkt.proto` into a
local. It is also the core motivation cited *for* allowing the
proto-local idiom in finding
`tier2-proto-local-vs-local-comparison-disallowed-but-no-error` —
the resolution there (allow `p == proto_keyword` equality) presumes
opaque byte equality, which contradicts the table.

### Symmetric divergence — `pkt.proto == icmp6` on a v4 packet

A v0.2 program that activates the v6 path (perhaps coincidentally):

```python
drop if pkt.proto == icmp6
allow if pkt.src_ip6 == ::1   # activates v6 path
default allow
```

A v4 frame with `proto == 58` (e.g. an injected probe, MLD-like
encapsulation, or just a malformed packet):

| Implementer reading | Verdict |
|---|---|
| Reading 1 (table — `icmp6` matches IPv6 only) | `XDP_PASS` |
| Reading 2 (opaque byte equality) | `XDP_DROP` |

## Why this matters

Three angles, in order of severity:

1. **Cross-oracle drift.** The interpreter (likely written from line
   559 — proto values are opaque, equality is byte-eq) and the emitter
   (likely written from the table at `:161-167` — emit family-conditional
   compare) will land on different answers for the same input on every
   `.pkt` test that exercises a malformed-family packet. Hone's BPF vs
   interpreter oracle disagrees, with no spec text to point at.

2. **Refactor hazard.** A user who rewrites `if pkt.proto == icmp:` as
   `p = pkt.proto; if p == icmp:` (a perfectly natural Tier 2 idiom)
   silently changes the program's behaviour on adversarially-crafted v6
   packets. Operators running v0.2 in front of public networks have to
   know which form is "correct".

3. **The proto-local feature has no consistent semantics.** The
   resolution applied to `finding/...-proto-local-vs-local-comparison`
   permits `p == otherp` and `p == pkt.proto` and `p == proto_keyword`.
   All three cases are sound under byte equality but at most one of
   them (`p == proto_keyword`) overlaps with the table's
   family-aware equality, and the overlap is incomplete (only for
   `tcp`/`udp` when the v6 path is active; never for `icmp`/`icmp6`).

## Three possible resolutions

| Resolution | Effect |
|---|---|
| (a) Make all `proto_keyword` equality byte-equality. | Drop the family-aware columns from the table at `:161-167` and rewrite each row as "IPv4 with proto == N, or IPv6 with next_header == N (no ext hdrs)". The strict-superset claim must then weaken (already partially weakened by the existing `tcp`/`udp` activation rule); v0.1 programs receive v6 packets that match `pkt.proto == tcp` only when the program *also* touches a v6 surface (i.e., never for purely-v0.1 programs — preserving strict superset). `icmp` would now match v6 packets with next_header=1 when v6 path is active; `icmp6` would match v4 packets with proto=58 when v6 path is active. The corpus must lock these cases. |
| (b) Make all `proto_keyword` equality structured (family-aware). | Strengthen line 559's "obvious meaning" to specifically *not* be byte equality when one side is a `proto_keyword` and the other side is `pkt.proto` or a `proto`-typed local that ultimately came from `pkt.proto`. The implementation must track family provenance through every assignment. Concretely: each `proto`-typed local needs a hidden family tag, mutated by every read of `pkt.proto`, used by the equality check. The motivation cited at line 559 ("matches the bool-vs-bool rule") then breaks — bool is just a bit; proto would need a hidden second bit. |
| (c) Restrict the equality forms. | Make `p == proto_keyword` (where `p` is a local of type `proto` derived from `pkt.proto`) a compile error, allowing only `pkt.proto == proto_keyword` (which uses the table) and `p == q` between two locals (which uses byte equality, harmless because no keyword is involved). This costs the user the natural refactor but eliminates the divergence. |

(a) is the cleanest and is consistent with what the implementation is
likely to do anyway — the BPF emitter cannot easily track family
provenance through Tier 2 control flow. The cost is reframing the
table as byte-equality with a v6-activation gate.

## Proposed fix (option a)

Replace `FWL_V02_SPEC.md:161-167` with:

```
| Keyword | `pkt.proto` byte value | Matches on packet family |
|---|---|---|
| `tcp`   | 6  | IPv4 always; IPv6 (no ext hdrs) only when v6 parse path is activated |
| `udp`   | 17 | IPv4 always; IPv6 (no ext hdrs) only when v6 parse path is activated |
| `icmp`  | 1  | IPv4 always; IPv6 (no ext hdrs) only when v6 parse path is activated |
| `icmp6` | 58 | IPv6 (no ext hdrs) only when v6 parse path is activated |

`pkt.proto == <kw>` evaluates to true exactly when the read of
`pkt.proto` succeeded (the L3 parse short-circuited on EtherType for
the kw's allowed family) AND the read byte equals the kw's IPPROTO.
On non-IP frames, on truncated frames, on frames with ext hdrs, and
on frames whose family is excluded by the v6-activation gate, the
comparison evaluates to false — the read short-circuits before the
byte equality test.
```

Add a paragraph after `:168`:

> *Refactor invariance.* Because `pkt.proto == <kw>` is byte equality
> on the read byte, `p = pkt.proto; if p == <kw>: …` is exactly
> equivalent to `if pkt.proto == <kw>: …` *when* `p`'s read site
> sits in the same control-flow region as the comparison (i.e., the
> assignment's L3 guard dominates the comparison). Tier 2 users may
> hoist `pkt.proto` into a local without changing program semantics.

Add an Edge case under "IPv6 Fields":

> *Adversarial v6 packet with non-standard `next_header`.* A v6 frame
> whose `next_header` byte is `1` (the v4 IPPROTO for ICMP) matches
> `pkt.proto == icmp` exactly when the program activates the v6
> parse path — an explicit consequence of byte-equality semantics.
> Operators who want to exclude this case write the family check
> explicitly: `if pkt.src_ip6 in ::/0 and pkt.proto == icmp: …` is
> v6-only by construction (the LHS is a v6 read).

And add corpus cases that lock the byte-equality on `icmp`/`icmp6`
and the `p = pkt.proto; if p == kw` refactor.

## Alternate — option (c) is simpler if the spec's intent really is
"icmp is for v4 only, end of story"

If the authors *want* `pkt.proto == icmp` to be false on v6 with
next_header == 1, then the only consistent path is to forbid the
proto-local `==` form against `proto_keyword` tokens. Concretely, add
a Compile-errors row:

| Condition | Error |
|---|---|
| `proto`-typed local compared against a `proto_keyword` (`==` / `!=`) | `error: cannot compare a proto local against a proto_keyword; the family information is not preserved across an assignment. Use `if pkt.proto == <kw>: …` directly.` |

This blocks the refactor hazard but contradicts the resolution of the
companion finding `tier2-proto-local-vs-local-comparison-disallowed-but-no-error`.
Authors must pick one direction or the other; the current spec sits in
the middle.

## Class

Class A — interpreter and emitter will reach different verdicts on
adversarial v6/v4 packets where the next_header / proto byte equals a
cross-family keyword's IPPROTO. Spec-layer because the contradiction
is between two paragraphs of the spec, not in any one implementation.
