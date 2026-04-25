---
id: miss/2026-04-25-internal-network-program
type: miss
protocol: [ipv4, tcp, udp, icmp]
builtins: [log]
severity: n/a
layer: program
pattern_tags: [cidr-list, log-action, rule-ordering, proto-guard]
status: open
source_file: fwl/examples/internal_network.fw
created: 2026-04-25
---

# 2026-04-25-internal-network-program

## Summary

Hunted bugs in `fwl/examples/internal_network.fw` and the FWL toolchain
that compiles/interprets it. No bugs found across 14 targeted .pkt
test cases plus static inspection of the emitted BPF C and the
analyzer's guard rules. The interpreter agreed with all natural-language
expectations of the program. BPF_PROG_RUN was unavailable in the
environment (unprivileged_bpf_disabled), so class A
(interpreter↔BPF runtime) divergence could not be tested at runtime;
clang compilation of the emitted C succeeded for every case.

## Program under test

```
@xdp(eth0)

allow if pkt.src_ip in [10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16]
allow if pkt.proto == tcp and pkt.dst_port in [80, 443, 22]
allow if pkt.proto == udp and pkt.dst_port == 53
log if pkt.proto == tcp and pkt.tcp.syn and not pkt.tcp.ack
default drop
```

## Hypotheses tested

| # | Hypothesis | Result |
|---|---|---|
| 1 | External TCP SYN to non-allowed port → default drop | confirmed (no bug) |
| 2 | External SYN+ACK → log rule should NOT fire (`not pkt.tcp.ack`) | confirmed |
| 3 | External TCP SYN to port 22 → rule 1 allow, log skipped | confirmed |
| 4 | External UDP to port 53 → rule 2 allow | confirmed |
| 5 | External UDP to other port → default drop | confirmed |
| 6 | External ICMP → default drop | confirmed |
| 7 | Internal ICMP from 10.x.x.x → rule 0 allow | confirmed |
| 8 | 10.0.0.0 (CIDR /8 lower boundary) → allow | confirmed |
| 9 | 10.255.255.255 (CIDR /8 upper boundary) → allow | confirmed |
| 10 | 172.16.0.0 (CIDR /12 lower boundary) → allow | confirmed |
| 11 | 172.31.255.255 (CIDR /12 upper boundary) → allow | confirmed |
| 12 | 172.32.0.0 (just outside /12) → drop | confirmed |
| 13 | 192.168.255.255 (/16 upper boundary) → allow | confirmed |
| 14 | 192.169.0.0 (just outside /16) → drop | confirmed |

## Compiler-surface checks (static)

- **Generated BPF C inspected**: CIDR masks 0xFF000000, 0xFFF00000,
  0xFFFF0000 are byte-correct for /8, /12, /16. Prefix values
  0x0A000000, 0xAC100000, 0xC0A80000 match RFC1918 in host byte order
  (post-`bpf_ntohl`). Port comparisons are bare `==` after the proto
  guard short-circuits. Log emission writes `rule_index = 3` matching
  the rule's zero-based position.

- **Analyzer guard rules**: rejected `log if pkt.tcp.syn and not
  pkt.tcp.ack` (no proto guard); rejected `not (pkt.proto == tcp) and
  pkt.tcp.syn` (NotOp discards the constraint per
  `analyzer.py:269-274`); accepted `(pkt.proto == tcp and pkt.tcp.syn)
  or pkt.proto == udp` (per-branch guards union into {TCP,UDP}).

- **Boolean composition**: the log rule's flat 3-operand `AndOp`
  (`pkt.proto == tcp and pkt.tcp.syn and not pkt.tcp.ack`) maps
  directly to `(proto == IPPROTO_TCP) && tcp_syn && !(tcp_ack)` in C.
  Both interpreter and emitter short-circuit left-to-right.

## What this rules out for this program

- Off-by-one in any of the three RFC1918 CIDR boundaries.
- Mis-ordering of allow/log/default that would silently allow or drop
  the wrong traffic class.
- Analyzer accepting unguarded `pkt.tcp.syn`/`ack` access in any
  rewrite of the log condition.
- Type mismatch between `tcp_syn`/`tcp_ack` (bool semantics) and the
  rule's truthy/falsy use in C.
- The "Visibility into unexpected SYNs from outside" comment matching
  the rule's actual semantics: the log rule fires precisely on TCP
  SYN-only packets that would otherwise hit `default drop` (ordering
  ensures internal sources and known-service ports are excluded).

## What this hunt did NOT cover

- Class A runtime divergence between BPF_PROG_RUN and the AST
  interpreter — environment lacked unprivileged BPF; only static
  inspection was possible.
- Counter or rate_limit interaction (program uses neither).
- Truncated-packet bounds-check fall-through paths (`truncate_to`
  is spec-only in v0.1; loader does not yet implement it per
  PKT_V01_SPEC.md:81).
- Log event payload assertions (`log_events` is spec-only; runner does
  not drain the ringbuf yet).

## Corpus added

14 .pkt cases under
`corpus/from_hunt/internal_network_*.pkt`. All pass against the
spec/interpreter oracles; the bpf oracle skipped (privilege).
