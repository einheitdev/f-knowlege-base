---
id: finding/2026-05-01-v02-spec-geoip-lhs-pkt-fields-only-vs-type-based-ambiguous
type: finding
protocol: []
builtins: [geoip]
severity: medium
layer: spec
pattern_tags: [spec-ambiguity, geoip, tier2, locals, type-rules, operators]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: [finding/2026-05-01-v02-spec-grammar-operand-overpermissive-for-geoip-call, finding/2026-05-01-v02-spec-tier2-list-typed-locals-grammar-permits-but-no-type]
---

# 2026-05-01-v02-spec-geoip-lhs-pkt-fields-only-vs-type-based-ambiguous

## Summary

The spec specifies the legal LHS of `... in geoip(...)` two
incompatible ways — one rule says "the four pkt IP fields, full
stop", the other says "any operand whose type is `ipv4` or `ipv6`,
which the compiler dispatches on". Tier 2 introduces ipv4/ipv6
LOCALS for the first time, and the two readings diverge on whether
`local_ipv4 in geoip(RU)` is legal.

The grammar admits the form (`comparison = lvalue "in"
set_or_range;` plus `lvalue = value_field | identifier;`), so this
is not a parse-time question. Two implementors of the analyser will
reach different verdicts on the same v0.2 source. Class B
(spec/spec contradiction across three sections that all bear on
the same surface).

Severity: medium because it surfaces only on user code that pulls
an ipv4/ipv6 packet field into a local before testing it against
geoip — which is exactly the natural Tier 2 idiom for "compute once,
test multiple ways".

## Locations

### Reading A — pkt-fields-only (Construct intro)

`FWL_V02_SPEC.md:297-301`:

> The result is an opaque set of IP prefixes; **both `pkt.src_ip` and
> `pkt.src_ip6` (and the `dst_*` variants) accept it as the right
> operand of `in`**.

This enumerates exactly the four `pkt` IP fields as the legal LHS
positions. No locals.

### Reading A again — pkt-fields-only (Operators section)

`FWL_V02_SPEC.md:883-884`:

> The right operand of `in` accepts two new forms in v0.2: …
> - A `geoip(...)` call (when the left operand is any of
>   `pkt.src_ip`, `pkt.dst_ip`, `pkt.src_ip6`, `pkt.dst_ip6`).

Same closed enumeration. Two sections agree on this reading.

### Reading B — type-based (Type rules)

`FWL_V02_SPEC.md:313-321`:

> `geoip(...)` produces an opaque value of type `set<ipv4 | ipv6>`.
> **The compiler picks which underlying index is queried based on the
> type of the left operand of `in`**:
> - `pkt.src_ip in geoip(...)` and `pkt.dst_ip in geoip(...)` query
>   the IPv4 LPM trie for this call site.
> - `pkt.src_ip6 in geoip(...)` and `pkt.dst_ip6 in geoip(...)` query
>   the IPv6 LPM trie for this call site.

The first sentence says **type-based** dispatch. The bullet list
that follows uses the four `pkt` fields as *examples*, not as a
closed set. A reader who keys off the first sentence — "based on
the type of the left operand" — concludes that any ipv4-typed
operand (including a Tier 2 ipv4 local) queries the v4 trie, and
any ipv6-typed operand queries the v6 trie.

### Grammar — admits the form

`FWL_V02_SPEC.md:1108-1114`:

```ebnf
comparison      = lvalue comp_op rvalue
                | lvalue "in" set_or_range ;

lvalue          = value_field
                | identifier ;
```

The grammar permits `<identifier> in geoip(...)` cleanly. There is
no parser-level rejection.

### Compile-errors table — silent

`FWL_V02_SPEC.md:457-466`:

| Mixed-family `geoip(...)` against a wrong-family operand | falls under the existing type rule of `in` (e.g. `pkt.src_ip in geoip(RU)` is fine; the operand types decide which trie is queried) |

This row is consistent with Reading B — "the operand types decide
which trie" — and uses the type-rule language. But it doesn't
mention locals, and it doesn't explicitly enumerate "Tier 2 local
LHS for `in geoip(...)` is rejected" as an error. Under Reading A,
this row is the only place the spec acknowledges that `in geoip(...)`
is type-driven, contradicting Reading A's pkt-fields-only
enumeration.

## Concrete user code that hits this

The natural "compute once, classify multiple ways" Tier 2 idiom:

```python
@xdp(eth0)

def firewall(pkt):
  src = pkt.src_ip                    # ipv4 local
  if src in [10.0.0.0/8, 192.168.0.0/16]:
    allow                             # internal
  if src in geoip(RU, CN, KP):        # <-- the surface this finding is about
    drop
  allow
```

Per Reading A: `src in geoip(RU, CN, KP)` is rejected because `src`
is not one of `pkt.src_ip`/`pkt.dst_ip`/`pkt.src_ip6`/`pkt.dst_ip6`.
The user must rewrite to `pkt.src_ip in geoip(RU, CN, KP)` and
duplicate the field access (which the dominator-rule and the
"compute once" motivation argued against).

Per Reading B: `src` has type `ipv4`; `geoip(RU, CN, KP)` is a
`set<ipv4|ipv6>`; the compiler selects the v4 LPM trie (because
LHS is `ipv4`); the lookup proceeds. The program is valid.

Same shape on the v6 side:

```python
@xdp(eth0)

def firewall(pkt):
  src6 = pkt.src_ip6
  if src6 in geoip(RU, CN):
    drop
  allow
```

Reading A rejects; Reading B accepts.

## Why this matters

### The Tier 2 "compute once" motivation depends on this surface

The Tier 2 design rationale (line 793-807, Example 4) showcases
exactly the pattern of "extract a field into a local, then use the
local in multiple subsequent expressions". Example 4's local
(`internal = pkt.src_ip in [...]`) is `bool`-typed because its
RHS is a comparison. The natural sister pattern — a `bool` *and*
ipv4-local extraction —

```python
src = pkt.src_ip
internal = src in [10.0.0.0/8, 192.168.0.0/16]
geo_blocked = src in geoip(RU, CN)
```

is what users will write. Whether `src in geoip(RU, CN)` is legal
is exactly the surface this finding asks about.

### The two readings produce different `expected.compiles` verdicts

A `.pkt` test built around the `src = pkt.src_ip; src in geoip(RU)`
pattern:

- Implementation tracking Reading A: `expected.compiles: false`,
  with an analyser error like
  `error: 'in geoip(...)' requires LHS to be one of pkt.src_ip,
  pkt.dst_ip, pkt.src_ip6, pkt.dst_ip6; got local 'src'`.
- Implementation tracking Reading B: `expected.compiles: true`,
  with `expected.actions` reflecting the geoip lookup result.

Two analyser implementors, both reading the spec, write different
.pkt fixtures and the corpus regression diverges across builds.

### The pkt-field enumeration reads as the limit, not the example

In the "Construct" paragraph (line 297-301), the four pkt fields
are listed in a sentence that grammatically reads as the *full*
allowed set, not as an enumeration of examples. A reader doesn't
hunt for a contradicting "actually any type-compatible operand"
sentence elsewhere; they take the first authoritative-looking
enumeration as the rule.

The Operators section (line 883-884) reinforces the same closed
enumeration with the same four fields. So Reading A is supported
by *two* sections.

The Type rules section (line 313-321) supports Reading B with
explicit "based on the type of the left operand" wording. The
bulleted examples that follow happen to enumerate the same four
pkt fields, but a reader takes those as illustrations of the
type-based rule, not as a separate closed set.

The ambiguity is exactly the kind that two careful spec readers
disagree on — one reads "field-enumerated", the other "type-based",
neither can point to text that explicitly forbids the other
reading.

### The `iso3166` validation and prefix-budget checks key off this

The compile-time validation flow for `geoip(...)` involves several
analyser passes:

1. Validate every `cc_code` is in the alpha-2 list.
2. For each call site, determine the bound family (Reading A:
   from the LHS field; Reading B: from the LHS type).
3. Allocate a map ID per call site (per the post-fix model — see
   the related finding on map-id-per-callsite-vs-per-callsite-
   family-pair).
4. At bundle time, count prefixes per call site against the
   65 536 budget.

Step 2 is the surface this finding is about. Reading A keys off
field name (a syntactic check). Reading B keys off operand type
(a type-system check). The two passes are different code paths in
the analyser; an implementor who picks one and not the other gets
verdicts the spec doesn't reconcile.

## Proposed fix

Pick one. Both are coherent. The wording fix is small in either
direction.

### Option A — pkt-fields-only

Edit `FWL_V02_SPEC.md:313-321` to drop the type-based language and
state the enumeration explicitly:

> `geoip(...)` produces an opaque value of type `set<ipv4 | ipv6>`.
> The legal LHS positions for `<lhs> in geoip(...)` are exactly the
> four pkt IP fields: `pkt.src_ip`, `pkt.dst_ip`, `pkt.src_ip6`,
> `pkt.dst_ip6`. The LHS field name determines the underlying
> trie:
> - `pkt.src_ip` and `pkt.dst_ip` query the IPv4 LPM trie.
> - `pkt.src_ip6` and `pkt.dst_ip6` query the IPv6 LPM trie.
>
> A Tier 2 local on the LHS — even one whose type is `ipv4` or
> `ipv6` — is rejected: `error: 'in geoip(...)' requires LHS to be
> one of pkt.src_ip, pkt.dst_ip, pkt.src_ip6, pkt.dst_ip6; got
> '<expr>'`.

Add the corresponding compile-errors row.

This is the more conservative resolution. It matches the
field-enumeration in the Construct intro and Operators section
verbatim. The cost is the user must inline the field access at
every geoip use site.

### Option B — type-based

Edit `FWL_V02_SPEC.md:297-301` and `FWL_V02_SPEC.md:883-884` to
drop the closed enumeration and use the type-based rule:

> The result is an opaque set of IP prefixes; **any LHS of type
> `ipv4` or `ipv6` accepts it as the right operand of `in`**. The
> LHS type determines which underlying trie is queried — `ipv4`
> LHSes hit the v4 trie, `ipv6` LHSes hit the v6 trie. In Tier 1
> the only `ipv4`/`ipv6`-typed operands are the four pkt fields
> (`pkt.src_ip`, `pkt.dst_ip`, `pkt.src_ip6`, `pkt.dst_ip6`);
> Tier 2 admits `ipv4`/`ipv6` locals on the LHS as well.

The Type rules section already says this; the other two sections
just need to align.

This is the more user-friendly resolution. It matches the natural
"compute once, reuse" Tier 2 idiom and aligns with the
already-spec'd `local in [<cidr_list>]` pattern (which the type
rules permit per the same type-based dispatch).

### What this finding does not change

Either resolution leaves untouched:

- The geoip codes-must-be-uppercase-alpha-2 rule.
- The map-allocation-per-call-site rule.
- The 65 536-prefix budget.
- The `geoip(...)`-only-as-RHS-of-`in` constraint.

Only the surface of "is `<local-of-ipv4-or-ipv6-type> in geoip(...)`
legal" changes.

## Class

Class B (spec contradiction — Construct intro and Operators say
"pkt-fields-only", Type rules say "type-based"; the grammar admits
the form and the compile-errors table is silent on the
construct-intro reading). Spec layer.
