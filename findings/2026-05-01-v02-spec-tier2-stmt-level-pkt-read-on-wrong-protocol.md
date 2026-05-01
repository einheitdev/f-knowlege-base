---
id: finding/2026-05-01-v02-spec-tier2-stmt-level-pkt-read-on-wrong-protocol
type: finding
protocol: [tcp, udp, icmp, icmp6]
builtins: []
severity: high
layer: spec
pattern_tags: [spec-ambiguity, tier2, protocol-guards, undefined-behavior]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: []
---

# 2026-05-01-v02-spec-tier2-stmt-level-pkt-read-on-wrong-protocol

## Summary

`FWL_V02_SPEC.md` documents the v0.1 short-circuit / protocol-guard
rule for *conditions* inside Tier 2, but says nothing about
*statement-level* reads of protocol-specific `pkt` fields (e.g.
`x = pkt.dst_port` on a non-TCP/UDP packet, `x = pkt.tcp.syn` on
ICMP, `x = pkt.src_ip6` on an IPv4 packet). Tier 2 introduces the
first context where `pkt.<L4 field>` can be evaluated outside an
`if` condition; the spec leaves that path undefined.

Class A semantically (interpreter and BPF runtime can land on
different answers because the spec gives them no rule), file as
spec-layer.

## The hole

### What the spec does cover

- `FWL_V02_SPEC.md:574-578`:

  > *Short-circuit and protocol guards.* All v0.1 short-circuit and
  > protocol-guard rules apply unchanged inside Tier 2 conditions.
  > A Tier 2 `if pkt.proto == tcp and pkt.dst_port == 22:` reads
  > `pkt.dst_port` only when the proto check passes, exactly as in
  > Tier 1.

This explicitly scopes the guarantee to "Tier 2 conditions". The
v0.1 rule it inherits is itself rule-scoped (`pkt.<L4>` inside a
*rule predicate* falls through when the L4 isn't readable).

### What it does not cover

`FWL_V02_SPEC.md:485-495` lists the statement forms:

- `if <cond>:`
- `<local> = <expr>` — local-variable assignment
- `allow`, `drop`, `log`, `count <n>`

Local assignments admit any `expression`, and the example in
`FWL_V02_SPEC.md:735-744` does exactly:

```python
def firewall(pkt):
  internal = pkt.src_ip in [10.0.0.0/8, 192.168.0.0/16]
  ...
```

`pkt.src_ip` is L3 and always readable on IPv4 packets, but the
example sets the precedent that `<local> = <pkt expr>` is
fine. By the same surface, the user can write:

```python
def firewall(pkt):
  port = pkt.dst_port      # L4 read, no proto guard
  if port == 22:
    drop
  drop
```

For an ICMP packet (or an IPv4 fragment, or a v6+HBH packet) `port`
is undefined per v0.1 fall-through semantics — but Tier 2 is not a
"rule" and there is no rule to "fall through". Possible
interpretations:

| Reading | Answer |
|---|---|
| (a) Statement read of an unreadable field is **undefined**: BPF emits whatever raw memory is at the offset; analyser may or may not warn | unsafe, classic spec hole |
| (b) The Tier 2 function aborts (verifier-style) and the program returns the implicit `allow` | needs a defined exit path |
| (c) The local is set to a defined sentinel (zero, max, …) | needs a defined sentinel value |
| (d) The analyser **rejects** any L4 field read in a position not dominated by a proto guard | strictest, easiest to verify |

The spec picks none of these. The interpreter and emitter — written
by independent reviewers — can land on (a) and (b) without either
violating the prose.

### Worse: the dogfood example uses this surface

`FWL_V02_SPEC.md:902-935` (Tier 2 dogfood example) does **not**
have any unguarded L4 reads, but the spec offers no constraint that
prevents an author from writing one. The corpus then either:

- Locks behaviour to whatever interp/BPF agree on by accident (and
  freezes it via a regression case nobody can change without
  knowing this is "spec'd by accident"), or
- Disagrees between oracles when the agent fuzzes a Tier 2 program
  with an unguarded L4 read — the bug shows up as a
  cross-oracle drift, but the *root cause* is the spec hole.

## Proposed fix

Pick one of:

- **(d) is cleanest:** add a Tier 2 type rule:

  > A `pkt.<L4 field>` (`pkt.src_port`, `pkt.dst_port`, `pkt.tcp.*`)
  > read in a position not dominated by a `pkt.proto == tcp|udp|icmp6`
  > guard is a compile error: `error: '<field>' read on a path not
  > guarded by a 'pkt.proto' check`.

  Symmetric rule for `pkt.src_ip6`/`pkt.dst_ip6`: must be guarded
  by `pkt.src_ip6` / IPv6-touching guard, or accept the v0.2
  fall-through wording. (This is essentially the dominator-style
  check the analyser already needs to do for the
  `local-read-before-assignment` rule, see
  `FWL_V02_SPEC.md:606-609`.)

- **(b) or (c):** add a "Statement-level pkt access on unreadable
  field" subsection to "Semantics" describing the fall-through (the
  function returns implicit `allow`, or the local takes a defined
  zero value).

Either is fine; "no rule at all" is the bug.
