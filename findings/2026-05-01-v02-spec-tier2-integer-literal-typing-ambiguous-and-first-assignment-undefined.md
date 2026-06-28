---
id: finding/2026-05-01-v02-spec-tier2-integer-literal-typing-ambiguous-and-first-assignment-undefined
type: finding
protocol: []
builtins: []
severity: medium
layer: spec
pattern_tags: [spec-ambiguity, tier2, locals, type-rules, integer-literals]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: [finding/2026-05-01-v02-spec-tier2-non-bool-local-as-bare-condition-undefined]
---

# 2026-05-01-v02-spec-tier2-integer-literal-typing-ambiguous-and-first-assignment-undefined

## Summary

Two adjacent holes in the Tier 2 type rules at
`FWL_V02_SPEC.md:559-579`:

1. **Integer literals have two candidate types.** The type table
   (`:565-566`) lists "integer literals 0..65535" under `u16` and
   "integer literals fitting u32" under `u32`. The two ranges
   overlap — every u16 value also fits u32. The spec does not say
   which type a literal in the overlap takes, and the
   first-assignment rule depends on it.

2. **"First" assignment is undefined.** The first-assignment rule
   (`:559-560`) reads:

   > Their type is the type of the right-hand side of the **first**
   > assignment to the name within the function

   "First" can mean "first in source (lexical) order" or "first in
   control-flow (executed) order". A function with branched
   assignments (`if cond: x = u16-thing else: x = ipv4-thing`) has
   different "first" assignments depending on the packet.

Both ambiguities affect the same surface — type binding of a
local — and produce divergent verdicts on the same source. Class B
(spec underspecification at the type-rules layer) with downstream
Class A risk on programs that assign to a local in a branch.

## Hole 1 — Integer literal type

### What the table says

`FWL_V02_SPEC.md:562-569`:

| Type | Width | Examples of values |
|---|---|---|
| `bool` | 1 bit | `pkt.tcp.syn`, ... |
| `u16` | 16 bits | `pkt.src_port`, `pkt.dst_port`, integer literals 0..65535 |
| `u32` | 32 bits | integer literals fitting u32 |
| `ipv4` | 32 bits | `pkt.src_ip`, ... |
| `ipv6` | 128 bits | `pkt.src_ip6`, ... |
| `proto` | 8 bits | `pkt.proto` |

The literal `80` is in the u16 range *and* fits u32. The literal
`0xff` is in the u16 range *and* fits u32. The literal `100000` is
*not* in the u16 range; it fits u32. The literal `0xffffffff` fits
u32; it does not fit u16. The literal `0x100000000` fits neither.

### Three implementer behaviours on a single source

```python
def firewall(pkt):
  x = 80
  if x == pkt.dst_port:
    drop
  allow
```

- **Implementer A** (literals are smallest-fitting-unsigned): `80`
  is `u16`. `x` is bound `u16`. `x == pkt.dst_port` is `u16 == u16`.
  Compiles, drops on dst_port=80.
- **Implementer B** (literals are u32 by default): `80` is `u32`.
  `x` is bound `u32`. `x == pkt.dst_port` is `u32 == u16`. Type
  error per the existing rule that comparison operands must share
  a type. Compile error.
- **Implementer C** (literals coerce to LHS type when context
  demands): `80` in `x = 80` has no LHS context, defaults to u32;
  `x == pkt.dst_port` coerces `x` to u16. Compiles, but the local
  has occupied 8 bytes of stack instead of 8 (no behavioural
  difference on this case but distorts the stack-budget estimate
  for long programs).

Same source, three verdicts.

### Cross-reference: u32 has *only* the integer-literal source

Of the six types, `u32` has exactly one entry in the "Examples of
values" column: "integer literals fitting u32". No `pkt` field is
of type `u32`. So the only way for a local to take type `u32` is
to be initialised from an integer literal that the spec classifies
as u32. Whether the local *can* take type u32 at all depends on
the answer to hole 1.

If implementer A's reading wins (literals are smallest-fitting),
then `u32` is reachable only via literals ≥ 65536, e.g. `x =
70000`. Below that, every literal is u16, so a local cannot be u32
even if the user wrote `x = 0` and meant a 32-bit zero.

If implementer B's reading wins, `u32` is the default and `u16`
locals are unreachable (no way to spell a u16 literal). The type
table's u16 entry "integer literals 0..65535" is unreachable
literature.

### Concrete propagation

The Tier 2 SSH example (`FWL_V02_SPEC.md:730-740`) and the dogfood
(`:968-1000`) do not assign integer literals to locals, but
Example 5 (`:806-832`) does:

```python
e = pkt.src_port
f = pkt.dst_port
if e == 22:
  log
if f == 22:
  count ssh
```

Here `e` and `f` are bound from `pkt.src_port`/`pkt.dst_port` (both
u16). The comparisons `e == 22` and `f == 22` use literal `22`.
Under implementer A: `22` is u16, comparison is u16 vs u16. ✓.
Under implementer B: `22` is u32, comparison is u16 vs u32 → type
error. The example doesn't compile.

Example 5's stack-budget claim ("The estimator should report ~64
bytes of stack ... the two ipv4 and two u16 locals share the
rest") commits to the u16 reading: if `e` and `f` were u32, the
total would be 80 bytes, not 64. So the example *implicitly*
endorses implementer A. But the spec's prose does not.

## Hole 2 — "First" assignment

### What the rule says

`FWL_V02_SPEC.md:559-560`:

> Locals are statically typed. Their type is the type of the
> right-hand side of the **first** assignment to the name within
> the function

### Concrete program where "first" matters

```python
def firewall(pkt):
  if pkt.proto == tcp:
    x = pkt.src_port      # u16 RHS
  else:
    x = 0                 # u16 or u32, depending on hole 1
  if x == 80:
    drop
  allow
```

In source (lexical) order, the first assignment is `x =
pkt.src_port` (u16). `x` is bound u16. The second assignment is in
the `else`-branch and must agree (u16). ✓

In control-flow (executed) order, on a TCP packet the first-
executed assignment is `x = pkt.src_port` (u16). On a non-TCP
packet the first-executed assignment is `x = 0` (u16 by hole 1's
implementer A reading; u32 by implementer B). Different bindings
per packet — implementations that pick "control-flow first" need
either a packet-dependent type or a join rule.

A more inflammatory example:

```python
def firewall(pkt):
  if pkt.proto == tcp:
    x = pkt.src_ip          # ipv4 RHS
  else:
    x = pkt.src_ip6         # ipv6 RHS — only valid if the packet IS v6
                             # (the `else` branch in fact runs on v6 too,
                             #  if proto != tcp on a v6 packet)
  if x in [10.0.0.0/8]:      # ipv4-only set
    drop
  allow
```

Source-order: `x` is ipv4 from line 3. The line-5 `x = pkt.src_ip6`
is type-mismatch (per the reassign rule at `:577-579`) — compile
error.

Control-flow-order on a TCP IPv4 packet: first assignment is
`x = pkt.src_ip` (ipv4). ✓

Control-flow-order on a UDP IPv6 packet: first assignment is
`x = pkt.src_ip6` (ipv6). The later `x in [10.0.0.0/8]` becomes
ipv6 vs ipv4 — type error at runtime, but the spec doesn't define
runtime type errors.

The example also intersects the IPv4/IPv6 dominator rule
(`:626-628`); the `pkt.src_ip` read in the then-branch needs an
IPv4-establishing guard, which `pkt.proto == tcp` does *not*
satisfy. So the example fails the dominator rule before the
first-assignment ambiguity matters. But replace `pkt.src_ip` and
`pkt.src_ip6` with `0` and `1.2.3.4` (or any literals of different
types) and the dominator rule is satisfied — only the
first-assignment ambiguity remains:

```python
def firewall(pkt):
  if pkt.proto == tcp:
    x = 0                # u16 or u32 (hole 1)
  else:
    x = 1.2.3.4          # ipv4
  ...
```

Source order: `x` bound to `int_type_of(0)` (u16 or u32). The
second assignment is ipv4 vs `int_type_of(0)` — type error per
the reassign rule.

Control-flow order on a TCP packet: `x` bound to `int_type_of(0)`.
Control-flow order on a non-TCP packet: `x` bound to ipv4.

The spec admits all four readings without picking one.

### The example "Local read before assigned"

`FWL_V02_SPEC.md:657`:

> *Local read before assigned.* Hard compile error: `error: local
> '<name>' read before assignment`. The check is path-sensitive:
> reading the local in a branch where it has not been assigned on
> every preceding path is the trigger.

This bullet *does* commit to control-flow analysis for the
read-before-assigned check: "every preceding path". So the
analyser already does control-flow tracking for one type rule.
The first-assignment rule could borrow the same control flow, or
it could be source-order. The spec doesn't say. The implementer
who reads the read-before-assigned bullet and the type-binding
sentence in sequence will likely pick control-flow for both — but
that's an inference, not a rule.

## Why this matters

Tier 2 locals are the central new construct in v0.2. Three Tier 2
worked examples (1, 4, 5) declare locals; one of them (5)
implicitly assumes the smallest-fitting-unsigned reading
(implementer A) for its stack-budget claim. The dogfood
(`:968-1000`) does not declare any integer locals — so the
canonical end-to-end example dodges hole 1 entirely.

Implementer A and implementer B will produce different verdicts on
Example 5 itself:

| Implementer | Example 5 verdict | Stack budget |
|---|---|---|
| A (smallest-fitting) | compiles | 64 bytes (matches the spec's claim) |
| B (u32 default) | compile error (`u16 == u32`) | n/a |
| C (LHS-coerce) | compiles | 80 bytes (disagrees with the spec's claim) |

Three Class-A flavours from one Class-B hole.

The same applies to hole 2 every time a user writes a local
inside an `if` branch — a normal idiom for "compute conditionally,
read later".

## Proposed fix

### For hole 1: pick A and state it.

Edit the integer-literal entries in the type table at
`FWL_V02_SPEC.md:565-566`:

| `u16` | 16 bits | `pkt.src_port`, `pkt.dst_port`, integer literals in `0..65535` (the inferred type for any decimal/hex literal that fits) |
| `u32` | 32 bits | integer literals in `65536..4294967295` (the inferred type for literals that exceed u16's range) |

Add one paragraph after the table:

> An integer literal's type is the smallest unsigned integer type
> from `{u16, u32}` that contains its value. So `5`, `0xff`,
> `65535`, `0x10` are `u16`; `65536`, `0x10000`, `4294967295` are
> `u32`. Literals exceeding u32 are a compile error: `error:
> integer literal <N> exceeds u32 range`.

(The "literals exceed u32" row goes into the Compile-errors table
too.)

This fixes Example 5's stack-budget claim — it now follows from
the spec rather than asserting against it.

### For hole 2: pick source order and state it.

Add to the first-assignment paragraph at `:559-560`:

> "First" means first in **source order** — the lexically earliest
> assignment to the name in the function body. Source-order
> binding is independent of which control-flow path actually
> reaches the assignment at runtime; the analyser walks the AST
> top-to-bottom, depth-first, and the first `<name> = <rhs>` it
> visits sets the type. Branched assignments
> (`if cond: x = a else: x = b`) must agree on type; the source-
> order-first one binds the local, and any subsequent assignment
> with a different type is a `cannot reassign as <T>` compile
> error.

The reassign rule at `:577-579` already covers the failure mode;
the new paragraph just ties source order to the existing rule.

## Class

Class B — spec underspecification at the type-rules layer. Two
ambiguities in the same five-line block. Implementations will
diverge silently on idiomatic Tier 2 programs. The fixes are
editorial; pick a reading and commit.
