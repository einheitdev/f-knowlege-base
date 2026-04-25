---
id: finding/2026-04-25-rate-limit-inversion-ssh-brute-force
type: finding
protocol: [ipv4, tcp]
builtins: [rate_limit, count]
severity: high
layer: program
pattern_tags: [rate-limit-semantics, user-intent-mismatch, spec-inconsistency]
status: fixed
source_file: fwl/examples/ssh_brute_force.fw
created: 2026-04-25
pkt_path: corpus/from_hunt/2026-04-25_ssh_brute_force/
---

# 2026-04-25-rate-limit-inversion-ssh-brute-force

## Summary

`fwl/examples/ssh_brute_force.fw` does the OPPOSITE of what its
comment says. The comment promises "Each source IP gets up to 3 new
SSH connection attempts per second. Beyond that, additional SYNs are
dropped." The actual implementation drops the FIRST 3 SYNs from each
source IP and allows the 4th onwards.

Class C user-rule bug per the hunting taxonomy. Also surfaces a
Class B inconsistency in `FWL_V01_SPEC.md` itself: the spec's formal
`rate_limit` semantic (action fires when `count < threshold`) directly
contradicts its own example explanation ("this drop rule only fires
for the 11th, 12th, 13th... packet").

## Hypothesis

Per spec FWL_V01_SPEC.md:288-292 the formal `rate_limit` semantics
are:

> If a packet matches the rule's condition AND the current rate
> for that bucket is below the threshold: the action applies (the
> rule is "active") and the rate counter for that bucket is
> incremented.

This is implemented in:

- `fwl/fwl/interpreter.py:89` — `return buckets[bucket_key] < mod.threshold`
- `fwl/fwl/emitter.py:495` — `if (cur < {mod.threshold}) { ... return ret; }`

For the SSH brute force rule `drop ... limited by rate_limit(3, per=src_ip)`:

| SYN # | bucket count | `count < 3` | drop fires? | comment intent |
|-------|--------------|-------------|-------------|----------------|
| 1     | 0            | true        | YES (drop)  | allow          |
| 2     | 1            | true        | YES (drop)  | allow          |
| 3     | 2            | true        | YES (drop)  | allow          |
| 4     | 3            | false       | no (fall through to allow) | drop |
| 5     | 3            | false       | no (allow)  | drop           |

Implementation behavior is the EXACT INVERSE of what the program's
natural-language comment promises.

## Test cases

All six .pkt cases under `corpus/from_hunt/2026-04-25_ssh_brute_force/`
PASS — the interpreter and BPF clang-compile both confirm the
implementation behaves as documented in the table above. BPF runtime
oracle was unavailable in this environment (unprivileged_bpf_disabled),
but interpreter and emitter share no code in `_rate_limit_allows`
vs. `_emit_rate_limit_gate` and both compute the same `count <
threshold` predicate, so a Class A divergence is unlikely.

| .pkt case | result | what it shows |
|---|---|---|
| `ssh_first_syn_actually_dropped.pkt` | drop | New IP, no state, 1st SYN gets dropped (intent: allow) |
| `ssh_3rd_syn_dropped.pkt` | drop | count=2, 3rd SYN gets dropped (intent: allow) |
| `ssh_fourth_syn_actually_allowed.pkt` | allow | count=3, 4th SYN allowed (intent: drop) |
| `ssh_established_unaffected.pkt` | allow | TCP without SYN → allow (matches intent) |
| `ssh_synack_handshake_allowed.pkt` | allow | SYN+ACK → allow (matches intent; `not pkt.tcp.ack` excludes) |
| `ssh_independent_buckets_per_src.pkt` | drop | Buckets are per-`src_ip`; one IP's saturation doesn't help another |

`ssh_first_syn_intent_allow.pkt` (kept out of the corpus) was the
mirror: same packet but `expected: allow` per the comment's intent.
It FAILS — both oracles produce drop. That failure is the bug.

## Root cause

Two-part:

1. **Spec inconsistency (Class B in `FWL_V01_SPEC.md`)**.
   FWL_V01_SPEC.md:288-292 (formal bullets) says the action fires
   when `count < threshold`; FWL_V01_SPEC.md:308-314 (example
   explanation) says the rule "only fires for the 11th, 12th,
   13th... packet" — i.e., when `count >= threshold`. These cannot
   both be true. The implementation followed the bullets; every
   example in the spec (and the user-program comment, which is a
   verbatim copy of the example's intent) reads the rule as the
   opposite.

2. **User-program comment (Class C in `ssh_brute_force.fw`)**.
   The comment block at the top of `ssh_brute_force.fw` and the
   inline comment "Drop new SSH connections beyond 3 per second per
   source IP" both state the rate-exceeds-threshold reading. With
   the v0.1 implementation, the program drops the first 3 SYNs and
   then opens the door — which is, for SSH brute force, dangerously
   wrong: a slow attacker doing 3 SYNs/sec gets every attempt past
   the third through.

The clean fix for the spec: change the formal semantics to "action
fires when `count >= threshold`" and update the impl predicate to
`>= threshold`. Then the example explanations and every "drop ...
limited by" usage in the spec become consistent. (Alternatively
flip the example's prose, but that makes `rate_limit` semantically
useless as a drop modifier — you'd be promising to drop the first
N and then giving up.)

## What this rules out

- Off-by-one at the threshold boundary: the impl is consistently
  `count < threshold`; both oracles agree on this strict-less-than.
- Rate-bucket key encoding mismatch: covered by the existing
  `corpus/05_rate_limit/` cases plus the new `..._independent_buckets_per_src.pkt`.
- TCP flag handling for established traffic: SYN-less and SYN+ACK
  packets correctly bypass the drop rule (matches intent).
- Multi-rule fall-through ordering: when drop fires it returns
  immediately; when it doesn't, count + allow run in order.

## What was NOT tested

- BPF_PROG_TEST_RUN (kernel runtime) — environment lacked
  unprivileged BPF. Class A divergence between interpreter and BPF
  runtime is unaddressed at kernel level here. The clang compile
  of every emitted program succeeded.
- Counter delta assertion (`counter_changes`) — runner does not
  drain the per-CPU counter map yet (PKT_V01_SPEC.md:144).
  Otherwise these cases would also assert that `count ssh_allowed`
  fires only when the drop rule falls through.
- 1-second window expiry — interpreter does not model the time
  axis. The BPF emitter does, but with `time.monotonic_ns()` from
  `runner._encode_rl_value_per_cpu` the pre-loaded timestamp is
  fresh enough that the window stays open during BPF_PROG_TEST_RUN.

## Severity

High for the user program — production deployment of this example
would silently mis-implement the brute-force defense.

Medium for the spec — the example explanations are wrong, but the
impl's behavior is unambiguously documented in the formal bullets.
A reader who only consults the bullets reaches the right (impl-
matching) understanding.

## Fix

Resolved by flipping the rate_limit firing predicate from
`count < threshold` to `count >= threshold` and aligning the spec
formal bullets with the example explanation:

- `fwl/fwl/interpreter.py:91,95,96` — predicate flipped.
- `fwl/fwl/emitter.py:455-499` — both `_emit_rate_limit_side_effect`
  and `_emit_rate_limit_gate` now increment the bucket on every
  matching packet and fire the action when the pre-increment count
  has reached the threshold.
- `docs/FWL_V01_SPEC.md:288-298` — formal bullets rewritten so that
  "the action fires once the bucket has accumulated N matches", which
  is the same reading every example program comment uses.

After the fix, `drop ... limited by rate_limit(N)` drops the
(N+1)-th matching packet onward — i.e., it behaves the way the
program comment promises. The original `from_hunt` .pkt cases that
demonstrated the inversion have been renamed `*_post_fix.pkt` and
their `expected.bpf_action` values flipped; they remain in the
regression corpus as anti-regression tests against re-introducing
the predicate flip.
