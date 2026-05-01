---
id: finding/2026-05-01-v02-spec-tier2-example1-rate-limit-per-src-ip-violates-dominator-rule
type: finding
protocol: [tcp]
builtins: [rate_limit]
severity: high
layer: spec
pattern_tags: [spec-inconsistency, tier2, dominator-rule, rate-limit, example-vs-rule, ipv4-guard]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: [finding/2026-05-01-v02-spec-tier2-example5-violates-grammar-and-dominator-rule, finding/2026-05-01-v02-spec-tier2-pkt-proto-icmp-as-v4-guard-vs-cross-family-icmp, finding/2026-05-01-v02-spec-dogfood-tier2-v6-ssh-bypass-rate-limit]
---

# 2026-05-01-v02-spec-tier2-example1-rate-limit-per-src-ip-violates-dominator-rule

## Summary

Tier 2 worked **Example #1** — the spec's flagship "SSH brute-force
protection" program at `FWL_V02_SPEC.md:747-758` — is presented as a
positive example of Tier 2 idiom and has `rate_limit(10,
per=src_ip)` inside an `if pkt.proto == tcp and pkt.dst_port == 22:`
guard. Per the spec's own dominator rule for the implicit `pkt.<per>`
read at `:625` and the explicit IPv4-guard list at `:637`, that
program is a **compile error**: the only IPv4-establishing guards in
v0.2 are direct reads of `pkt.src_ip`/`pkt.dst_ip`, and `pkt.proto
== tcp` is excluded by name.

The spec author was clearly aware of the rule when writing the
*dogfood* example, which adds `if pkt.src_ip in [0.0.0.0/0]:` as the
v4-establishing guard before the same rate-limit. Example #1 forgot
to do the same. Either Example #1 is wrong, or the dominator rule's
v4-guard list is wrong — they cannot both be right.

Class B (spec/example contradiction). The spec's first Tier 2
example will be the first thing a corpus author tries to lift into a
positive `.pkt` test; the analyser implementing the dominator rule
will reject it.

## Locations

### Example #1 as written — `FWL_V02_SPEC.md:747-758`

```python
@xdp(eth0)

def firewall(pkt):
  if pkt.proto == tcp and pkt.dst_port == 22:
    if pkt.tcp.syn and not pkt.tcp.ack:
      if rate_limit(10, per=src_ip):
        drop
    allow
  allow
```

The spec's prose at `:761-765` describes this as a valid, idiomatic
Tier 2 program: "The first `if` enters the SSH branch only when the
proto + port guard passes. Inside, a new SSH SYN that exceeds the
per-source rate is dropped..."

### The dominator rule the example trips — `FWL_V02_SPEC.md:625`

> *`rate_limit(N, per=<field>)` performs an implicit `pkt.<field>`
> read at the call site to compute its bucket key. The dominator
> check applies to that implicit read identically to a bare-field
> read — `per=src_port`/`per=dst_port` requires a `pkt.proto == tcp`
> or `pkt.proto == udp` dominator; `per=src_ip`/`per=dst_ip`
> requires an IPv4-establishing dominator; the four `per=` fields
> all require their respective field be readable on every packet
> that can reach the call site. A `rate_limit` whose `per=` field
> is not dominated is a compile error: `error: rate_limit(per=<field>)
> call site does not dominate the implicit read of pkt.<field>`.

So `rate_limit(10, per=src_ip)` requires an **IPv4-establishing
dominator**.

### The IPv4-guard enumeration the example fails — `FWL_V02_SPEC.md:637`

> The only v4-establishing guards in v0.2 are conditions that read
> `pkt.src_ip` or `pkt.dst_ip` directly (the parse short-circuits
> on EtherType `0x0800` and reaching the inside implies success).
> `pkt.proto == tcp`, `pkt.proto == udp`, and `pkt.proto == icmp`
> do **not** satisfy the v4 guard, because every one of those
> proto keywords matches v6 frames in v6-active programs (per the
> proto-enum table). There is no proto-only IPv4 guard in v0.2;
> users who want one must include an `pkt.src_ip in [0.0.0.0/0]`
> or similar v4-only L3 read in the condition.

The example's outer guard is `if pkt.proto == tcp and pkt.dst_port
== 22:`. Neither conjunct reads `pkt.src_ip` or `pkt.dst_ip`. The
inner guard is `if pkt.tcp.syn and not pkt.tcp.ack:`. Neither bool
field is a v4 L3 read either. So at the `rate_limit(10,
per=src_ip)` call site, no IPv4-establishing guard has been
established. The implicit `pkt.src_ip` read fails the dominator
check.

### The dogfood example does the right thing — `FWL_V02_SPEC.md:1015-1022`

```python
  if pkt.proto == tcp and pkt.dst_port == 22:
    if pkt.src_ip in [0.0.0.0/0]:               # v4-establishing guard
      if pkt.tcp.syn and not pkt.tcp.ack:
        if rate_limit(10, per=src_ip):
          drop
      count ssh_seen
      allow
```

Same SSH SYN flood pattern, but with `if pkt.src_ip in [0.0.0.0/0]:`
inserted as the v4-establishing guard before the `rate_limit`. The
comment ("v4-establishing guard") is the spec author's own
acknowledgment of the rule. Example #1 is missing exactly that line.

## Why this isn't subsumed by existing findings

- `finding/2026-05-01-v02-spec-dogfood-tier2-v6-ssh-bypass-rate-limit`
  documents a different bug (the v6 bypass *of the dogfood
  example*). That example is structurally compliant with the
  dominator rule (it has the explicit v4 guard); its bug is a
  Class C intent-mismatch, not a dominator failure.
- `finding/2026-05-01-v02-spec-tier2-example5-violates-grammar-and-dominator-rule`
  is about Example #5 (the stack-budget stress-test). Different
  example, different fault — Example #5 also reads `pkt.src_port`
  bare, and uses single-line `if cond: stmt` form. Example #1 has
  neither problem; its only fault is the missing v4 guard before
  `rate_limit(per=src_ip)`.
- `finding/2026-05-01-v02-spec-rate-limit-per-src-ip-with-v6-only-predicate-dead-rule`
  is about a *Tier 1* dead-rule shape (v6 predicate + v4 `per=`).
  Example #1 is Tier 2 with a v4-shape predicate but no v4 guard.

The pattern (rate_limit(per=src_ip) without v4 dominator) is
already known at the rule level; what's new is that **the spec's
own first Tier 2 example violates it**.

## Why this matters

The Tier 2 examples are the spec's contract for what a "real" v0.2
program looks like. Corpus authors will lift them verbatim into
`.pkt` files with `expected.compiles: true`. Two distinct failure
modes follow:

1. **Analyser implementer A — strict reading of the dominator
   rule.** Rejects Example #1. Corpus case fires
   `expected.compiles: true` against an analyser that emits
   `error: rate_limit(per=src_ip) call site does not dominate the
   implicit read of pkt.src_ip`. The spec's prose says the program
   is valid; the spec's own rule says it isn't.

2. **Analyser implementer B — pragmatic reading, blesses Example
   #1.** Adds `pkt.proto == tcp` (or `udp`, `icmp`) as an
   IPv4-establishing guard so the example compiles. But the spec
   at `:637` *explicitly* excludes those keywords from the v4
   guard list with a four-sentence justification. The analyser
   now diverges from the spec's stated rule on every other
   program of the same shape — for example, in v6-active
   programs where the spec's exclusion is load-bearing
   (`pkt.proto == tcp` matches v6 there too, so a `pkt.src_ip`
   read inside is unsound).

   Worse, this implementer's "fix" reintroduces the exact
   soundness hole that `finding/2026-05-01-v02-spec-tier2-pkt-proto-icmp-as-v4-guard-vs-cross-family-icmp`
   was filed to close.

The two implementer choices land on different accept/reject
verdicts for the same program. Hone's interpreter-vs-BPF oracle
disagrees with no spec text to point at.

## Concrete `.pkt` evidence (planned for v0.2 corpus)

```yaml
# corpus/from_hunt/2026-05-01_v02_spec/tier2_example1_dominator_v4_guard.pkt
name: "Tier 2 spec Example #1 verbatim — does the analyser accept?"
# Hypothesis: per FWL_V02_SPEC.md:625+637, the implicit pkt.src_ip
# read by rate_limit(per=src_ip) requires a v4-establishing dominator.
# pkt.proto == tcp does not satisfy the v4 dominator (line 637).
# So a strict analyser must REJECT Example #1 of the Tier 2 section.
source_fw: |
  @xdp(eth0)

  def firewall(pkt):
    if pkt.proto == tcp and pkt.dst_port == 22:
      if pkt.tcp.syn and not pkt.tcp.ack:
        if rate_limit(10, per=src_ip):
          drop
      allow
    allow

# v4 TCP SYN to port 22 — what the example is meant to drop (after
# 10/sec/src), but the program may not even compile.
test_packet:
  builder: tcp(src_ip="10.0.0.1", dst_ip="10.0.0.2", dst_port=22, syn=1)

expected:
  # Strict reading of the spec rule:
  compiles: false
  compile_error_pattern: "rate_limit\\(per=src_ip\\) call site does not dominate"

  # Liberal reading (the spec's prose treats Example #1 as valid):
  # compiles: true
  # bpf_action: allow      # 1st SYN under the rate limit
```

A second `.pkt` with the dogfood-style fix locks the *correct*
shape:

```yaml
# corpus/from_hunt/2026-05-01_v02_spec/tier2_example1_dominator_v4_guard_fixed.pkt
name: "Tier 2 SSH brute-force with explicit v4-establishing guard"
source_fw: |
  @xdp(eth0)

  def firewall(pkt):
    if pkt.proto == tcp and pkt.dst_port == 22:
      if pkt.src_ip in [0.0.0.0/0]:
        if pkt.tcp.syn and not pkt.tcp.ack:
          if rate_limit(10, per=src_ip):
            drop
        allow
    allow

test_packet:
  builder: tcp(src_ip="10.0.0.1", dst_ip="10.0.0.2", dst_port=22, syn=1)

expected:
  compiles: true
  bpf_action: allow
```

The pair pins the spec's intended shape and locks the divergence
point.

## Three possible resolutions

| Resolution | Effect |
|---|---|
| (a) Fix Example #1. | Insert `if pkt.src_ip in [0.0.0.0/0]:` as the v4-establishing guard, exactly as the dogfood example already does. Cost: one extra example line plus an indent shift. |
| (b) Loosen the v4-guard list to admit `pkt.proto == tcp\|udp\|icmp` *in v0.1-shaped programs only*. | The example is v0.1-shaped (no v6 surface), so in v0.1-shaped programs `pkt.proto == tcp` is a valid v4 guard. Requires the dominator analyser to be v6-activation-aware; reintroduces the cross-family soundness hazard for v6-active programs (which the dogfood-style v4 guard is supposed to avoid). |
| (c) Add a new "v4-only program" sentinel to the spec. | Promote the v6-activation status to a first-class analyser fact; in v6-inactive programs, use a relaxed v4-guard list. Larger spec change. |

(a) is the smallest, lowest-risk change and matches the spec
author's clearly-intended idiom (the dogfood example demonstrates
they know the rule). Adopting (b) or (c) is a substantive analyser
change that should be defended on its own merits, not adopted
silently to make a single example compile.

## Proposed fix (option a)

Replace `FWL_V02_SPEC.md:749-758` with:

```python
@xdp(eth0)

def firewall(pkt):
  if pkt.proto == tcp and pkt.dst_port == 22:
    if pkt.src_ip in [0.0.0.0/0]:               # v4-establishing guard
      if pkt.tcp.syn and not pkt.tcp.ack:
        if rate_limit(10, per=src_ip):
          drop
      allow
  allow
```

Add one sentence to the surrounding prose at `:761-765`:

> The inner `if pkt.src_ip in [0.0.0.0/0]:` is the
> IPv4-establishing guard required by the dominator rule for the
> implicit `pkt.src_ip` read inside `rate_limit(per=src_ip)`. See
> the dominator rule earlier in this section.

The corpus item above with `expected.compiles: true` then locks
the fixed example.

## Class

Class B at the spec layer — the spec's first Tier 2 worked example
fails the spec's own dominator rule. Two implementers reading the
spec land on different accept/reject verdicts for the same program.
Spec-only fix.
