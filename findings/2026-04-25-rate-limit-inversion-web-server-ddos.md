---
id: finding/2026-04-25-rate-limit-inversion-web-server-ddos
type: finding
protocol: [ipv4, tcp, udp]
builtins: [rate_limit, count]
severity: high
layer: program
pattern_tags: [rate-limit-semantics, user-intent-mismatch, spec-inconsistency]
status: fixed
source_file: fwl/examples/web_server_ddos.fw
created: 2026-04-25
pkt_path: corpus/from_hunt/2026-04-25_web_server_ddos/
related: [finding/2026-04-25-rate-limit-inversion-ssh-brute-force]
---

# 2026-04-25-rate-limit-inversion-web-server-ddos

## Summary

`fwl/examples/web_server_ddos.fw` does the OPPOSITE of what its
top-of-file comment says. The comment promises:

> Drops any source IP exceeding 1000 packets/second. Allows
> HTTP/HTTPS inbound and counts both ports separately for traffic
> dashboards.

The actual implementation drops the FIRST 1000 packets per source
IP each second (including legitimate HTTP requests, HTTP responses,
and TCP non-SYN packets) and then — once a source has crossed 1000
packets/sec — passes ANY HTTP/HTTPS packet straight through. A real
DDoS source becomes the only thing that gets allowed.

This is a recurrence of the rate_limit inversion already documented
in `finding/2026-04-25-rate-limit-inversion-ssh-brute-force`. Same
root cause (the implementation evaluates `count < threshold` for
"action fires" while every program comment in the codebase reads as
"count >= threshold"). Different victim file. Severity is higher
here than for ssh_brute_force.fw because:

1. The rule has no `if` guard — every packet from every src_ip is
   subject to the inversion, not just SSH SYNs.
2. The downstream `count http_traffic` / `count https_traffic`
   counters NEVER increment under threshold (the drop terminates
   evaluation), so the dashboard reads zero traffic until a source
   saturates — and at that point the counters start ticking for
   the very source the firewall is trying to block.
3. A flooding source crosses threshold and from then on its packets
   to ports 80/443 fall through `drop` and hit the
   `allow if pkt.dst_port in [80, 443]` rule — i.e. attackers get
   white-glove service while legitimate clients are silently dropped.

Class C user-rule bug per the hunting taxonomy.

## Hypothesis

Per spec FWL_V01_SPEC.md:288-292 the formal `rate_limit` semantic
is "action fires when count < threshold". Implemented in:

- `fwl/fwl/interpreter.py:89` — `return buckets[bucket_key] < mod.threshold`
- `fwl/fwl/emitter.py:495` — `if (cur < {mod.threshold}) { ... return ret; }`

For `drop limited by rate_limit(1000, per=src_ip)` with no `if`
clause, every packet from a given src_ip increments that src_ip's
bucket. With current semantics:

| src_ip bucket | drop fires? | net effect on packet |
|---|---|---|
| 0..999  | YES | XDP_DROP (regardless of L4) |
| 1000+   | no  | falls through; HTTP/HTTPS allow |

The natural-language comment ("drops any source exceeding 1000/sec")
maps to the inverse table. With v0.1 deployed as documented, the
firewall:

- Silently swallows the first 1000 packets per src_ip per second,
  including the TCP three-way handshake (1 SYN + 1 ACK + ... = many
  packets per session) and HTTP responses, without counting them.
- Lets every packet from a src that has exceeded 1000/sec through,
  if the destination is port 80 or 443 — the very flood the rule
  was supposed to block.

## Test cases

All six .pkt cases under `corpus/from_hunt/2026-04-25_web_server_ddos/`
PASS — both interpreter and clang-compile of the BPF agree on the
inverted behavior. BPF_PROG_RUN was unavailable in this environment
(unprivileged_bpf_disabled), so kernel runtime divergence with the
interpreter wasn't measured at runtime, but interpreter and emitter
share no code in `_rate_limit_allows` vs. `_emit_rate_limit_gate`
and both compute `count < threshold`. Class A divergence is unlikely.

| .pkt case | result | what it shows |
|---|---|---|
| `web_first_http_pkt_actually_dropped.pkt` | drop | New IP, no state, first HTTP SYN gets dropped (intent: allow) |
| `web_threshold_999_drops.pkt` | drop | bucket=999, the 1000th packet still drops (off-by-one pinned: `<` not `<=`) |
| `web_saturated_src_actually_allowed.pkt` | allow | bucket=1000 (saturated source), HTTP SYN passes through (intent: drop) |
| `web_saturated_tcp_nonsyn_anyport_allowed.pkt` | allow | Saturated source's TCP non-SYN to port 6667 (IRC) is allowed via the `allow if not pkt.tcp.syn` heuristic |
| `web_saturated_udp_default_drop.pkt` | drop | Saturated source's UDP packet falls through to default drop (matches intent — UDP isn't a target service) |
| `web_legit_http_response_dropped.pkt` | drop | A legit incoming HTTP response (TCP non-SYN, ack=true) at bucket=5 is eaten by the drop modifier before the heuristic allow rule can match |

## Root cause

Identical to `finding/2026-04-25-rate-limit-inversion-ssh-brute-force`:
the spec's formal bullets at FWL_V01_SPEC.md:288-292 state "action
fires when count < threshold," while the spec's example explanation
at FWL_V01_SPEC.md:308-314 and every user-program comment using
`rate_limit` (including this one) use the opposite reading: "action
fires when count >= threshold."

The implementation matches the formal bullets. Every example in the
spec and every example in `fwl/examples/` is wrong by the same sign
flip.

## What this rules out for this program

- Class A divergence between interpreter and BPF code generator on
  the rate_limit gate: both use the same predicate.
- Off-by-one at the threshold: `web_threshold_999_drops.pkt` confirms
  count=999 still drops (consistent strict `<`).
- Per-bucket independence: only the per=src_ip bucket gates this
  rule; each src has its own count.
- TCP/UDP/ICMP differentiation under saturation: documented in three
  separate .pkts; UDP/ICMP fall to default drop, TCP is the only path
  with a heuristic allow.
- Counter rules being skipped pre-saturation is a CONSEQUENCE of the
  inversion (drop terminates), not an independent count bug.

## What this hunt did NOT cover

- BPF_PROG_TEST_RUN at kernel level — environment lacks unprivileged
  BPF. Class A is checked by code-share inspection only, not runtime.
- 1-second window expiry — interpreter doesn't model time; the BPF
  emitter does, and runner pre-loads `time.monotonic_ns()` so the
  window stays open during the single-packet test, but multi-packet
  cross-window behavior isn't covered.
- Counter delta assertion (`counter_changes`) — runner doesn't drain
  the per-CPU counter map yet. Otherwise these cases would also
  assert "no http_traffic increment" pre-saturation and "+1
  http_traffic increment" post-saturation — a striking demonstration
  that the wrong traffic gets counted.
- Truncated-packet behavior. `truncate_to` is spec-only.

## Severity

High for the user program. Operationally deploying
`fwl/examples/web_server_ddos.fw` as-is would:

- Silently drop ~1000 packets per second per legitimate visitor.
- Keep the count_http_traffic / count_https_traffic dashboards near
  zero for legit traffic (drop fires before count).
- Bypass the drop modifier for every packet from a flooding source,
  and even count those flood packets, so dashboards show traffic
  spikes precisely from sources the rate limit was supposed to
  block.

Medium for the spec — same Class B inconsistency between formal
bullets (FWL_V01_SPEC.md:288-292) and example explanation
(FWL_V01_SPEC.md:308-314) already documented in the SSH finding.
The fix surface is the same: pick one reading, propagate to the
other location.

## Fix

Resolved together with `finding/2026-04-25-rate-limit-inversion-ssh-brute-force`.
See that finding's "Fix" section for the predicate flip in
interpreter/emitter and the spec rewrite. Same root cause; this
finding was a recurrence in a richer surface (no `if` guard, two
count rules, fail-open via `allow if dst_port in [80,443]`).

Post-fix verification: `web_server_ddos.fw` now drops the (1001)-th
and subsequent packets per src_ip per second — the documented DDoS
defense behavior. Counter rules ride on the rate-limited drop's
fall-through path so dashboards count legitimate traffic instead of
flood spikes from saturated sources.
