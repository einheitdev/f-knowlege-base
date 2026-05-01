---
id: finding/2026-05-01-v02-spec-tier2-ordered-comparison-on-u16-u32-locals-undefined
type: finding
protocol: [tcp, udp]
builtins: []
severity: high
layer: spec
pattern_tags: [spec-contradiction, tier2, locals, type-rules, operators, ordered-comparison]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: [finding/2026-05-01-v02-spec-tier2-proto-local-vs-local-comparison-disallowed-but-no-error, finding/2026-05-01-v02-spec-tier2-integer-literal-typing-ambiguous-and-first-assignment-undefined]
---

# 2026-05-01-v02-spec-tier2-ordered-comparison-on-u16-u32-locals-undefined

## Summary

The Operators section at `FWL_V02_SPEC.md:887-889` says:

> Ordered comparisons remain restricted to integer port fields
> (`pkt.src_port`, `pkt.dst_port`) and integer literals.

This restricts ordered comparisons (`<`, `<=`, `>`, `>=`) to *exactly
two* LHS positions — `pkt.src_port` and `pkt.dst_port`. By a strict
reading, ordered comparisons with a Tier 2 `u16` or `u32` local on
the LHS are illegal: `if my_port < 1024:` does not name `pkt.src_port`
or `pkt.dst_port`, so the LHS does not satisfy the operators rule.

The Tier 2 type rules at `FWL_V02_SPEC.md:557-572` and the compile-
errors table at `FWL_V02_SPEC.md:706-731` say the opposite (or nothing
at all):

- The type rules describe `u16` and `u32` as first-class local types
  with only one cross-type restriction ("no implicit widening or
  narrowing — a comparison between a `u16` operand and a `u32`
  operand is a type error"). There is no operator-class restriction
  (no equivalent of the `proto`-only "ordered comparisons are not
  permitted" sentence at line 572).
- The compile-errors table lists "`proto`-typed value used with
  `<`, `<=`, `>`, `>=`, `in`, or arithmetic" but does **not** list
  any "`u16` local used with ordered comparison" or
  "ordered comparison on a non-port-field operand" row.

So the operator section forbids `if my_port < 1024:`, the type-rules
section permits it, and the compile-errors table has no error to
emit for the Operators-section reading. Two implementors of the v0.2
analyser will produce divergent verdicts on the same source.

Class B (spec contradiction across sections). Spec layer.
Load-bearing because the natural Tier 2 idiom — compute a value
once, compare it multiple times — is the construct most likely to
hit this.

## Locations

### Operators section — restrictive

`FWL_V02_SPEC.md:887-889`:

> Comparison operators (`==`, `!=`) accept IPv6 literals on the right
> when the left is an `ipv6` field. **Ordered comparisons remain
> restricted to integer port fields (`pkt.src_port`,
> `pkt.dst_port`) and integer literals.**

The bolded text enumerates a closed set of two LHS positions for
ordered comparisons. No locals appear in that set.

### Type rules — permissive (by omission)

`FWL_V02_SPEC.md:561-572`:

| Type | Width | Examples of values |
|---|---|---|
| `u16` | 16 bits | `pkt.src_port`, `pkt.dst_port`, integer literals in `0..65535` |
| `u32` | 32 bits | integer literals in `65536..4294967295` |

> An integer literal's type is the smallest unsigned integer type
> from `{u16, u32}` that contains its value. … There is no implicit
> widening or narrowing in v0.2 — a comparison between a `u16`
> operand and a `u32` operand is a type error.

The only operator-class restriction in the entire type-rules
section is the `proto`-only paragraph at line 572:

> Locals of type `proto` may participate in equality and inequality
> comparisons (`==`, `!=`) … Arithmetic, ordered comparisons (`<`,
> `<=`, `>`, `>=`), `in`, and comparisons against any non-`proto`-
> typed value are all type errors.

The same paragraph is conspicuously absent for `u16` and `u32`. A
reader takes that as "ordered comparisons on u16/u32 locals are
fine" by analogy with how they're fine on `pkt.src_port` /
`pkt.dst_port` (which the type table tells them are also `u16`).

### Compile-errors table — silent

`FWL_V02_SPEC.md:721-722`:

| Condition | Error |
|---|---|
| `proto`-typed value used with `<`, `<=`, `>`, `>=`, `in`, or arithmetic | `error: 'proto' values support only equality (==, !=); got '<op>'` |
| `proto`-typed value compared against a non-`proto`-typed value | `error: cannot compare proto value with <T> value` |

No row exists for "ordered comparison with a `u16` or `u32` local
on the LHS" or "ordered comparison with a non-port-field operand on
the LHS". An analyser that follows the operators section's
restrictive rule would have to invent its own error message.

### Grammar — admits the form

The grammar `comparison = lvalue comp_op rvalue;` plus
`lvalue = value_field | identifier;` plus
`comp_op = "==" | "!=" | "<" | ">" | "<=" | ">=";` admits
`my_port < 1024` cleanly. The Operators-section restriction is
therefore an analyser-only check, but the analyser has no spec'd
error to emit.

## Concrete user code that hits this

The natural "compute once, compare twice" idiom Tier 2 was added to
support:

```python
@xdp(eth0)

def firewall(pkt):
  if pkt.proto == tcp:
    port = pkt.dst_port      # u16 local, type inferred from RHS
    if port < 1024:          # is this a privileged-port destination?
      log
    if port == 22:
      drop
  allow
```

- The dominator rule passes (every `pkt.dst_port` read sits inside
  `if pkt.proto == tcp:`).
- `port` is a `u16` local (RHS is a `u16` packet field).
- `port == 22` is plainly fine (equality, both `u16`).
- `port < 1024` is the surface this finding is about.

A second example — using `u32` for a future-proof construct (the
spec carves out `u32` as a type even though no v0.2 packet field is
`u32`):

```python
@xdp(eth0)

def firewall(pkt):
  threshold = 100000           # u32 literal (≥ 65536)
  if pkt.proto == tcp:
    if pkt.src_port < threshold:    # u16 LHS vs u32 RHS — already
                                    # spec'd as a type error
      drop
  allow
```

Here the `u16` vs `u32` mismatch is a spec'd error (per line 570).
But:

```python
@xdp(eth0)

def firewall(pkt):
  threshold = 22               # u16 literal (< 65536)
  if pkt.proto == tcp:
    if pkt.src_port < threshold:    # u16 LHS, u16 RHS local
      drop
  allow
```

Per the type rules, `pkt.src_port` is `u16`, `threshold` is `u16`
(its first-assignment RHS is the literal `22`, which fits `u16`).
The comparison is `u16 < u16`. The Operators section's restrictive
reading rejects this because the RHS isn't a literal — it's a
local. The type-rules-section reading accepts it. Two analysers
disagree.

## Why this matters

### The dogfood example dodges the question

The Tier 2 dogfood at `FWL_V02_SPEC.md:975-1005` uses ordered
comparison nowhere — every L4-port test is an `==` against a
literal or an `in [list]`. So the corpus-as-written never exercises
the surface. A user copying the dogfood pattern and adding a port-
range check is the trigger.

### The "no implicit widening" rule already requires per-operator
type checks

The analyser already has to do per-operator type checking to enforce
"u16 vs u32 is a type error". Adding "ordered comparisons need a
port-field LHS" or "ordered comparisons need both operands to be
the same integer type" is the same kind of check. The choice
matters; the spec doesn't make it.

### The Tier 1 vs Tier 2 split is invisible in the operators section

The Operators section header — "Operators (delta from v0.1)" — and
the section's content read as describing the operator surface for
*all* of v0.2, both Tier 1 and Tier 2. A user reading the operators
section in isolation infers that ordered comparisons are
port-field-only across the board.

In Tier 1 the rule is non-controversial: there are no locals, so
the only way to put a value on the LHS of `<` is via a packet field
or a literal, and `pkt.src_port` / `pkt.dst_port` are the only
ordered-compatible packet fields. The Tier 1 rule literally cannot
be violated by user code.

In Tier 2 the rule becomes load-bearing. Locals admit any of six
types as LHS. Two of them (`u16`, `u32`) are integer types whose
ordered comparison has obvious semantics. The spec inherits the
Tier 1 wording without saying whether it widens for Tier 2.

### Two implementations diverge on the same `.pkt`

A `.pkt` test built around the "compute once, compare twice"
pattern:

```fwl
@xdp(eth0)

def firewall(pkt):
  if pkt.proto == tcp:
    port = pkt.dst_port
    if port < 1024:
      drop
  allow
```

with packets at `pkt.dst_port == 22` and `pkt.dst_port == 8080`:

- The "operators-section is canonical" implementation refuses to
  compile the program. The .pkt's `expected.compiles: false` test
  matches.
- The "type-rules-section is canonical" implementation compiles it,
  drops the port-22 packet, allows the port-8080 packet. The .pkt's
  `expected.actions` for each packet is checkable.

Two analyser implementors, both reading the spec, produce different
verdicts on the same source. Class B.

## Proposed fix

Pick one. Both are coherent; the spec must commit.

### Option A — extend ordered comparisons to integer locals

Replace `FWL_V02_SPEC.md:887-889` with:

> Ordered comparisons (`<`, `<=`, `>`, `>=`) require both operands
> to be the same integer type (`u16` or `u32`). The valid LHS forms
> are `pkt.src_port`, `pkt.dst_port` (both `u16`), or any Tier 2
> local of type `u16` or `u32`. The valid RHS forms are integer
> literals (whose type is inferred per the integer-literal-typing
> rule at the start of the Tier 2 Type rules), Tier 2 locals of
> the same integer type as the LHS, or `pkt.src_port` / `pkt.dst_port`
> (when the LHS is also one of those fields).

Then add a row to the compile-errors table:

| Ordered comparison with mismatched integer types | `error: cannot compare <T1> with <T2> using <op>; ordered comparisons require matching integer types` |

This is the more user-friendly resolution. It matches the natural
"compute once, reuse" idiom Tier 2 was added for.

### Option B — keep ordered comparisons port-field-only

Replace `FWL_V02_SPEC.md:887-889` with the explicit version:

> Ordered comparisons (`<`, `<=`, `>`, `>=`) require the LHS to be
> one of `pkt.src_port` or `pkt.dst_port` and the RHS to be an
> integer literal in `0..65535`. Ordered comparisons with any other
> LHS — including a Tier 2 `u16` or `u32` local, an `ipv4` or `ipv6`
> field, or a `proto`-typed value — are a compile error: `error:
> ordered comparison <op> requires a port-field LHS; got '<expr>'`.

Then add the corresponding row to the compile-errors table.

This is the more conservative resolution. It matches the literal
wording of the current operators section but breaks the natural
Tier 2 idiom and forces users to inline every ordered comparison.

### What this finding does not change

Either resolution leaves untouched:

- The `proto`-only equality rule at line 572.
- The `ipv4`/`ipv6` ordered-comparison ban (no useful semantics).
- The `u16` vs `u32` type-mismatch rule.

Only the surface of "is `local < literal` legal in Tier 2" changes.

## Class

Class B (spec contradiction across sections — Operators says
"port-field-only", Type rules says "u16 locals are first-class with
no operator restriction", Compile errors lists no error for the
Operators-section reading). Spec layer.
