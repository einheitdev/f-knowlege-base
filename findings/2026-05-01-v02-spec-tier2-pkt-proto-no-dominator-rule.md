---
id: finding/2026-05-01-v02-spec-tier2-pkt-proto-no-dominator-rule
type: finding
protocol: [tcp, udp, icmp, icmp6]
builtins: []
severity: high
layer: spec
pattern_tags: [spec-ambiguity, tier2, protocol-guards, undefined-behavior, l3-parse]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: [finding/2026-05-01-v02-spec-tier2-stmt-level-pkt-read-on-wrong-protocol, finding/2026-05-01-v02-spec-tier2-stmt-level-ipv6-l3-read-undefined-value]
---

# 2026-05-01-v02-spec-tier2-pkt-proto-no-dominator-rule

## Summary

The Tier 2 statement-level dominator rule (added to resolve the
related stmt-level holes) enumerates every `pkt` field that needs a
guard *except* `pkt.proto` itself. So `p = pkt.proto` at the top of a
function body — with no preceding L3 guard — is grammar-legal,
analyser-permitted (no rule rejects it), and runtime-undefined when
the packet is not IP (ARP, LLDP, MPLS, raw 802.1Q with a non-IP
inner, etc.).

Class A (interpreter and BPF runtime can land on different answers
for the same non-IP packet because the spec gives them no rule);
filed as spec-layer because the root cause is the missing rule.

## The hole

### What the dominator rule covers

`FWL_V02_SPEC.md:599-611` (Statement-level `pkt` reads — guard
dominator rule):

- `pkt.src_port`, `pkt.dst_port` — must be dominated by `pkt.proto
  == tcp` or `pkt.proto == udp`.
- `pkt.tcp.syn`, `pkt.tcp.ack` — must be dominated by `pkt.proto ==
  tcp`.
- `pkt.src_ip6`, `pkt.dst_ip6` — must be dominated by an
  IPv6-establishing guard.
- `pkt.src_ip`, `pkt.dst_ip` — must be dominated by an
  IPv4-establishing guard.

### What it doesn't cover

`pkt.proto` reads in statement position (e.g. `p = pkt.proto`).

`pkt.proto` IS in the v0.2 `value_field` production
(`FWL_V02_SPEC.md:1123-1127`, after the
`finding/2026-05-01-v02-spec-grammar-value-field-missing-proto-and-tcp-fields`
fix). `value_field` is admitted as a `scalar_expr`
(`FWL_V02_SPEC.md:1107-1110`). So the grammar accepts
`p = pkt.proto` as a statement.

But `pkt.proto` itself requires the L3 parse to succeed:

- For an IPv4 frame (EtherType `0x0800`), `pkt.proto` is byte 9 of
  the IP header — needs ≥ 14 + 9 + 1 bytes.
- For an IPv6 frame (EtherType `0x86DD`), `pkt.proto` is the
  `next_header` byte at offset 14 + 6 — needs ≥ 14 + 7 bytes.
- For any other EtherType, there is no L3 to read.

The spec at `FWL_V02_SPEC.md:101-103` describes the EtherType
dispatch ("Other EtherType values fall through every rule (no
match), reaching the default action") in the context of *rule
predicates*. Tier 2 statement reads are a different surface — they
do not "fall through".

### Concretely, what the spec lets the user write

```python
def firewall(pkt):
  p = pkt.proto                 # statement read of pkt.proto;
                                # no preceding L3 guard
  if p == tcp:
    drop
  allow
```

For an ARP packet (EtherType `0x0806`), this program either:

- (a) Reads garbage bytes from the Ethernet payload at offset 23
      (the v4-proto offset), assigns whatever value to `p`, and
      then the `if` may or may not match — undefined.
- (b) Aborts the function with implicit `allow` (or `drop`,
      depending on how the runtime handles unreadable fields).
- (c) Sets `p` to a defined sentinel (zero, e.g.).

The spec picks none of these. The interpreter and emitter can land
on different answers; cross-oracle drift on the corpus surfaces as
"BPF and interp disagree" without revealing that the *spec* is what
disagrees.

### The companion holes were fixed; this one slipped

The two related findings — `tier2-stmt-level-pkt-read-on-wrong-protocol`
and `tier2-stmt-level-ipv6-l3-read-undefined-value` — were resolved
by adding the dominator rule at `FWL_V02_SPEC.md:599-611`. That rule
covers L4 fields and L3 v4/v6 fields. But `pkt.proto` is itself a
"L3-derived" field (you must parse the L3 header to find it), and
the dominator rule's enumeration of "what needs guards" is missing
this one entry.

The omission is easy to miss because in Tier 1 `pkt.proto` is the
guard — it appears on the LHS of the very `pkt.proto == tcp`
comparison the dominator rule talks about. So the rule's authors
(reasonably) treated `pkt.proto` as a primitive. But Tier 2 makes
`pkt.proto` readable in arbitrary statement positions, where it is
no longer a guard but a guarded read.

## Three possible resolutions

| Resolution | What changes |
|---|---|
| (a) Extend the dominator rule to `pkt.proto` | "A read of `pkt.proto` in statement position is valid only at points dominated by an L3-establishing guard — a condition that reads any of `pkt.src_ip`, `pkt.dst_ip`, `pkt.src_ip6`, `pkt.dst_ip6`, or that compares `pkt.proto` against a `proto_keyword` (which is itself a v4 or v6 guard, since each keyword is family-specific or family-overloaded — see line 132-138)." Note the recursion: `pkt.proto == tcp` is admitted as a guard *for* the `pkt.proto` read, which is fine because the comparison's reading of `pkt.proto` is short-circuited by EtherType per v0.1 semantics. |
| (b) Define an L3-fall-through behaviour | "If `pkt.proto` is read in statement position on a non-IP packet, the function aborts and returns implicit `allow`." Symmetric with (b) of the L4 hole's resolution. |
| (c) Sentinel value | "If `pkt.proto` is unreadable, the local takes value 0 (a reserved IPPROTO that no `proto_keyword` matches)." Probably the easiest to implement but invents a value the spec must then also document everywhere `pkt.proto` reads appear. |

(a) is the cleanest and matches the established style of the
existing dominator rule.

## Proposed fix

Add one bullet to `FWL_V02_SPEC.md:599-611`:

> - A read of `pkt.proto` is valid only at program points dominated
>   by an L3-establishing guard. L3-establishing guards are any
>   condition that reads `pkt.src_ip`, `pkt.dst_ip`, `pkt.src_ip6`,
>   or `pkt.dst_ip6` (the parse short-circuits on EtherType, so
>   reaching the inside implies the L3 parse succeeded for the
>   relevant family). A `pkt.proto == <proto_keyword>` condition
>   does **not** by itself satisfy the guard — its reading of
>   `pkt.proto` is the surface this rule constrains, not the
>   guard. (In a Tier 1 *condition* `pkt.proto == tcp` is fine by
>   v0.1 short-circuit semantics, but in a Tier 2 *statement* a
>   bare `p = pkt.proto` needs a separately-established L3 guard.)

And add a Compile-errors row:

| Condition | Error |
|---|---|
| `pkt.proto` read in statement position not dominated by an L3-establishing guard | `error: 'pkt.proto' read on a path not guarded by an L3-establishing condition` |

Then update the dogfood example sanity-check ("every L4 read sits
inside an `if pkt.proto == tcp`...") to also cover the `pkt.proto`
read-in-statement case if the corpus introduces one.

## Class

Class A (interpreter/BPF will diverge on non-IP packets). Spec-layer
because the spec has no rule for the case.
