---
id: pattern/user-rule-intent-mismatch
type: pattern
protocol: [tcp, udp, icmp6]
builtins: [rate_limit]
severity: high
layer: [program, spec, user-rule]
pattern_tags: [user-intent-mismatch, rate-limit-semantics, cross-family, ipv6, ndp, dogfood, security, comment-vs-runtime]
status: hypothesis
created: 2026-05-01
instance_count: 4
instances:
  - finding/2026-04-25-rate-limit-inversion-ssh-brute-force
  - finding/2026-04-25-rate-limit-inversion-web-server-ddos
  - finding/2026-05-01-v02-spec-dogfood-tier2-v6-ssh-bypass-rate-limit
  - finding/2026-05-01-v6-internal-fw-ndp-dad-and-rs-dropped-against-stated-intent
---

# user-rule-intent-mismatch

## Description

The natural-language comment of a sample / dogfood / example FWL
program promises a behaviour the runtime — under the spec's *correct*
semantics — does not deliver. The interpreter and BPF runtime agree
with each other (Class A is not the issue), but both disagree with
the comment's stated intent. This is Class C in the hunting taxonomy
("user-rule bug"), but the bugs land in *spec / shipped-example*
files, not third-party operator programs:

- **Rate-limit polarity inversion.** `ssh_brute_force.fw` and
  `web_server_ddos.fw` v0.1 examples drop the *first* N matching
  packets per bucket and then allow flood traffic past the
  threshold — the exact opposite of the comment. Same root cause
  in two examples.
- **Cross-family rate-limit gap.** The v0.2 dogfood example uses
  `rate_limit(per=src_ip)` (v4-only bucket) inside a `pkt.proto ==
  tcp` guard that is cross-family in v6-active programs. v6 SSH
  SYN floods reach the rate-limit call site, the v4 bucket key is
  undefined, the call evaluates to false, every v6 SSH packet
  reaches `allow`. The comment says "Rate-limit new SSH SYNs per
  source"; the runtime says "v4 yes, v6 always allow."
- **NDP control-plane drops.** `v6_internal.fw` rule says "IPv6
  neighbour discovery / link-local control plane stays open" but
  filters on `pkt.src_ip6 in fe80::/10`. RFC-mandated DAD
  Neighbor Solicitations and pre-LL Router Solicitations use `::`
  as source, not link-local. Two of seven NDP message types
  silently drop; DAD broken; address-conflict detection invisible.

The unifying root cause is that the natural-language comment
describes a **policy intent** (rate-limit floods, allow control
plane) while the implementation describes a **mechanism** (drop the
first N matching packets, filter on link-local source). Operators
read the comment, copy the example, ship a firewall whose
mechanism doesn't realise the policy. The bugs survive review
because every existing `.pkt` test either confirms the mechanism
matches the implementation (it does — both oracles agree) or fails
to exercise the corner the policy promised to cover (DAD packets,
v6 SSH SYNs, the (N+1)-th matching packet).

The hunt mechanism that surfaces these is **adversarial
intent-driven testing**: read the program's English-language
intent, write a packet that exercises the intent (not the
mechanism), confirm both oracles produce the wrong action.

## Check Strategy

1. **For every shipped `.fw` example, lift each natural-language
   comment into a packet hypothesis.** "Drop SSH brute force" →
   send 100 SYNs to port 22 from one source, confirm the 11th and
   beyond drop. "Allow control plane" → send each RFC-mandated
   NDP message type (DAD NS, RS pre-LL, NA, RA, MLD, etc.),
   confirm each is allowed. Cases where the runtime disagrees
   with the comment are findings.
2. **For every `rate_limit(per=<field>)` site, identify packet
   shapes that reach the call but have an undefined bucket key.**
   v4-only `per=` field on a v6-reachable site, v6-only `per=`
   field on a v4-reachable site (when `per=src_ip6` lands in
   v0.3), L4 `per=` field on an ICMP packet. Confirm the rule
   evaluates the way the comment expects (rarely — these are
   typically dead/silently-bypassed paths).
3. **Audit examples for "policy" vs "mechanism" comment style.**
   Comments that describe what the rule *should* accomplish
   (rather than how) are high-risk; the implementer typed the
   how, the comment described the what. Cross-check by writing
   the policy's adversarial input.
4. **Run RFC-conformance tests against any `.fw` example whose
   comment mentions a protocol's "control plane" or "discovery"
   or "negotiation"**. RFC 4861/4862/5722 NDP messages, ARP DAD,
   ICMP Echo Replies — each protocol's control plane has corner
   cases (unspecified address, all-zeros source, etc.) that
   policy-level comments rarely enumerate.
5. **Audit the dogfood example under v0.2 in particular.** It is
   the spec's flagship "production-ready firewall" and the
   regression target for the ≥48h soak. Its comments are
   policy-level; the soak measures whether the mechanism delivers
   the policy. Currently the dogfood example has at least one
   confirmed instance of this pattern.

## Known Instances

- `finding/2026-04-25-rate-limit-inversion-ssh-brute-force` —
  comment "drops new SSH connections beyond 3/sec" but mechanism
  drops the first 3 SYNs and allows the 4th onwards.
- `finding/2026-04-25-rate-limit-inversion-web-server-ddos` —
  comment "drops sources exceeding 1000/sec" but mechanism allows
  the source past threshold and counts the flood traffic.
- `finding/2026-05-01-v02-spec-dogfood-tier2-v6-ssh-bypass-rate-limit`
  — comment "Rate-limit new SSH SYNs per source"; v6 SSH SYNs
  bypass the v4-only `per=src_ip` bucket entirely.
- `finding/2026-05-01-v6-internal-fw-ndp-dad-and-rs-dropped-against-stated-intent`
  — comment "IPv6 neighbour discovery / link-local control plane
  stays open"; DAD-NS and pre-LL RS use `::` source, dropped.

## Where to Look Next

- **Every `.fw` example in `fwl/examples/`.** v0.1 examples
  (`internal_network.fw`, `ssh_brute_force.fw`,
  `web_server_ddos.fw`) have been hunted; the v0.2 examples
  (`v6_internal.fw`, `geoip_block.fw`, `dogfood_v02.fw`) have one
  finding each so far. Each has multiple comments worth a
  policy-level intent test:
  - `geoip_block.fw` — does the geoip drop catch every CIDR in
    the country's prefix list? Is the manifest ordered correctly
    so a packet whose v4 source is in a "geo-block" prefix
    actually gets blocked?
  - `v6_internal.fw` — beyond the NDP-DAD finding, do the other
    NDP messages (Redirect, Inverse Neighbor Solicitation,
    Multicast Router Solicitation/Advertisement) traverse?
  - `dogfood_v02.fw` — beyond the v6-bypass finding, does the
    geoip-block branch also have a v4/v6 cross-family asymmetry?
- **`per=src_port`/`per=dst_port` rate-limits on ICMP/ICMPv6.**
  These have no L4 ports. The bucket key is undefined; the
  comment likely says "rate-limit per port". Same shape as the
  v6 bypass finding.
- **`per=src_ip` rate-limits on a path that includes UDP fragment
  reassembly or v6 extension headers.** v0.2 doesn't read past v6
  ext-headers in the standard parse path; an "anti-flood" rule
  that touches L4 may silently miss fragments / ext-header
  packets — the policy says "all UDP", the mechanism says "first
  fragment, no ext headers".
- **Geoip "block country X" intent tests.** Test packets from
  every prefix in the country's actual current CIDR list (from
  RIPE/MaxMind/etc.). The intent is "block country X"; the
  mechanism reads from a bundled prefix list that may be stale.
  This is a Class C in the *daemon* path, not the compiler — the
  daemon's geoip refresh discipline is unhunted.
- **Multi-rule fall-through intent.** Examples like "allow
  established traffic, drop everything else" rely on rule
  ordering and on the parser's interpretation of "established".
  Comments may use loose language ("established") that the
  mechanism implements as a heuristic (`not pkt.tcp.syn` plus
  `pkt.tcp.ack`). The TCP RST/FIN case is typically not
  enumerated in the comment but is in scope per the policy.
- **Cross-pattern: spec worked-examples whose comments lie.** The
  rate-limit-inversion finding bled into the v0.1 spec's example
  prose. Audit every spec narrative paragraph that says
  "intuitively, this rule does X" — if the example program does
  not do X under the formal rules, that's an
  example-contradicts-rule finding *and* a user-rule-intent
  finding (cross-pattern).
