---
id: miss/2026-05-01-v6-internal-stateless-synack-drop-is-by-design
type: miss
protocol: [tcp]
builtins: []
severity: n/a
layer: user-rule
pattern_tags: [stateless-firewall, conntrack, synack, tcp-established, by-design]
status: ruled-out
source_file: fwl/examples/v6_internal.fw
created: 2026-05-01
---

# 2026-05-01-v6-internal-stateless-synack-drop-is-by-design

## Hypothesis

`v6_internal.fw` would drop the legitimate SYN-ACK reply that the
outside world sends back when an internal host opens an outbound
connection (e.g. internal `10.0.0.5` connects to `93.184.216.34:443`,
the reply is `93.184.216.34:443 → 10.0.0.5:<ephemeral>` with
`syn=true, ack=true`). The dst port is ephemeral (not in `[80, 443,
22]`), so rule 2 doesn't match; the source is external so rule 0
doesn't match; rule 5 only matches pure SYN. Default drop fires.

Test case
`/tmp/hunt/v6_internal_v4_synack_drop.pkt` confirmed both oracles
return `drop`. Naïvely this looks like the firewall is breaking
established outbound TCP — a Tier C user-rule bug.

## Why it's not a bug

This is a **stateless** XDP firewall. There is no conntrack
primitive in v0.1 or v0.2 (deferred per
`FWL_V01_SPEC.md:528`'s "What Is Not in v0.1" list, and v0.2 doesn't
add it either). The policy is exactly what the rules say: nothing
inbound except (i) sources in RFC 1918 / RFC 4193 ranges,
(ii) services on the standard ports, (iii) link-local NDP. Any
host that needs to talk *outbound* and accept replies through this
firewall must also live behind upstream NAT/conntrack that handles
the established traffic before it arrives at the XDP program — or
the operator must add explicit rules.

The v0.1 sibling `internal_network.fw` has the same property and
ships unchanged. The v0.2 spec preserves it: the `web_server_ddos.fw`
example uses an explicit `allow if pkt.proto == tcp and not
pkt.tcp.syn` heuristic for "established" because there's nothing
better; v6_internal.fw deliberately does not.

The natural-language intent — "Internal-network policy" — is
*server-side* policy: control what arrives unsolicited from outside.
SYN-ACKs from outside are by definition replies to outbound
connections that this XDP firewall didn't authorise (because it
doesn't *see* outbound). A real deployment puts conntrack upstream;
v6_internal.fw is the leaf-layer policy below that.

## Decision

This is intentional. The corpus case
`v6_internal_v4_synack_drop.pkt` is retained as a regression
anchor — if a future v0.3 conntrack lands and silently flips this
to allow, the test will fail and surface the change.

## Notes for next time

When evaluating user-rule bugs against natural-language intent,
distinguish "the rule does something the comment doesn't say"
(real bug — see the DAD finding 2026-05-01) from "the rule does
exactly what the comment says, but the rule's class of policy is
incomplete on its own" (this miss). The second is a
documentation / deployment concern, not a firewall bug.
