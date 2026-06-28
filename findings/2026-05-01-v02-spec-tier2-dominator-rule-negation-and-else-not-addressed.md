---
id: finding/2026-05-01-v02-spec-tier2-dominator-rule-negation-and-else-not-addressed
type: finding
protocol: [tcp, udp, icmp, icmp6]
builtins: []
severity: high
layer: spec
pattern_tags: [spec-ambiguity, tier2, protocol-guards, dominator, negation, else-branch, ipv6, undefined-behavior]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: [finding/2026-05-01-v02-spec-tier2-pkt-proto-no-dominator-rule, finding/2026-05-01-v02-spec-tier2-nested-if-condition-reads-not-covered-by-dominator-rule, finding/2026-05-01-v02-spec-tier2-stmt-level-ipv6-l3-read-undefined-value]
---

# 2026-05-01-v02-spec-tier2-dominator-rule-negation-and-else-not-addressed

## Summary

The dominator rule for L3 v4/v6 reads (`FWL_V02_SPEC.md:614-622`)
enumerates *positive* guard-establishing conditions but does not say
what happens for `not`-wrapped conditions, `else` branches, or `elif`
chains where the establishing predicate fails. Reaching the body of

```python
if not (pkt.src_ip6 == ::1):
  addr = pkt.src_ip6        # statement-position read
```

means the comparison evaluated to **false**, which can happen for
two distinct reasons:

1. The packet is IPv6 and `src_ip6 != ::1`. v6 parse succeeded.
2. The packet is **not** IPv6 (EtherType ≠ `0x86DD`). The
   comparison-against-an-unreadable-field rule from v0.1 makes the
   comparison false. v6 parse did **not** succeed.

So inside the `if not (...)` body, `pkt.src_ip6` may or may not be
readable. Reading it produces undefined bytes on the
case-(2) path. The dominator rule as written does not say whether this
program is well-formed.

The `else` branch of `if pkt.src_ip6 == ::1: ... else:` has the same
issue. So does an `elif` chain where every preceding condition fails.

Class A (interpreter and BPF runtime will diverge on non-IPv6
packets that reach a `not`-guarded body, because the spec gives no
rule). Filed as spec-layer because the root cause is the missing
rule — the dominator analysis itself is fine, but its predicate of
"a condition that reads pkt.src_ip6 directly" is too coarse to
distinguish positive from negative reaches.

## Locations

### What the spec says about v6/v4 dominator establishment

`FWL_V02_SPEC.md:614-617` (in the Statement- and inner-condition
dominator rule, after the cross-statement-`pkt`-reads paragraph
header):

> - A read of `pkt.src_ip6`, `pkt.dst_ip6` is valid only at program
>   points dominated by a guard that establishes the packet is IPv6.
>   Two ways to satisfy that guard:
>   - A condition that reads `pkt.src_ip6` or `pkt.dst_ip6` directly
>     (e.g. `if pkt.src_ip6 == ::1:` or `if pkt.src_ip6 in ::/0:`).
>     Reaching the inside of such a branch implies the v6 parse
>     succeeded.
>   - A condition that reads `pkt.proto == icmp6` ...

`FWL_V02_SPEC.md:619`:

> "Dominated by" means: every control-flow path from the function
> entry to the read passes through an `if` whose condition has
> already established the required guard. Reads inside the
> then-branch of `if pkt.proto == tcp:` are guarded; reads after
> the `if` block are not (because the false branch reaches the same
> read without the guard).

The spec is explicit that a *then-branch* gets the guard and the
*after-block* position does not. It is silent on:

| Position | Reaches it implies |
|---|---|
| `else` branch of `if cond:` | `cond` was false |
| `elif` body when previous `if`/`elif` failed | all earlier conditions were false |
| `if not cond:` body | `cond` was false |
| Body of `if cond_a or cond_b:` where only `cond_a` reads `pkt.src_ip6` | reaching does NOT imply v6 parse succeeded — `cond_b` may have been the true branch |

### v0.1 short-circuit semantics make the case (2) path real

`FWL_V01_SPEC.md:194-198`:

> Example: a rule reading `pkt.dst_port` on a packet whose IP header
> claims TCP but is truncated before the TCP header. The bounds
> check fails, the rule does not match — evaluation falls through
> to the next rule.

In Tier 2 condition position, "rule does not match" translates to
"the condition evaluates to **false**". So `pkt.src_ip6 == ::1` on a
v4 packet evaluates to false (the v6 parse failed; the comparison
defaults to no-match). `not (pkt.src_ip6 == ::1)` is therefore
**true** on every v4 packet, and the body of `if not (...)` runs
on every v4 packet.

The same logic applies to `else` and `elif` paths: every v4 packet
flows past a positive `if pkt.src_ip6 == ::1:` test (which evaluates
false) and into the `else` body.

### The OR-disjunction case has the same shape

The spec's L4 dominator rule (`FWL_V02_SPEC.md:612`) explicitly
mentions "a disjunction whose branches together cover both, per the
v0.1 union-of-branches rule" for `pkt.proto == tcp or pkt.proto ==
udp`. But the v6 dominator rule (`:614`) does not mention
disjunction at all.

So `if pkt.src_ip6 == ::1 or pkt.proto == tcp:` — what does reaching
the body imply? Possibly v6, possibly not (v4 TCP packets satisfy
the right disjunct without establishing the v6 parse). The spec
gives no rule.

## Concrete examples that the spec leaves undefined

### Negation

```python
def firewall(pkt):
  if not (pkt.src_ip6 == ::1):
    addr = pkt.src_ip6        # spec says nothing
  drop
```

On a v4 packet: condition true, body executes, `pkt.src_ip6` reads
undefined bytes from offset 22 of the Ethernet payload (the IPv6
src offset extending into where the v4 IP header would be).

### `else` branch

```python
def firewall(pkt):
  if pkt.src_ip6 == ::1:
    allow
  else:
    addr = pkt.src_ip6        # spec says nothing
  drop
```

On a v4 packet: positive comparison evaluates false, else-body runs,
same undefined read.

### `elif` chain after the v6-establishing condition

```python
def firewall(pkt):
  if pkt.src_ip6 == ::1:
    allow
  elif pkt.dst_port == 22:    # need a v4-or-v6 guard for dst_port
                              # — does the elif's predecessor satisfy
                              # that?
    drop
```

Reaching the `elif` test implies the `if` was false; v6 parse may
or may not have succeeded; the `elif` test reads `pkt.dst_port`
which itself needs a tcp/udp guard. The spec gives neither L3 nor
L4 rules for this configuration.

### OR-disjunction not enumerated

```python
def firewall(pkt):
  if pkt.src_ip6 == ::1 or pkt.proto == tcp:
    addr = pkt.src_ip6        # spec says nothing
  drop
```

Reaching the body: v6-and-`::1`, OR (v4 or v6) TCP. The v4 TCP path
does not establish v6.

## Why this matters

Two implementers diverge on the same source:

| Reading | Behaviour |
|---|---|
| Positive-only: only the THEN branch of a positively-guarded `if` is dominated | Negation/else/elif/disjunction examples are all rejected with "v6 read on a path not guarded by an IPv6-establishing condition" |
| "If the condition mentions pkt.src_ip6, the body is guarded" (a literal reading of `:614`'s wording, ignoring polarity) | All the examples above are accepted; runtime reads garbage on v4 packets |

Both are spec-conformant. The `.pkt` author writing
`pkt.src_ip6` reads in `else` blocks (a not-uncommon idiom: "if it
was a specific v6 host, allow; otherwise, log the source v6
address") gets compile-success on one implementation, compile-error
on another.

The downstream Class-A oracle drift is on the
"compile-success-with-wrong-runtime-behaviour" branch: when the
program compiles, the BPF emitter and AST interpreter must both
implement the case-(2) path the same way. The spec does not specify
what that path produces (zero? garbage from the Ethernet payload?
implicit fall-through to default?), so the two oracles can pick
different sentinels.

## The same hole for IPv4 L3 and L4 guards

`FWL_V02_SPEC.md:622` extends the dominator rule to v4 L3 reads with
the same "Two ways to satisfy that guard" structure: a positive read
of `pkt.src_ip` or `pkt.dst_ip`, or `pkt.proto == icmp`. The
negation/else/elif/disjunction questions apply identically.

`FWL_V02_SPEC.md:612` does mention disjunction for the L4 case
(tcp-or-udp), but it does not mention negation or else. So the same
issue affects:

```python
def firewall(pkt):
  if not (pkt.proto == tcp):
    port = pkt.dst_port      # is the read guarded?
  drop
```

`not (pkt.proto == tcp)` is true for all non-TCP packets including
non-IP. The dominator rule was meant to keep us from reading
`pkt.dst_port` on non-TCP/UDP frames; under the literal reading
("the condition mentions `pkt.proto`, so it's guarded"), this
program compiles. Under the polarity-aware reading, it doesn't.

## Proposed fix

Edit `FWL_V02_SPEC.md:614` (the v6 bullet of the dominator rule) and
`:622` (the v4 paragraph) to spell out polarity and disjunction.
Replace each "Two ways to satisfy that guard:" sub-list with one
canonical paragraph:

> The required guard is **established along a control-flow path** when
> the path enters the *then-branch* of an `if`/`elif` whose condition
> evaluates to true *because of* a sub-expression that reads the
> guarding field. Specifically:
>
> - The then-branch of `if pkt.src_ip6 ==/!= <ipv6 expr>:` or
>   `if pkt.src_ip6 in <ipv6 set>:` (and analogously `pkt.dst_ip6`,
>   `pkt.proto == icmp6`) establishes the v6 guard.
> - The same condition under `not`, in an `else`/`elif` body, or
>   inside a disjunction whose other branches do not also establish
>   the guard, **does not establish** the v6 guard. The body of
>   `if not (pkt.src_ip6 == ::1):` and the `else:` branch of
>   `if pkt.src_ip6 == ::1: ... else: ...` are *not* dominated.
> - A disjunction `cond_a or cond_b` establishes the guard for the
>   then-branch only when *both* `cond_a` and `cond_b` independently
>   establish it (per the v0.1 union-of-branches rule).
>
> Reads in non-establishing paths fail the dominator check with the
> existing "'<field>' read on a path not guarded by ..." error.

Apply the same paragraph (with the appropriate field/family list)
to the L4 dominator (`:612`) and the IPv4 L3 dominator (`:622`) so
the rule is uniform.

A Compile-errors row addition is not required — the existing rows
already match these messages, and the dominator analysis the
analyser must already implement (per `:620`'s "the same pass") is
naturally polarity-aware.

## Class

Class A (cross-oracle drift on `not`/`else`/`elif`/disjunction
paths over non-IPv6 / non-TCP packets) with a spec-layer root cause.
Spec underspecifies polarity and the union-of-branches rule for
L3 family guards.
