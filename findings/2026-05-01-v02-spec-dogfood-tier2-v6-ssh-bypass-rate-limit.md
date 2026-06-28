---
id: finding/2026-05-01-v02-spec-dogfood-tier2-v6-ssh-bypass-rate-limit
type: finding
protocol: [tcp]
builtins: [rate_limit]
severity: high
layer: spec
pattern_tags: [user-rule-bug, ipv6, rate-limit, tier2, dogfood, cross-family, intent-mismatch, security]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: [finding/2026-05-01-v02-spec-rate-limit-per-src-ip-with-v6-only-predicate-dead-rule]
---

# 2026-05-01-v02-spec-dogfood-tier2-v6-ssh-bypass-rate-limit

## Summary

The Tier 2 dogfood example (`FWL_V02_SPEC.md:962-998`) is the spec's
flagship "production-ready firewall" example and the basis for the
Phase 2 ≥ 48h soak (per `planning/PHASE_2_PLAN.md` Definition of
Done). It bills itself, in a code comment at `:980`, as
"Rate-limit new SSH SYNs per source." Per the spec's own runtime
rules, this comment is wrong on IPv6 traffic: every IPv6 SSH SYN
flood reaches the function's `allow` action regardless of how many
SYNs the same v6 source sends per second.

The bypass is a Class C finding (intent vs runtime mismatch) caused
by the unaddressed cross-family interaction between
`per=src_ip` (v4-only, per `:181-192`) and the v0.2 v6-parse
activation (which makes v6 TCP/22 packets reach the rate-limit call
site). The flaw is in the example program, not in any single
implementation — both the interpreter and the BPF emitter correctly
implement the spec, and they will agree that the v6 SSH SYN flood is
allowed. The operator who copies this example into production gets a
silently broken SSH brute-force protection on every v6 endpoint.

## The example, line by line

`FWL_V02_SPEC.md:962-998`:

```python
@xdp(eth0)

def firewall(pkt):
  # Trust internal and ULA address space
  if pkt.src_ip in [10.0.0.0/8, 192.168.0.0/16]:
    allow
  if pkt.src_ip6 in fc00::/7:                      # touches v6 surface
    allow

  # Geo-block at the v4 and v6 layer
  if pkt.src_ip in geoip(RU, CN, KP):
    drop
  if pkt.src_ip6 in geoip(RU, CN, KP):
    drop

  # Rate-limit new SSH SYNs per source                <-- the comment
  if pkt.proto == tcp and pkt.dst_port == 22:        # 1
    if pkt.tcp.syn and not pkt.tcp.ack:              # 2
      if rate_limit(10, per=src_ip):                 # 3 -- per=src_ip is v4-only
        drop
    count ssh_seen
    allow
  ...
```

The program touches `pkt.src_ip6` (line 7), so the v6 parse path is
activated per `:907-912`. As a consequence (per `:163-167`),
`pkt.proto == tcp` matches both v4 and v6 TCP packets, and
`pkt.dst_port` is readable on v6 frames without extension headers
(per `:139-156`). So a v6 SSH SYN packet reaches the rate-limit
call.

At the call site, `rate_limit(10, per=src_ip)` looks up the bucket
keyed by the packet's IPv4 source. The v6 packet has no IPv4 source.
Per `:181-192`:

> a rule with `per=src_ip` does not match IPv6 packets (the field
> is unreadable, so the per-bucket key is undefined and the rule
> falls through).

Tier 2 inherits the runtime semantics of the rate-limit primitive
(per `:629-637` "Both built-ins are valid inside Tier 2, but only
inside the contexts that v0.2 permits"). So the call evaluates to
**false** on v6 packets. The `if rate_limit(10, per=src_ip):` body
(`drop`) does not run.

Control falls through to `count ssh_seen` and `allow`. Every v6 SSH
SYN packet — including the millionth from a single source in one
second — is counted and allowed.

## What the operator wanted vs what the example delivers

| Source             | Comment promises (`:980`) | Actual runtime under v0.2 |
|---|---|---|
| v4 source, 1st–10th SYN to TCP/22 | rate-limited; first 10 allowed | first 10 reach `count ssh_seen`, `allow` (rate_limit pre-increment 0..9 < 10, returns false) |
| v4 source, 11th+ SYN to TCP/22    | rate-limited; dropped | dropped (rate_limit returns true) |
| **v6 source, any SYN to TCP/22**  | **rate-limited; dropped after 10** | **always reaches `count ssh_seen` and `allow`** — `per=src_ip` cannot bucket on v6, condition false, no drop ever |
| Non-SSH                          | (irrelevant) | falls through to later rules |

A v6 attacker can run SSH SYN floods against any host using this
firewall and never hit a rate limit. The operator's stated intent
("Rate-limit new SSH SYNs per source") is silently broken for IPv6.

## Why this is a finding, not just a usability nit

Three reasons:

1. **The dogfood example is the contract.** Per
   `planning/PHASE_2_PLAN.md` Definition of Done, the Phase 2 soak
   runs `dogfood_v02.fw` — this exact program — for ≥ 48h on a real
   dev VM. The soak measures *the example's intended behaviour*. If
   the example's behaviour does not match the comment, the soak is
   measuring something the operator was not promised. A v6 SYN flood
   during the soak will be silently allowed; the soak will pass; the
   operator will think v6 SSH brute-force protection works.

2. **The spec's `per=src_ip6` deferral creates a footgun**, not just
   an incomplete feature. `:1014-1015` defers `per=src_ip6` to v0.3.
   v0.2 has no spelling for "rate-limit by v6 source." Operators who
   need v6 SSH protection will copy this example, see it compile,
   ship it, and discover the gap only under attack. The deferral is
   reasonable; the lack of a *diagnostic* for the cross-family
   incoherence is not.

3. **The fix is local.** The example can be made correct by either:
   - Restricting the rate-limit branch to v4 explicitly (add a v4
     guard around the rate_limit), so the v6 path doesn't reach a
     useless rate limit and falls through visibly to a different
     branch.
   - Adding a v6-specific rate-limit branch using `per=dst_port`
     (which works on both families and is a reasonable proxy when
     `per=src_ip6` doesn't exist yet) or `per=dst_ip` (less useful
     but correct).
   - Splitting the SSH branch into v4 and v6 sub-branches with
     different rate-limit strategies.

   Each option is one or two extra lines in the example.

## Concrete `.pkt` evidence the bug is in the *example*, not the implementation

```yaml
# corpus/from_hunt/dogfood-v6-ssh-bypass.pkt
program: dogfood_v02.fw    # the spec's example, verbatim
description: |
  Send 100 v6 TCP SYNs to port 22 from the same v6 source in one
  second. Per the example's comment "Rate-limit new SSH SYNs per
  source", we expect the 11th+ SYN to be dropped. Per the spec's
  rate_limit per=src_ip rule (FWL_V02_SPEC.md:181-192), the
  bucket key is undefined for v6 packets and the rule falls
  through, so all 100 are allowed.
packets:
  - tcp6(src_ip="2001:db8::dead", dst_ip="2001:db8::cafe", dport=22,
         flags=SYN)
    repeat: 100
expected:
  - {action: allow}      # iteration 1   -- expected; first 10 allowed
  - ...
  - {action: allow}      # iteration 10  -- expected; first 10 allowed
  - {action: drop}       # iteration 11  -- comment promises drop;
                         #                  spec actually says allow
  - ...                  # the rest also allow under spec
```

If both oracles return `allow` for iterations 11..100, the
implementation is correct per the spec. The disagreement is between
the spec's *prose comment* in the example (intent: rate-limit) and
its *runtime rules* for `per=src_ip` on v6 (no bucket → fall
through).

## Three possible resolutions

| Resolution | Effect |
|---|---|
| (a) Fix the example. | Replace the SSH block in the dogfood example with a form that explicitly bifurcates v4 and v6, each with a working rate-limit (or with a comment "v6 SSH rate-limit deferred to v0.3"). Cost: 4–6 example lines. |
| (b) Add a Compile-error or warning for cross-family incoherence inside Tier 2 conditions. | Same diagnostic shape as my companion finding `rate-limit-per-src-ip-with-v6-only-predicate-dead-rule`, but lifted to Tier 2: any `if rate_limit(N, per=src_ip):` branch that is reachable on v6 packets must be guarded by a v4-establishing condition. The analyser already does dominator analysis for L3 reads; adding rate-limit family-checking is the same pass. |
| (c) Add an Edge-case note acknowledging the gap. | Document at `:980` that v0.2's `per=src_ip` does not bucket v6 packets and the comment's "per source" reading is v4-only. The operator who copies the example at least sees the warning. |

(b) is the clean answer; (a) is the quick documentation fix; (c) is
the minimum that makes the spec self-consistent. (a) and (c) are
non-exclusive.

## Proposed fix (option a — fix the example, with option c documentation)

Replace `:986-993`:

```python
  # v4 SSH brute-force: rate-limit new SYNs per source v4 IP
  if pkt.proto == tcp and pkt.dst_port == 22:
    if pkt.src_ip in [0.0.0.0/0]:                  # v4 establishment
      if pkt.tcp.syn and not pkt.tcp.ack:
        if rate_limit(10, per=src_ip):
          drop
      count ssh_seen
      allow
    # v6 SSH: per=src_ip6 deferred to v0.3 — bucket on
    # dst_port instead, which works for both families.
    if pkt.src_ip6 in [::/0]:                       # v6 establishment
      if pkt.tcp.syn and not pkt.tcp.ack:
        if rate_limit(100, per=dst_port):           # 100/sec/22 cap
          drop
      count ssh_seen
      allow
```

And amend `:980` from "Rate-limit new SSH SYNs per source" to
"Rate-limit new SSH SYNs (v4 buckets per source IP; v6 buckets per
destination port — `per=src_ip6` is a v0.3 feature)".

The dogfood corpus must include both v4 and v6 SSH SYN flood cases,
locking the rate-limit firing on each family independently.

## Class

Class C — user-rule bug at the spec layer. The example program's
runtime behaviour does not match the natural-language intent stated
in its own comment. The interpreter and BPF runtime both implement
the spec correctly; the gap is between the example's prose and the
example's rate-limit primitive. Filed at the spec layer because the
example IS the spec's contract for "what a real v0.2 program looks
like."
