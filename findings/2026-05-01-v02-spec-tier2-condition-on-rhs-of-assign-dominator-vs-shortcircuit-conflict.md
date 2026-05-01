---
id: finding/2026-05-01-v02-spec-tier2-condition-on-rhs-of-assign-dominator-vs-shortcircuit-conflict
type: finding
protocol: [tcp, udp, icmp, icmp6]
builtins: []
severity: high
layer: spec
pattern_tags: [spec-ambiguity, spec-inconsistency, tier2, dominator, statement-position, short-circuit, example-violation]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: [finding/2026-05-01-v02-spec-tier2-stmt-level-pkt-read-on-wrong-protocol, finding/2026-05-01-v02-spec-tier2-stmt-level-ipv6-l3-read-undefined-value, finding/2026-05-01-v02-spec-tier2-dominator-rule-negation-and-else-not-addressed]
---

# 2026-05-01-v02-spec-tier2-condition-on-rhs-of-assign-dominator-vs-shortcircuit-conflict

## Summary

The v0.2 spec resolves the prior "statement-level `pkt` read on the
wrong protocol" hole by adding a dominator rule (`:611-633`) that
classifies reads of protocol-specific `pkt` fields *in statement
position* as compile errors when no preceding `if` has established
the relevant guard. The fix added a carve-out at `:635`:

> Reads of `pkt.src_ip`/`pkt.dst_ip` *inside* a `condition` (a
> comparison or `in` expression) remain governed by v0.1's
> short-circuit semantics — reaching such a comparison reads the
> field only when the EtherType matches, and the comparison
> evaluates to false otherwise. **The Tier 2 dominator rule applies
> only to statement-position reads (the RHS of an assignment).**

The sentence is internally contradictory: the carve-out exempts
"reads inside a condition", and the closing sentence pins the
dominator rule to "statement-position reads (the RHS of an
assignment)". The RHS of an assignment may itself **be** a condition
(`scalar_expr` admits the entire `condition` non-terminal at
`:1145-1149`). Two of the spec's own surfaces collide on the
overlap:

1. **Example 4 of the Tier 2 section** (`:791-805`):

   ```python
   def firewall(pkt):
     internal = pkt.src_ip in [10.0.0.0/8, 192.168.0.0/16]
     if internal:
       allow
     ...
   ```

   The first line is a statement-position read (RHS of an
   assignment) **and** an in-expression containing a `pkt.src_ip`
   read. Per `:633` ("`pkt.src_ip`/`pkt.dst_ip` in statement position
   must be dominated by a guard that establishes the packet is
   IPv4. The only v4-establishing guards in v0.2 are conditions
   that read `pkt.src_ip` or `pkt.dst_ip` directly"), the
   surrounding function has no preceding guard — so the example
   is a **compile error**. Per `:635`'s "inside a condition" carve-
   out, it is **fine**.

2. **Edge case at `:661`** (Unguarded statement-level L4 read):

   > `port = pkt.dst_port` at the top of the function (no preceding
   > proto guard) is a compile error per the dominator rule above.

   This is presented as the canonical illustration of the rule. But
   what about `is_ssh = pkt.dst_port == 22`? Same statement
   position, same lack of proto guard — but the read of
   `pkt.dst_port` now lives "inside a condition" (an `==`). Two
   readings:

   | Reading | Outcome |
   |---|---|
   | `:633` wins (RHS-of-assignment ⇒ statement position ⇒ dominator) | compile error, same as bare `port = pkt.dst_port` |
   | `:635` wins (the read is inside a condition ⇒ short-circuit) | accepts; on non-TCP packets `is_ssh` is `false` |

   The spec gives no tiebreaker.

## The two-axis classification the spec leaves ambiguous

A `pkt`-field read in Tier 2 sits in some combination of:

| Axis A — outer container | Axis B — immediately enclosing expression |
|---|---|
| `if`/`elif`/`while` **condition** | bare field (`pkt.dst_port`) |
| RHS of an **assignment** | inside a comparison (`pkt.dst_port == 22`) |
| ` ` | inside an `in` (`pkt.src_ip in [10.0.0.0/8]`) |

Five non-trivial cells:

| Outer | Immediate | Spec resolution |
|---|---|---|
| `if`-cond | bare bool field (`if pkt.tcp.syn:`) | short-circuit (`:610`); dominator never applies |
| `if`-cond | comparison / `in` (`if pkt.dst_port == 22:`) | short-circuit (`:610`) |
| RHS-of-assign | bare field (`port = pkt.dst_port`) | **dominator applies** (`:633`, `:661`) |
| RHS-of-assign | comparison (`is_ssh = pkt.dst_port == 22`) | **ambiguous** — `:633` says yes, `:635` says no |
| RHS-of-assign | `in` (`internal = pkt.src_ip in [10.0.0.0/8]`) | **ambiguous** — Example 4 says no, `:633` says yes |

The two ambiguous cells are exactly the cells the spec exhibits in
its own examples and edge cases, in opposite polarities.

## Why this matters

Cross-oracle drift on a non-edge surface. Two implementers reading
the spec land on contradictory decisions for `internal = pkt.src_ip
in [10.0.0.0/8]`:

- **Implementer A** (reads `:633` literally — "RHS-of-assignment is
  statement position"): rejects the program with `error: 'pkt.src_ip'
  read on a path not guarded by an IPv4-establishing condition`.
- **Implementer B** (reads `:635` literally — "reads inside a
  condition use short-circuit"): accepts; on a v6 packet, `internal`
  evaluates to `false` (no v4 source), and the program proceeds
  normally.

Implementer A's compiler produces a hard error on the spec's own
Example 4. Implementer B accepts both Example 4 and `is_ssh =
pkt.dst_port == 22` at function top — but the latter is exactly the
kind of read that the original spec hole `tier2-stmt-level-pkt-read-
on-wrong-protocol` (now `status: fixed`) was meant to close.

The fix-of-the-fix can swing either way; what the spec cannot do is
leave both readings open while citing examples that demand
incompatible resolutions.

## Concrete .pkt cases two oracles will diverge on

### Case 1 — Example 4 verbatim, against an IPv6 packet

```python
@xdp(eth0)

def firewall(pkt):
  internal = pkt.src_ip in [10.0.0.0/8, 192.168.0.0/16]
  if internal:
    allow
  drop
```

Send: `tcp6(src_ip="2001:db8::1")`.

- Reading A (compile error): the program never builds.
- Reading B (accept; short-circuit makes `internal` false on v6):
  `internal = false`, falls through to `drop`. Action: `drop`.

If both implementers ship and one of them happens to evaluate
`internal` to `true` on a v6 packet (e.g., reads garbage bytes from
the v6 source-address offset that happen to fall in `10.0.0.0/8`
— low probability but real), the action becomes `allow` instead of
`drop`. **Class A drift.**

### Case 2 — `is_ssh = pkt.dst_port == 22` at function top

```python
@xdp(eth0)

def firewall(pkt):
  is_ssh = pkt.dst_port == 22
  if is_ssh:
    drop
  allow
```

Send: `icmp(src_ip="1.2.3.4")`.

- Reading A (compile error): never builds.
- Reading B (accept; short-circuit on non-TCP/UDP makes `is_ssh`
  false): `is_ssh = false`, action `allow`.
- Reading C (a third "literal" reading where `:635` exempts L3
  reads only — note `:635` says `pkt.src_ip`/`pkt.dst_ip`, not all
  L4 fields): rejects only the L4 cells, accepts the L3 cells.
  Then `is_ssh = pkt.dst_port == 22` is a compile error but Example
  4 (`internal = pkt.src_ip in [...]`) is fine. The two examples
  diverge cleanly under reading C.

Reading C — restricting the carve-out to L3 reads — is the most
defensible reading of `:635` taken in isolation, but it produces an
asymmetry the spec does not motivate: why would L3 reads-inside-a-
condition-on-the-RHS-of-an-assignment escape the dominator while
L4 reads-inside-a-condition-on-the-RHS-of-an-assignment do not?
Both have the same garbage-byte hazard on a non-matching packet.

## Proposed fix

Pick one of the three readings explicitly. The cleanest is reading
**B** (short-circuit wins for any `pkt`-field read appearing inside
a comparison or `in` expression, regardless of where the comparison
sits):

1. Edit `:635` to read:

   > Reads of any `pkt.<field>` *inside* a comparison or `in`
   > expression are governed by v0.1's short-circuit semantics —
   > reaching the comparison reads the field only when the parse
   > succeeded for the field's family/protocol, and the comparison
   > evaluates to false otherwise. This holds whether the comparison
   > sits inside an `if`/`elif` condition, inside an assignment RHS,
   > or inside a parenthesized sub-condition.
   >
   > The Tier 2 dominator rule applies only to **bare-field reads
   > in statement position** — assignments of the form
   > `<local> = pkt.<field>` (no surrounding comparison or `in`),
   > which evaluate the field directly with no fall-through path.
   > Example 4 of this section
   > (`internal = pkt.src_ip in [10.0.0.0/8, 192.168.0.0/16]`) is
   > governed by short-circuit semantics, not by the dominator
   > rule, because the read sits inside an `in` expression.

2. Edit the table at `:713-716` to scope the dominator-error rows
   to "bare-field" reads:

   | Condition | Error |
   |---|---|
   | `<local> = pkt.<L4 field>` (bare; no surrounding comparison) at a point not dominated by a proto guard | `error: '<field>' read on a path not guarded by 'pkt.proto == <required>'` |
   | `<local> = pkt.src_ip6` / `pkt.dst_ip6` (bare) in statement position not dominated by an IPv6-establishing guard | … |
   | `<local> = pkt.src_ip` / `pkt.dst_ip` (bare) in statement position not dominated by an IPv4-establishing guard | … |

3. Mirror the wording in `:633` so the v4 paragraph distinguishes
   "statement-position read" from "comparison-RHS read".

The alternative — reading A (dominator wins; reject Example 4 and
Case 1) — closes the same hole but invalidates Example 4 and forces
the user to wrap every L3-read assignment in a guard, which
Example 4 explicitly chose not to do. Reading C (L3 carve-out, L4
not) is the least defensible: it asymmetrically privileges L3.

## Class

Class A precursor (cross-oracle drift on a real Tier 2 idiom) with
a spec-layer root cause. The fix to a previous spec hole introduced
this new one — the "RHS of an assignment is statement position"
phrasing collides with the "RHS of an assignment can be a
condition" surface that the same spec admits at `:1145`.

## Test idea

Once a fix lands, the corpus should include:

```yaml
program: |
  @xdp(eth0)

  def firewall(pkt):
    internal = pkt.src_ip in [10.0.0.0/8, 192.168.0.0/16]
    if internal:
      allow
    drop
expected:
  compiles: true
  cases:
    - packet: tcp(src_ip="10.0.0.1")
      action: allow
    - packet: tcp(src_ip="8.8.8.8")
      action: drop
    - packet: tcp6(src_ip="2001:db8::1")
      action: drop          # internal=false; v6 src has no v4 read
```

And the negative case to lock the bare-field rule:

```yaml
program: |
  @xdp(eth0)

  def firewall(pkt):
    port = pkt.dst_port
    if port == 22:
      drop
    allow
expected:
  compiles: false
  compile_error_pattern: "pkt.dst_port.* not guarded"
```
