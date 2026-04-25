---
id: finding/2026-04-25-rate-limit-spec-bullets-vs-example-still-inconsistent
type: finding
protocol: [tcp, udp, icmp]
builtins: [rate_limit]
severity: medium
layer: spec
pattern_tags: [rate-limit-semantics, spec-inconsistency, off-by-one]
status: fixed
source_file: docs/FWL_V01_SPEC.md
created: 2026-04-25
pkt_path: corpus/from_hunt/2026-04-25_rate_limit_off_by_one/
related: [finding/2026-04-25-rate-limit-inversion-ssh-brute-force, finding/2026-04-25-rate-limit-inversion-web-server-ddos]
---

# 2026-04-25-rate-limit-spec-bullets-vs-example-still-inconsistent

## Summary

The `rate_limit` semantics section of `docs/FWL_V01_SPEC.md` still
contradicts itself after the prior fix. The earlier finding
(`finding/2026-04-25-rate-limit-inversion-ssh-brute-force`) said
the formal bullets had been rewritten to match the example, but in
practice the two readings remain off-by-one:

- **Bullets** (FWL_V01_SPEC.md:288-292):
  > "the action only fires once the bucket has accumulated `<N>`
  > matches"
  > "If the post-increment count is AT OR ABOVE the threshold:
  > the action applies"

  → For threshold N=3, post-increment 3 ≥ 3 fires on the **3rd**
  matching packet.

- **Example** (FWL_V01_SPEC.md:294):
  > "the first `N` matching packets per bucket per second are NOT
  > dropped by this rule; the (N+1)-th and beyond ARE dropped."

  → For threshold N=3, fires on the **4th** matching packet.

These two statements describe different behaviors. The
implementation matches the example (`cur >= threshold` where `cur`
is the **pre**-increment count, `fwl/fwl/interpreter.py:91` and
`fwl/fwl/emitter.py:476,503`), so the bullets are still wrong.

Class B (spec inconsistency).

## Hypothesis

For a rule `drop if pkt.proto == tcp and pkt.dst_port == 22 limited
by rate_limit(10, per=src_ip)` and a pre-test bucket count of 9,
the test packet is the 10th matching packet for that source IP.

| Reading | Behavior on the 10th packet |
|---|---|
| Bullets at line 288-292 | post-increment count = 10, 10 ≥ 10 → action applies → drop |
| Example at line 294 | first 10 NOT dropped, 11th onwards dropped → 10th allowed |
| Implementation | `cur=9 >= threshold=10` → false → don't fire → allow |

Implementation matches the example. The bullets are inconsistent
with both.

## Test case

`corpus/from_hunt/2026-04-25_rate_limit_off_by_one/nth_packet_per_spec_bullets_fires_drop.pkt`
encodes the bullets' reading: `expected.bpf_action: drop`. It
**fails**, with both interpreter and BPF reporting `XDP_PASS`. The
failure is the bug — it pins the spec inconsistency.

```yaml
state:
  rate_limit:
    0:
      "9.9.9.9": 9
test_packet:
  builder: tcp(src_ip="9.9.9.9", dst_port=22)
expected:
  bpf_action: drop   # per bullets — actual: allow
```

## Root cause

The fix recorded under
`finding/2026-04-25-rate-limit-inversion-ssh-brute-force` rewrote
the spec bullets to "the action fires once the bucket has
accumulated <N> matches" but kept the example's wording at
line 294. "Accumulated N matches" naturally reads as
post-increment count = N (i.e., the Nth packet), but the example
explicitly says the (N+1)-th packet is the first dropped.

The implementation's predicate `cur >= threshold` (pre-increment
count) means the first packet that fires has pre=N, post=N+1 —
i.e., the (N+1)-th packet. So the impl matches "first N not
dropped, (N+1)-th onwards dropped" exactly.

To make all three (bullets, example, impl) consistent, one of the
following fixes is required:

| Path | Change |
|---|---|
| Keep impl, fix bullets (recommended) | Lines 288-292: "the action fires once the bucket has accumulated MORE than N matches" / "If the **pre**-increment count is BELOW the threshold: the rule does not apply" / "If the pre-increment count is AT OR ABOVE the threshold: the action applies" |
| Keep bullets, change impl + example | Switch impl to `post >= threshold` (i.e., `cur + 1 >= threshold` or equivalently `cur >= threshold - 1`), and update line 294 to "first (N-1) not dropped; the Nth and beyond ARE dropped". This is breaking semantics for every example program. |

## What this rules out

- Class A interpreter↔BPF divergence on the gate. Both compute the
  same `cur >= threshold` predicate from `pre-increment count`.
- Off-by-one in either oracle alone. They agree.
- Spec inconsistency in the *example explanation* — that part is
  internally consistent ("first N not dropped, (N+1)-th onwards
  dropped" matches the impl).

## What this hunt did NOT cover

- BPF_PROG_TEST_RUN at kernel level — the environment lacks
  unprivileged BPF, so the divergence vs the bullets reading is
  observable only via the AST interpreter here. The clang compile
  passes, so the BPF C is at least syntactically well-formed for
  the impl semantics.
- Window expiry. Both oracles only look at pre-state count; the
  one-second sliding-window reset is BPF-only. Cross-window
  off-by-one isn't testable until the runner can advance time.

## Severity

Medium for the spec — the example readings, every program comment
in the codebase, and the impl are aligned with each other. The
formal bullets at lines 288-292 are the only outlier. A reader
who consults the bullets as the authoritative section will
implement a future port off-by-one.

Low for the impl — implementation is internally consistent and
matches both the example explanation and operator intent
("drop traffic that exceeds N/sec").

## Fix

Patch `docs/FWL_V01_SPEC.md` lines 288-292 to read pre-increment
semantics. Suggested wording:

```
Every matching packet increments the bucket counter, but the
action only fires once the bucket has accumulated MORE than
<N> matches.

- If a packet matches the rule's condition: the rate counter for
  its bucket is incremented.
- If the pre-increment count is BELOW the threshold: the rule
  does not apply this packet. Evaluation continues to the next
  rule.
- If the pre-increment count is AT OR ABOVE the threshold: the
  action applies (the rule is "active") for this packet.
```

This makes the bullets, the example, and the implementation all
say "first N matches per bucket are not affected; the (N+1)-th
matches and beyond fire the action."

## Resolution

Applied the recommended bullet rewrite at `docs/FWL_V01_SPEC.md:288-292`:
the bullets now say **pre-increment count >= threshold** and the
prose explicitly notes "the action fires once the bucket already
holds N or more matches before this packet". Bullets, example, and
impl now align: first N not dropped, (N+1)-th onwards dropped.
