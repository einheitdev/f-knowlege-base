---
id: finding/2026-05-01-v02-spec-tier2-pkt-proto-icmp-as-v4-guard-vs-cross-family-icmp
type: finding
protocol: [icmp]
builtins: []
severity: high
layer: spec
pattern_tags: [spec-contradiction, tier2, dominator-rule, ipv6, cross-family, soundness, statement-position-read]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: [finding/2026-05-01-v02-spec-strict-superset-vs-cross-family-pkt-proto, finding/2026-05-01-v02-spec-tier2-pkt-proto-no-dominator-rule]
---

# 2026-05-01-v02-spec-tier2-pkt-proto-icmp-as-v4-guard-vs-cross-family-icmp

## Summary

The Tier 2 dominator-rule paragraph at `FWL_V02_SPEC.md:633` lists
`pkt.proto == icmp` as one of the two v4-establishing guards for a
statement-position `pkt.src_ip`/`pkt.dst_ip` read, justifying it
parenthetically with "icmp is a v4-only proto enum value." That is
flatly contradicted by the proto-enum table at `FWL_V02_SPEC.md:167`,
which says `icmp` (byte 1) matches "IPv4 always; IPv6 (no ext hdrs)
**only when the program activates the v6 parse path**." For any
v6-active Tier 2 program, `pkt.proto == icmp` evaluates true on a v6
frame whose `next_header == 1`, so a `pkt.src_ip` read inside the
then-branch reads the bytes at the v4-source-address offset on a
v6-shaped frame — exactly the soundness hole the dominator rule was
introduced to prevent.

Class B (spec contradiction; downstream Class A as soon as the
analyser implements line 633 verbatim and the runtime emits the v6
parse path).

## The contradiction

### Reading 1 — line 633 (Tier 2 dominator rule)

`FWL_V02_SPEC.md:633` (paragraph "*IPv4 L3 reads in statement
position obey the same dominator rule.*"):

> v4-establishing guards are conditions that read `pkt.src_ip` or
> `pkt.dst_ip` (the parse short-circuits on EtherType `0x0800` and
> reaching the inside implies success), or `pkt.proto == icmp`
> (**icmp is a v4-only proto enum value**). `pkt.proto == tcp` and
> `pkt.proto == udp` do **not** satisfy the v4 guard, since both
> proto values exist on both families.

This treats `icmp` as v4-only without qualification.

### Reading 2 — line 167 (proto-enum table)

`FWL_V02_SPEC.md:163-168`:

> | Keyword | `pkt.proto` byte | Family allowed |
> |---|---|---|
> | `tcp`   | 6  | IPv4 always; IPv6 (no ext hdrs) only when the program activates the v6 parse path |
> | `udp`   | 17 | IPv4 always; IPv6 (no ext hdrs) only when the program activates the v6 parse path |
> | `icmp`  | 1  | IPv4 always; IPv6 (no ext hdrs) only when the program activates the v6 parse path |
> | `icmp6` | 58 | IPv6 (no ext hdrs) only when the program activates the v6 parse path |

The `icmp` row sits in the same "v4 always + v6 when active" column
as `tcp` and `udp`. The enum-table prose at `:170` reinforces this:
"`pkt.proto == <kw>` evaluates true exactly when ... the packet's
family is in the keyword's "Family allowed" column".

### The two readings disagree

`pkt.proto == icmp` true ⇒ ?
| Program shape | Per :167 | Per :633 |
|---|---|---|
| v6-active (any v6 surface touched) | matches v4-icmp **or** v6-with-next_header=1 | claimed v4-only |
| v0.1-shaped (no v6 surface) | matches v4-icmp only | claimed v4-only (correct here) |

For v6-active programs the readings split. The dominator rule's
v4-only claim is a strict over-promise.

## Concrete soundness counterexample

A v0.2 Tier 2 program that activates the v6 parse path and uses
`pkt.proto == icmp` as the spec-blessed v4 guard for a
statement-position `pkt.src_ip` read:

```python
@xdp(eth0)

def firewall(pkt):
  # Touches a v6 surface, so the v6 parse path is active.
  if pkt.src_ip6 in ::/0:
    log

  # Spec line 633 says this is a valid v4-establishing guard.
  if pkt.proto == icmp:
    addr = pkt.src_ip       # statement-position v4 L3 read
    if addr == 10.0.0.1:
      drop

  allow
```

A `tcp6` packet — no, an *IPv6 packet whose `next_header` is `1`* —
hits this program. (`next_header == 1` on a v6 frame is unusual but
not malformed; nothing in IPv6 forbids byte 1 in the next-header
field, and an attacker can craft it. The proto-enum table is what
tells the analyser whether to admit such packets.)

| Step | Outcome |
|---|---|
| `if pkt.src_ip6 in ::/0:` | v6 parse succeeds, `::/0` matches every v6 packet → enter, `log`, fall through. |
| `if pkt.proto == icmp:` | v6 parse path is active, packet's `next_header == 1`, family-allowed column for `icmp` admits "IPv6 when active" → comparison evaluates **true**, enter the branch. |
| `addr = pkt.src_ip` | Statement-position v4 L3 read. v4 parse fails (EtherType is 0x86DD), so the bytes at the v4-source-address offset are *whatever happens to be at offset 14+12=26 in the frame* — i.e. the IPv6 source address's high 32 bits. **Garbage from the user's perspective**. The dominator rule was supposed to prevent this. |
| `if addr == 10.0.0.1: drop` | Compares garbage bytes against `10.0.0.1`. Some adversarially-crafted v6 source addresses will satisfy this; the packet is dropped despite the user's intent that the rule fire only on v4 ICMP from `10.0.0.1`. |

The user wrote a Tier 2 program that the spec says is well-formed
and that the dominator rule promises is sound. The runtime delivers
neither.

## Root cause

The dominator-rule paragraph at `:633` was lifted from a v0.1
mental model where `icmp` is unconditionally v4-only. The
proto-enum table at `:167` is a v0.2 addition that makes `icmp`
cross-family in v6-active programs. The two paragraphs were not
reconciled. The pattern is identical to
`finding/2026-05-01-v02-spec-strict-superset-vs-cross-family-pkt-proto`
— a v0.1-era invariant ("icmp ⇒ v4") that v0.2's cross-family rule
broke, and that the spec's later sections continue to assume.

The same paragraph at `:633` correctly excludes `tcp`/`udp` from the
v4 guard list ("both proto values exist on both families"). The
exclusion logic applies to `icmp` for the same reason once the
proto-enum table at `:167` is taken seriously — the author appears
to have applied the rule to `tcp` and `udp` and forgotten to
re-check `icmp`.

## Why this matters

The v4 dominator rule is the only thing that makes statement-position
v4 L3 reads sound in v0.2 Tier 2. If the analyser implements line
633 verbatim, then for every v6-active program with the shape above,
`pkt.src_ip` reads garbage on v6-with-next_header=1 frames — a
soundness regression the spec explicitly set out to close (per the
"strict resolution of two related spec holes" sentence at `:631`).

A user who writes `pkt.src_ip` reads with `pkt.proto == icmp` as the
guard will get correct behaviour on v0.1-shaped programs and silent
data corruption on v6-active programs. The conditional behaviour is
exactly the failure mode the strict-superset finding warned about,
re-introduced in a section that was supposed to fix it.

The class-A risk activates the moment the analyser is implemented:
- Reading 1 (verbatim :633): analyser accepts the program, runtime
  reads garbage on v6 next_header=1 frames → spec/runtime drift.
- Reading 2 (proto enum :167 takes precedence): analyser must
  reject `pkt.proto == icmp` as a v4-establishing guard whenever the
  program is v6-active → users who lift the spec's blessed
  `pkt.proto == icmp` idiom into a program that incidentally touches
  a v6 surface get a confusing rejection.

The two readings disagree on accept/reject for the same program.
Hone fuzz will surface the disagreement immediately once the
analyser exists.

## Proposed fix

The simplest spec-only fix is to qualify line 633 to match the
proto-enum table:

> v4-establishing guards are conditions that read `pkt.src_ip` or
> `pkt.dst_ip` (the parse short-circuits on EtherType `0x0800` and
> reaching the inside implies success). `pkt.proto == icmp` is **not**
> a v4-establishing guard in v0.2: in a v6-active program `icmp`
> matches v6 frames whose `next_header == 1`, so reaching the
> then-branch does not imply v4. (It does imply L3 was readable, so
> a statement-position `pkt.proto` read inside the branch is fine —
> just not a `pkt.src_ip`/`pkt.dst_ip` read.) `pkt.proto == tcp` and
> `pkt.proto == udp` likewise do not satisfy the v4 guard.

Drop the `pkt.proto == icmp` example from line 633's v4 list. The
only v4-establishing guards in v0.2 Tier 2 are the two L3 v4 field
reads — symmetric with the v6 list at `:619`, which only admits the
two L3 v6 field reads plus `pkt.proto == icmp6` (correctly v6-only
because `icmp6` is family-restricted by the enum table, unlike
`icmp`).

If the spec author wants to keep a proto-only v4 guard, it would
have to be a synthetic "no v6 next_header value" — there is no such
keyword in v0.2, so this fix is the minimum viable change.

The two related Compile-errors rows at `:715-716` are unaffected:

| `pkt.src_ip`/`pkt.dst_ip` read in statement position not dominated by an IPv4-establishing guard | `error: '<field>' read on a path not guarded by an IPv4-establishing condition` |

The error wording stays; what changes is the set of conditions the
analyser counts as IPv4-establishing.

A second-order fix the spec should also make: add an explicit
"there is no proto-only IPv4-establishing guard in v0.2" sentence
under "Edge cases" of the Tier 2 section, so a user who writes the
program above gets a `error: '<field>' read on a path not guarded by
an IPv4-establishing condition` rather than the silent-corruption
runtime path.

## Suggested corpus item

A `.pkt` test that activates the v6 parse path, sends an IPv6 frame
with `next_header == 1`, and expects the analyser to reject the
program (post-fix) — or the interpreter and BPF runtime to disagree
(pre-fix). Skeleton:

```yaml
program: |
  @xdp(eth0)

  def firewall(pkt):
    if pkt.src_ip6 in ::/0:
      log
    if pkt.proto == icmp:
      addr = pkt.src_ip
      if addr == 10.0.0.1:
        drop
    allow

packet: |
  # IPv6 frame, src=2001:db8::1 (high32 = 0x20010db8), next_header=1.
  eth(type=ipv6) / ip6(src="2001:db8::1", dst="::1", nh=1) / b"\x00"*16

# Pre-fix expected: interpreter treats the read as garbage,
#                   BPF emitter same; oracles agree on garbage but
#                   the program is unsound.
# Post-fix expected: compile error
#                   "'pkt.src_ip' read on a path not guarded by an
#                   IPv4-establishing condition".
expected:
  compiles: false
  compile_error_pattern: "pkt.src_ip read on a path not guarded by an IPv4-establishing condition"
```

The pre-fix test pins the disagreement point so a v0.2 implementer
who lifts line 633 verbatim into the analyser fires the corpus
oracle.

## Class

Class B at the spec layer; promotes to Class A in the corpus the
moment the analyser implements the dominator rule against a v6-
active program. Two paragraphs in the same document disagree on
whether `pkt.proto == icmp` is v4-establishing. The interpreter and
emitter cannot land on the same answer without picking one.
