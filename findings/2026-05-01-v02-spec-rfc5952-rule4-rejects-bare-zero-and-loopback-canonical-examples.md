---
id: finding/2026-05-01-v02-spec-rfc5952-rule4-rejects-bare-zero-and-loopback-canonical-examples
type: finding
protocol: []
builtins: []
severity: high
layer: spec
pattern_tags: [spec-inconsistency, spec-contradiction, ipv6, canonicalization, rfc-5952, example-vs-rule]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: [finding/2026-05-01-v02-spec-ipv4-mapped-ipv6-canonical-form-ambiguous]
---

# 2026-05-01-v02-spec-rfc5952-rule4-rejects-bare-zero-and-loopback-canonical-examples

## Summary

Canonicalization rule 4 in the IPv6-Fields section
(`FWL_V02_SPEC.md:116`) was added to fix the
`finding/2026-05-01-v02-spec-ipv4-mapped-ipv6-canonical-form-ambiguous`
ambiguity. As written, it requires that the entire `::/96` block —
"IPv4-compatible" addresses, *i.e.* every address whose high 96 bits
are zero — be expressed in dotted-quad form. That block contains
`::` (the unspecified address) and `::1` (the IPv6 loopback). Both
addresses are listed as canonical-form examples in the same section
(`FWL_V02_SPEC.md:59` and `:95`) and used unwrapped throughout the
spec's worked examples. The two rules contradict.

A second, smaller error in the same rule: RFC 5952 §5 itself does
not require dotted-quad form for IPv4-compatible addresses — it
applies only to IPv4-mapped (`::ffff:0:0/96`). The spec attributes a
broader rule to RFC 5952 §5 than the RFC actually mandates.

Class B (spec self-contradiction; spec layer; pre-implementation, so
the fix is editorial). The downstream Class-A risk is concrete — the
parser, AST interpreter, and BPF emitter are about to be written
against a rule that says `::1` is not canonical, while every example
they will be tested against uses `::1`.

## The contradiction in one place

`FWL_V02_SPEC.md:116` (rule 4 of the four canonicalization rules):

> 4. *RFC 5952 §5 — IPv4-mapped and IPv4-compatible IPv6 addresses.*
>    When the address is an IPv4-mapped IPv6 address (high 80 bits
>    zero, next 16 bits `0xFFFF`, low 32 bits the embedded IPv4
>    address — the `::ffff:0:0/96` block) **or an IPv4-compatible
>    IPv6 address (high 96 bits zero, low 32 bits the embedded IPv4
>    address — the `::/96` block)**, the trailing 32 bits **must**
>    be expressed as a dotted quad. Hex form (`::ffff:0102:0304`)
>    is non-canonical for these addresses; the canonical form is
>    the dotted-quad form (`::ffff:1.2.3.4`).

Read against the literal IPv6 spec for "high 96 bits zero":

| Address | Decoded | Falls in `::/96`? | Spec rule 4 says |
|---|---|---|---|
| `::`   | `0:0:0:0:0:0:0:0`     | yes (all 128 bits zero) | non-canonical; canonical is `::0.0.0.0` |
| `::1`  | `0:0:0:0:0:0:0:1`     | yes                     | non-canonical; canonical is `::0.0.0.1` |
| `::a`  | `0:0:0:0:0:0:0:a`     | yes                     | non-canonical; canonical is `::0.0.0.10` |
| `::ff` | `0:0:0:0:0:0:0:ff`    | yes                     | non-canonical; canonical is `::0.0.0.255` |
| `::ffff` | `0:0:0:0:0:0:0:ffff` | yes                    | non-canonical; canonical is `::0.0.255.255` |
| `::ffff:1.2.3.4` | `0:0:0:0:0:ffff:0102:0304` | no (group 6 is `ffff`) | covered by IPv4-mapped half of rule 4 — dotted quad is correct |
| `::ffff:0102:0304` | same | no | non-canonical (per IPv4-mapped half) |

So under rule 4 the first five rows are non-canonical literals, and
`::1` and `::` are reduced to `::0.0.0.1` and `::0.0.0.0`.

Now read the spec's **own** example tables and worked examples:

- **Lexical Structure new-literal-kinds table** (`:59`):
  > | IPv6 literal | RFC 5952 canonical form | `2001:db8::1`, `::1`, `::`, `::ffff:1.2.3.4` |

  `::1` and `::` are listed verbatim as canonical-form examples.

- **IPv6 Fields operand table** (`:95`):
  > | IPv6 literal | RFC 5952 canonical form | `2001:db8::1`, `::1`, `::` |

  Same.

- **Edge cases** (`:211-213`):
  > *Zero address (`::`) and all-ones address ...* Both are valid
  > literals and matchable. `::/0` matches every IPv6 packet; it is
  > the v6 equivalent of `0.0.0.0/0`.

  `::/0` is a CIDR where the address half is `::`. The CIDR rule
  ("the CIDR's address half is canonicalized identically to a
  literal", `:225`) means `::` must be canonical. Rule 4 says it
  isn't; the example says it is.

- **Examples — Log all IPv6 traffic** (`:268-273`):
  > ```python
  > log if pkt.src_ip6 in ::/0
  > ```

  Same `::/0` literal, presented as a valid program. Rule 4 makes
  it a compile error.

The contradiction is not subtle: a one-line fix to canonicalization
rule 4 erased the canonical-ness of two of the four IPv6 literals
the spec itself uses as canonical-form examples.

## Why this is the fix-finding's regression

The earlier finding
`finding/2026-05-01-v02-spec-ipv4-mapped-ipv6-canonical-form-ambiguous`
(status: fixed) proposed rule 4 to disambiguate the IPv4-mapped
case. The proposed text was:

> 4. For IPv4-mapped IPv6 addresses (`::ffff:0:0/96`) and
>    IPv4-compatible IPv6 addresses (`::/96`, deprecated but still
>    representable), the trailing 32 bits MUST be expressed as a
>    dotted-quad. Hex form is non-canonical.

That proposal is what landed (more or less verbatim) at line 116.
The bug is in the proposal: extending RFC 5952 §5's dotted-quad
rule from the IPv4-mapped block to the entire `::/96` block was an
overreach.

RFC 5952 §5's actual scope (quoted from the RFC):

> 5. Text Representation of Special Addresses
>
> Addresses such as IPv4-Mapped IPv6 addresses ([RFC4291], Section
> 2.5.5.2), ISATAP [RFC5214], and IPv4-translatable addresses
> [ADDR-FORMAT] have an embedded IPv4 address.
>
> When expressing an IPv4-mapped IPv6 address, the form
> `::ffff:172.16.254.1` SHOULD be used. ...

Note: the IPv4-Compatible IPv6 address (`::a.b.c.d`) is *not* in
RFC 5952 §5's enumeration; RFC 4291 §2.5.5.1 (which defines it) was
declared deprecated by RFC 4291 itself. RFC 5952 §5 deliberately
does not take a position on its canonical text form, because no one
should be writing those addresses anymore.

So the spec is wrong on two counts:

1. Rule 4 says RFC 5952 §5 mandates dotted-quad for IPv4-compatible
   addresses. It does not.
2. Rule 4 says the `::/96` block must use dotted-quad. That block
   contains `::` and `::1`, which the spec uses freely as canonical.

The first error is editorial. The second is operational — it
breaks the spec's own examples.

## Why this matters

Three implementer paths land on three different behaviours:

| Implementer | Reading | Effect |
|---|---|---|
| (a) Reads rule 4 literally | rejects `::`, `::1`, `::a`, `::ff`, `::feed`, etc.; rejects `::/0`, `::1/128`; the spec's examples don't compile | `log if pkt.src_ip6 in ::/0` is a compile error |
| (b) Reads the canonical-form table at line 95 as authoritative | accepts `::`, `::1`; rejects only `::ffff:0102:0304` (the IPv4-mapped hex form) | matches the rest of the spec but contradicts rule 4 |
| (c) Asks "what does RFC 5952 actually say?" and follows RFC 5952 §5 strictly | applies dotted-quad only to the `::ffff:0:0/96` block (IPv4-mapped); leaves IPv4-compatible alone | matches the spec's examples but the rule-4 text is wrong about `::/96` |

The corpus oracle implications are concrete: a `.pkt` test that
writes `pkt.src_ip6 == ::1` lands in compile-error verdict on (a)
and compile-success on (b)/(c) — silently. The dogfood example at
the end of `FWL_V02_SPEC.md` does not write `::` or `::1`, but the
"v6-aware internal network" example writes `fc00::/7` (which is
fine — `fc00` is non-zero in the high bits) and `::/0` (which is
broken on path (a)).

The `truncate_to` worked example in `PKT_V02_SPEC.md:184` uses
`drop if pkt.src_ip6 in ::/0` directly. On path (a) implementers,
the `.pkt` test fails to compile rather than asserting truncation
behaviour.

## Proposed fix

Edit `FWL_V02_SPEC.md:116`. Drop the IPv4-compatible half of rule 4
entirely; restrict the dotted-quad mandate to the IPv4-mapped block:

> 4. *RFC 5952 §5 — IPv4-mapped IPv6 addresses.* When the address
>    is an IPv4-mapped IPv6 address (high 80 bits zero, next 16
>    bits `0xFFFF`, low 32 bits the embedded IPv4 address — the
>    `::ffff:0:0/96` block, RFC 4291 §2.5.5.2), the trailing 32 bits
>    **must** be expressed as a dotted quad. Hex form
>    (`::ffff:0102:0304`) is non-canonical; the canonical form is
>    the dotted-quad form (`::ffff:1.2.3.4`).
>
>    The IPv4-compatible IPv6 address form (`::a.b.c.d`, deprecated
>    by RFC 4291) is **not** specially canonicalized in v0.2 —
>    addresses in `::/96` other than the IPv4-mapped block follow
>    rules 1–3 only. `::`, `::1`, `::a`, `::ffff:1.2.3.4` are all
>    canonical.

Then update the Compile-errors row at `FWL_V02_SPEC.md:240` to drop
the IPv4-compatible half and rename:

| Hex form of an IPv4-mapped IPv6 address (`::ffff:0102:0304`) | `error: IPv4-mapped IPv6 literal must use dotted-quad form per RFC 5952 §5; expected '<canonical>'` |

The single example used in the table (`::0102:0304` for the
IPv4-compatible case) goes away — there is no canonical-form rule
to enforce on it under rules 1–3.

A second, narrower fix: leave rule 4's IPv4-compatible language in
place but mark `::` and `::1` as **explicit exceptions** ("the
unspecified address `::` and the loopback `::1` are canonical in
their bare form, not as `::0.0.0.0` and `::0.0.0.1`"). This keeps
the spec's intent (force dotted-quad on most `::/96` addresses) but
preserves the two well-known special-purpose addresses. The risk
is that the exception list is hard to bound — `::a` is the test
address `::0.0.0.10`; should that round-trip to `::a` or
`::0.0.0.10`? RFC 5952 doesn't decide. Picking a clean fix
(option-1 above — drop the IPv4-compatible half) is simpler.

## Class

Class B — spec self-contradiction. The same section that defines
`::` and `::1` as canonical-form examples also defines a rule that
makes them non-canonical. Implementations will diverge on every
small `::/96` literal. Pre-implementation, so the fix is editorial.
