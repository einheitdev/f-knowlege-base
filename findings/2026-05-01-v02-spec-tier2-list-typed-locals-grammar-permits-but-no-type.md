---
id: finding/2026-05-01-v02-spec-tier2-list-typed-locals-grammar-permits-but-no-type
type: finding
protocol: []
builtins: []
severity: medium
layer: spec
pattern_tags: [spec-inconsistency, grammar, tier2, locals, type-rules]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: [finding/2026-05-01-v02-spec-tier2-locals-not-in-condition-grammar]
---

# 2026-05-01-v02-spec-tier2-list-typed-locals-grammar-permits-but-no-type

## Summary

The Tier 2 `expression` production at `FWL_V02_SPEC.md:1095-1099`
admits `list`, `cidr`, and `cidr_list` as the right-hand side of a
local assignment. The Tier 2 type table at
`FWL_V02_SPEC.md:539-547`, which enumerates every legal local type,
does *not* include any list-shaped type — the only types are
`bool`, `u16`, `u32`, `ipv4`, `ipv6`, `proto`. The compile-error
table at `FWL_V02_SPEC.md:683-700` does not list "non-scalar RHS in
local assignment" as an error class.

So a program like `ranges = [10.0.0.0/8, 192.168.0.0/16]` is
grammar-legal, has no defined local type, has no defined error
message, and (separately) has no usable downstream surface because
`set_or_range` does not admit `identifier`. An implementer cannot
tell whether the assignment compiles, what type the local takes if
it does, or what error to emit if it doesn't.

Class B (spec/grammar inconsistency).

## The contradiction

### Grammar admits the assignment

`FWL_V02_SPEC.md:1095-1099`:

```ebnf
expression    = condition
              | identifier                        (* local read *)
              | value_field                       (* packet field read *)
              | integer | ipv4 | ipv6
              | cidr | list | cidr_list ;
```

`list = "[" operand { "," operand } "]"` and `cidr_list = "[" cidr
{ "," cidr } "]"` (lines 1132, 1141). So a Tier 2 assignment of the
form

```python
ranges = [10.0.0.0/8, 192.168.0.0/16]
```

parses cleanly under `assign_stmt = identifier "=" expression
NEWLINE` (line 1033). Same for `cidrs = [10.0.0.0/8]`, `singletons
= [80, 443]`, `addr = 10.0.0.0/8`.

### Type table forbids it implicitly

`FWL_V02_SPEC.md:539-547`:

| Type | Width | Examples of values |
|---|---|---|
| `bool` | 1 bit | … |
| `u16` | 16 bits | `pkt.src_port`, integer literals 0..65535 |
| `u32` | 32 bits | integer literals fitting u32 |
| `ipv4` | 32 bits | `pkt.src_ip`, `pkt.dst_ip`, IPv4 literals |
| `ipv6` | 128 bits | `pkt.src_ip6`, `pkt.dst_ip6`, IPv6 literals |
| `proto` | 8 bits | `pkt.proto` |

Six rows. None list `list`, `cidr`, `cidr_list`, or `set` as a
local type. The first-assignment rule (`FWL_V02_SPEC.md:537`) says
"the type of the right-hand side of the **first** assignment" — but
the right-hand side `[10.0.0.0/8]` has type `cidr_list`, which is
not in the table.

### Compile-error table does not name it

`FWL_V02_SPEC.md:683-700` is the comprehensive compile-error list
for Tier 2. It includes:

- `local read before assigned`
- `local re-assigned with different type`
- `pkt.<L4 field>` read at a point not dominated by a proto guard
- `local named pkt`
- empty function body
- recursion, etc.

None of these match "RHS of assignment is a list literal" or "local
of type list/cidr/cidr_list is not allowed".

### Downstream surface is also closed

`FWL_V02_SPEC.md:1131`:

```
set_or_range  = list | range | cidr | cidr_list | geoip_call ;
```

`set_or_range` is the RHS of `... in <set_or_range>`. It does *not*
admit `identifier`. So even if the local could be assigned a
list, it could not be used:

```python
ranges = [10.0.0.0/8, 192.168.0.0/16]
if pkt.src_ip in ranges:    # parse error: identifier not in set_or_range
  allow
```

The user has no way to bind a list to a name and reuse it. The
construct is half-baked: assignable but not usable.

## Why this matters

Three independent implementer paths disagree:

| Implementer | Reading | Effect |
|---|---|---|
| (a) Honour the grammar, infer a `cidr_list`/`list` local type | Local is created but cannot be used; subsequent `pkt.<x> in ranges` is a parse error | The "Local declared but never read" warning fires; user sees a confusing surface where the assignment "works" but the only natural use of it doesn't |
| (b) Reject the assignment with an analyser-pass type error | Need a new compile-error message that the spec doesn't list | Different implementations land on different messages |
| (c) Tighten the grammar so `assign_stmt`'s `expression` cannot be a list literal | Grammar diverges from the prose `expression` production | Implementer must rewrite the grammar against the spec |

The most plausible user code that hits this is exactly Example 4
generalised:

```python
def firewall(pkt):
  internal = pkt.src_ip in [10.0.0.0/8, 192.168.0.0/16]
  trusted_v6 = pkt.src_ip6 in [2001:db8::/32, fc00::/7]
  ...
```

That works because `pkt.src_ip in [...]` is a `condition` (the
whole comparison), and `condition` evaluates to `bool` — a defined
local type. The user (or fuzzer) one step off would write:

```python
def firewall(pkt):
  internal_subnets = [10.0.0.0/8, 192.168.0.0/16]
  if pkt.src_ip in internal_subnets:
    allow
  ...
```

which parses partway and fails at the `in` site. The diagnostic the
user sees is unspecified.

## Proposed fix

Pick one and edit the spec to match.

### Option 1 — Tighten the grammar (preferred)

Restrict `assign_stmt`'s RHS to scalar expressions:

```ebnf
assign_stmt   = identifier "=" scalar_expr NEWLINE ;

scalar_expr   = condition
              | identifier
              | value_field
              | integer | ipv4 | ipv6 ;

(* `cidr`, `list`, `cidr_list`, `range`, `geoip_call` are NOT
   admissible as the RHS of a local assignment in v0.2; they exist
   only as the RHS of `in`. *)
```

And add a row to the compile-error table at
`FWL_V02_SPEC.md:683-700`:

| RHS of local assignment is a list, range, or CIDR literal | `error: '<name>' assignment RHS must be scalar (bool/integer/ipv4/ipv6/proto); list/range/CIDR literals are only valid on the right of 'in'` |

This is the cleanest fix because Tier 2 has no list-shaped values
anywhere else: comparisons, `in` RHS, and assignments are the three
ways to name a value, and only `in` RHS uses lists.

### Option 2 — Add a `cidr_list` local type and admit `identifier` in `set_or_range`

Extend the type table with one or two new rows (`cidr_list` for
v4-CIDR-only lists, `cidr_list_v6` for v6-CIDR-only — type-checked
on use), and amend `set_or_range`:

```ebnf
set_or_range  = list | range | cidr | cidr_list | geoip_call
              | identifier ;
```

This is a bigger surface change and creates a new local type
category (non-scalar) the spec did not discuss elsewhere; defer to
v0.3 if the user ergonomic is desired.

### Option 3 — Document "useless local" as the intended behaviour

Edit the spec to say: "A local of list-shaped type is permitted by
the grammar but has no usable surface in v0.2; this is by design,
to keep parsing rules simple. Implementations may emit a warning."

Worst of the three; promotes a confusing dead surface into the
language. Not recommended.

## Class

Class B — spec/grammar inconsistency. Spec layer. Pre-implementation,
so the fix is editorial. Without it, every Phase 2 implementer
team will independently choose among (a)/(b)/(c) above and the
diagnostic surface will fragment.
