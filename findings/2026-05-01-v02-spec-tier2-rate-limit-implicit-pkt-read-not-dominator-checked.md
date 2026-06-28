---
id: finding/2026-05-01-v02-spec-tier2-rate-limit-implicit-pkt-read-not-dominator-checked
type: finding
protocol: [tcp, udp, icmp, icmp6]
builtins: [rate_limit]
severity: high
layer: spec
pattern_tags: [spec-ambiguity, tier2, dominator-rule, rate-limit, implicit-read, undefined-behavior]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: [finding/2026-05-01-v02-spec-tier2-stmt-level-pkt-read-on-wrong-protocol, finding/2026-05-01-v02-spec-rate-limit-per-src-ip-with-v6-only-predicate-dead-rule]
---

# 2026-05-01-v02-spec-tier2-rate-limit-implicit-pkt-read-not-dominator-checked

## Summary

The Tier 2 dominator rule (`FWL_V02_SPEC.md:617-635`) covers
*user-text* reads of `pkt.<field>` — those that appear as a
statement-position RHS or as an inner-condition operand. It says
nothing about the *implicit* `pkt.<field>` read that
`rate_limit(N, per=<field>)` performs to compute its bucket key.

The implicit read has identical soundness requirements: on a
non-TCP/UDP packet `pkt.src_port` is unreadable, on a non-IPv4
packet `pkt.src_ip` is unreadable, etc. With no dominator check
covering it, the spec admits programs whose `rate_limit` call
happens on a packet for which the bucket key is undefined — and
provides no rule for what the `if`-condition's truth value is in
that case.

The single existing compile-error row for `per=src_ip` /
`per=dst_ip` (`:247`) addresses only one corner: the call site is
reachable **only** on v6 packets. Every other configuration —
reachable on a mix of families, reachable on non-IP frames,
reachable on a v4 packet but with `per=src_port` and no
TCP/UDP guard — is unreached by any spec rule.

Class A semantically (interpreter and BPF runtime can land on
different answers — false vs garbage-bucket-key vs
implicit-fallthrough — for the same packet). Filed as spec-layer
because the root cause is the dominator rule's silence on
`rate_limit`'s implicit read.

## The hole

### What the dominator rule covers

`FWL_V02_SPEC.md:617-619`:

> Both bare-field statement-position reads and inner-condition
> position reads are governed by the same control-flow dominator
> check:
>   - A read of `pkt.src_port`, `pkt.dst_port` is valid only at
>     program points dominated by a `pkt.proto == tcp` or
>     `pkt.proto == udp` guard ...

The two contexts called out are (1) statement-position RHS of an
assignment and (2) inner-condition position. A `rate_limit(...,
per=src_port)` call is neither: it is an opaque built-in that the
user invokes. The user-text does not contain `pkt.src_port` — only
the keyword `src_port` after `per=`.

### What `rate_limit` actually reads

`FWL_V02_SPEC.md:187-192`:

> The v0.1 `rate_limit` modifier accepts
> `per=src_ip|dst_ip|src_port|dst_port`. The bucket key is
> whichever field's runtime value is current at the packet.

So `rate_limit(10, per=src_port)` reads the packet's `src_port`
byte (a 16-bit field two bytes into the L4 header). On a non-TCP
non-UDP packet — ICMP, ICMPv6, or a non-IP frame — that field is
unreadable.

### What the spec does *not* cover

The Compile-errors table at `:247` covers exactly one case:

> `rate_limit(..., per=src_ip)` or `per=dst_ip` reachable only on
> v6 packets — either as a Tier 1 `limited by` modifier on a
> v6-only predicate, or as a Tier 2 `if rate_limit(...):` site
> whose dominator-analysis path establishes only the v6 family.

The "reachable only on v6" qualifier is narrow. The following
programs satisfy *no* compile-error row in the spec:

```python
# Program A — per=src_port at top of function, reachable on EVERY packet.
@xdp(eth0)
def firewall(pkt):
  if rate_limit(10, per=src_port):
    drop
  allow
```

```python
# Program B — per=src_ip at top of function. Spec gap: not v6-only,
# so :247 doesn't fire. Reaches on v4, v6, and non-IP frames.
@xdp(eth0)
def firewall(pkt):
  if rate_limit(10, per=src_ip):
    drop
  allow
```

```python
# Program C — per=src_port inside a v6 guard with no L4 guard.
# pkt.src_port is unreadable on v6 packets without a TCP/UDP next-header
# (which the v6 guard alone doesn't establish).
@xdp(eth0)
def firewall(pkt):
  if pkt.src_ip6 in ::/0:
    if rate_limit(10, per=src_port):
      drop
  allow
```

For each of these, on a packet where the implicit `pkt.<field>`
read fails, the spec answers nothing about:

- Whether the analyser rejects the program.
- The truth value of the `if` condition.
- The state of the bucket key (garbage from the L4 offset, zero
  sentinel, or "no bucket" / fall-through).

### Why the v0.1 wording doesn't transfer

`FWL_V02_SPEC.md:191-192` says, for the Tier 1 v6-vs-`per=src_ip`
case: "the field is unreadable, so the per-bucket key is undefined
and the rule falls through." Two problems lifting this to Tier 2:

1. **There is no "rule" to fall through.** Tier 2 has statements,
   not rules. The user wrote `if rate_limit(...): drop` — the only
   meaningful runtime semantics for the if-condition is `true` or
   `false`. The "rule falls through" wording maps to neither.

2. **The :191-192 paragraph only covers `per=src_ip`/`per=dst_ip`
   on v6.** It does not cover `per=src_port`/`per=dst_port` on
   ICMP, ICMPv6, or non-IP frames. The other corners are
   genuinely silent.

### Composition with the existing dominator rule is incoherent

The dominator rule rejects:

```python
def firewall(pkt):
  port = pkt.src_port      # ERROR: not dominated by pkt.proto == tcp/udp
  drop
```

But admits:

```python
def firewall(pkt):
  if rate_limit(10, per=src_port):  # implicit pkt.src_port read; unchecked
    drop
  allow
```

The two reads are at the same program point with the same
soundness requirement. The first is a compile error; the second is
silently accepted. A user who hits the first error and works
around it by switching to the rate_limit form gets a program the
spec admits but cannot say what it does.

## Three possible resolutions

| Resolution | What changes |
|---|---|
| (a) Extend the dominator rule to cover `rate_limit`'s implicit read. | Add a bullet: "`rate_limit(N, per=<field>)` is treated, for the dominator check, as if the call's per-position implicitly read `pkt.<field>` at the call site. The same dominator rules apply: `per=src_port`/`per=dst_port` requires a `pkt.proto == tcp` or `pkt.proto == udp` dominator; `per=src_ip`/`per=dst_ip` requires an IPv4-establishing dominator." Programs A, B, C above all become compile errors. The single existing `:247` row generalises to "reachable on any packet for which `per=<field>` is unreadable." |
| (b) Define a runtime fall-through for the implicit-read failure. | Add a Semantics bullet: "When `rate_limit(N, per=<field>)` is evaluated on a packet for which `pkt.<field>` is unreadable, the call evaluates to `false` (the bucket cannot be incremented or compared, so the rate-limit cannot be considered "exceeded")." This pushes the rule to runtime — the analyser remains permissive but the interpreter and emitter share a defined answer. |
| (c) Reject all `rate_limit(...)` calls except those reachable only when `per=<field>` is provably readable. | Strictest: add an analyser pass that walks the dominator state at each `rate_limit` call site and rejects any call whose `per=<field>` is not satisfied by the active dominator set. Equivalent to (a) but as a separate analyser pass rather than as an extension of the dominator rule. |

(a) and (c) produce the same accept/reject set; (a) is the
smaller spec change. (b) is the lowest-risk change for users (no
new errors) but requires the runtime to define the bucket-key
fallback explicitly, which is currently undefined.

## Suggested fix (option (a))

Add to `FWL_V02_SPEC.md:617-619`:

> - A `rate_limit(N, per=<field>)` call's `per=<field>` is treated
>   as an implicit read of `pkt.<field>` at the call site, and is
>   subject to the same dominator check as a user-text read of
>   `pkt.<field>`. `per=src_port` and `per=dst_port` require a
>   `pkt.proto == tcp` or `pkt.proto == udp` dominator;
>   `per=src_ip` and `per=dst_ip` require an IPv4-establishing
>   dominator. A `rate_limit` call whose `per=<field>` is not
>   dominated is a compile error: `error: rate_limit(per=<field>)
>   call site does not dominate the implicit read of pkt.<field>`.

Subsumes the existing `:247` row (which becomes a special case of
the general rule) and closes the silent-acceptance corners
(Programs A, B, C above).

## Suggested corpus item

```yaml
program: |
  @xdp(eth0)
  def firewall(pkt):
    if rate_limit(10, per=src_port):
      drop
    allow

# Pre-fix: spec admits the program; runtime behaviour on
# non-TCP/UDP packets is undefined; interpreter and BPF can
# disagree.
# Post-fix (option a): compile error.
expected:
  compiles: false
  compile_error_pattern: "rate_limit\\(per=src_port\\) call site does not dominate the implicit read of pkt.src_port"
```

A second discriminator that exercises the v4-only `per=src_ip`
case at the top of a v6-active function:

```yaml
program: |
  @xdp(eth0)
  def firewall(pkt):
    if pkt.src_ip6 in ::/0:    # touches v6, activates v6 parse
      log
    if rate_limit(10, per=src_ip):
      drop
    allow

# Pre-fix: not v6-only at the rate_limit site (it's the
# fall-through path, reachable on v4 too), so :247 doesn't fire.
# Spec accepts; runtime undefined on v6 packets where pkt.src_ip
# is unreadable.
# Post-fix (option a): compile error — per=src_ip needs a v4 dom.
expected:
  compiles: false
```

## Class

Class A in the corpus once the analyser exists (the interpreter
and BPF emitter will pick different fallback semantics for the
implicit unreadable-field read), filed at the spec layer because
the root cause is the dominator rule's silence on `rate_limit`'s
implicit read.
