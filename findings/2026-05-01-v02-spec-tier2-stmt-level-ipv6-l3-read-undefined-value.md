---
id: finding/2026-05-01-v02-spec-tier2-stmt-level-ipv6-l3-read-undefined-value
type: finding
protocol: []
builtins: []
severity: high
layer: spec
pattern_tags: [spec-ambiguity, tier2, ipv6, undefined-behavior, locals]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: [finding/2026-05-01-v02-spec-tier2-stmt-level-pkt-read-on-wrong-protocol]
---

# 2026-05-01-v02-spec-tier2-stmt-level-ipv6-l3-read-undefined-value

## Summary

The earlier finding
`finding/2026-05-01-v02-spec-tier2-stmt-level-pkt-read-on-wrong-protocol`
identified a hole in the spec: Tier 2 statement-level reads of
protocol-specific `pkt` fields were undefined when the field is
unreadable for the packet at hand. The applied fix (option d, the
dominator rule) covers L4 fields and `pkt.tcp.*` only. The fix
**explicitly punts** on `pkt.src_ip6`/`pkt.dst_ip6`:

> A read of `pkt.src_ip6`, `pkt.dst_ip6` is unguarded — the runtime
> fall-through (no match on EtherType `0x0800`) substitutes for a
> static guard, identical to v0.1's `pkt.src_ip` rule.
> — `FWL_V02_SPEC.md:601`

But the "rule fall-through" abstraction is a *Tier 1* concept: in
Tier 1 the rule's predicate evaluates to false and the rule "does
not fire". In Tier 2 there is no rule to fall through; an
assignment statement must produce *some* value for the local. The
spec defines no such value. Interpreter and BPF emitter, written
independently, will land on different concrete values for the same
program / IPv4-packet pair.

This is the same Class A bug-class as the original finding,
narrowed to v6 L3 reads. The original finding's fix forgot the
symmetric leg.

## The hole, concretely

Consider:

```python
@xdp(eth0)

def firewall(pkt):
  src6 = pkt.src_ip6
  if src6 == ::ffff:1.2.3.4:
    drop
  if src6 == ::1:
    log
  allow
```

Per the spec at line 601, `src6 = pkt.src_ip6` is grammar-legal,
type-legal (the local takes type `ipv6`), and dominator-rule-legal
(no guard required for v6 L3 reads).

Now feed this program an **IPv4** packet. What value does `src6`
hold?

| Reading | Concrete `src6` | Source |
|---|---|---|
| (a) Whatever bytes are at the IPv6 source-address offset relative to the start of the frame, regardless of whether they're inside the IPv4 packet | uninitialised / garbage | naive emitter |
| (b) `::` (the all-zero address) | sentinel | analogous to v0.1's "field not readable" semantics for IPv4 fields |
| (c) The bits of the IPv4 source address, zero-extended into the IPv6 width | implementer's "obvious" interpretation | reasonable but undocumented |
| (d) The Tier 2 function aborts and the implicit `allow` returns immediately | needs a defined exit path | not described anywhere |

Each of these produces different downstream behaviour. Under (a)
the BPF verifier may even reject the program if the read goes
out-of-bounds for the IPv4 frame — but the interpreter has no such
verifier and will produce *some* answer. The two oracles disagree.

Under (b) the next line (`if src6 == ::ffff:1.2.3.4`) deterministically
falls through; under (c) it might match if the IPv4 source bits
happen to align with the literal's lower bits; under (d) the IPv4
packet exits at the first `pkt.src_ip6` read and skips the `if
src6 == ::1` check entirely. All three readings are consistent with
the prose at line 601.

## Why the original finding's fix doesn't reach

The original finding proposed two alternate fixes:

> Symmetric rule for `pkt.src_ip6`/`pkt.dst_ip6`: must be guarded
> by `pkt.src_ip6` / IPv6-touching guard, or accept the v0.2
> fall-through wording.
> — finding `2026-05-01-v02-spec-tier2-stmt-level-pkt-read-on-wrong-protocol`,
> proposed-fix section

The "fall-through wording" choice was taken (line 601). But that
wording is the *Tier 1 rule* fall-through — which doesn't define a
concrete local value when the field is unreadable in a Tier 2
statement assignment. The original finding's recommendation
implicitly assumed the fall-through wording carried defined runtime
semantics; in Tier 1 it does (the rule simply doesn't fire); in
Tier 2 it has no analogue.

The L4-field arm of the original fix (option d, the dominator rule)
has clean Tier 2 semantics — the program does not compile, so the
question of "what value does the local hold" never arises. The
v6-L3 arm, taking the alternate path, leaves the question open.

## Cross-family rule for Tier 1 vs Tier 2

`FWL_V02_SPEC.md:144-149`:

> *Cross-family rules.* A rule whose condition references
> `pkt.src_ip6` or `pkt.dst_ip6` does not match IPv4 packets (the
> parse for IPv6 fields fails on EtherType `0x0800`). … There is no
> compile error for accessing v6 fields without an explicit guard —
> the parse fall-through is the guard.

In Tier 1 this is well-defined: "the rule does not match". In
Tier 2 there is no rule. The same prose carries no semantic content
when transplanted into a Tier 2 assignment.

The dogfood example does not exercise this surface (it tests
`pkt.src_ip6` only inside `if` conditions, not in assignments), so
the corpus does not surface the disagreement until a fuzzer writes
a program that does.

## Proposed fix

Pick one of three:

### (i) Strict dominator rule (mirror of the L4 fix)

Edit `FWL_V02_SPEC.md:601` to:

> A read of `pkt.src_ip6`, `pkt.dst_ip6` in a Tier 2 *statement
> position* (the RHS of an assignment) is valid only at program
> points dominated by an `if` whose condition either references
> the same field or is a guard equivalent to "the packet is IPv6"
> (e.g. `pkt.proto == tcp` against an IPv6 frame is satisfied; an
> empty/unguarded read is a compile error).

This is the cleanest match for the L4 arm and avoids the runtime
question entirely.

### (ii) Defined sentinel value

Add a "Statement-level pkt access on unreadable field" subsection
to "Semantics" that defines:

> When a Tier 2 assignment reads a `pkt` field that is unreadable
> for the current packet (e.g. `pkt.src_ip6` on an IPv4 frame, or
> `pkt.src_ip` on an IPv6 frame), the local takes the all-zero
> value of its type (`::` for `ipv6`, `0.0.0.0` for `ipv4`, `0` for
> port and proto types). Subsequent comparisons read the sentinel
> like any normal value.

This makes the semantics explicit but adds a new concept (sentinel
values) that the rest of the spec doesn't use.

### (iii) Implicit early return

Add to "Semantics":

> When a Tier 2 assignment reads a `pkt` field that is unreadable
> for the current packet, the function immediately returns the
> implicit `allow` (or, with a future spec extension, the implicit
> default action). No further statements execute.

This is the safest runtime behaviour but is the most surprising
semantically — a single line of code becomes an early exit.

The author may also pick "(i) for L3 too", which is the symmetric
fix the original finding's proposal section listed as the first
alternative.

## Class

Class A — spec ambiguity that produces interpreter/BPF disagreement
when exercised, narrowed to a specific subset of Tier 2 surfaces.
Spec layer; fix is editorial.

The original finding's fix is documented as "complete." This
finding documents the leg that the original fix did not in fact
close.
