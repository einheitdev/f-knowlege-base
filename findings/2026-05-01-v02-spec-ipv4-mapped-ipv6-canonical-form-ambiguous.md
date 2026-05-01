---
id: finding/2026-05-01-v02-spec-ipv4-mapped-ipv6-canonical-form-ambiguous
type: finding
protocol: []
builtins: []
severity: medium
layer: spec
pattern_tags: [spec-ambiguity, ipv6, canonicalization, rfc-5952]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: []
---

# 2026-05-01-v02-spec-ipv4-mapped-ipv6-canonical-form-ambiguous

## Summary

`FWL_V02_SPEC.md` requires IPv6 literals to be in "RFC 5952
canonical form" but only enumerates three of the RFC's
canonicalization rules. The IPv4-mapped/embedded-IPv4 special case
defined in RFC 5952 §5 is not restated, and the rules that *are*
restated contradict it. As a result, `::ffff:1.2.3.4` and
`::ffff:0102:0304` cannot both be canonical under the spec — but
the spec presents the dotted-quad form as a valid literal in its
Edge cases section. An implementer cannot tell which form the
parser must accept.

Class B (spec ambiguity, parser-layer impact).

## The contradiction

### What the spec says canonical means

`FWL_V02_SPEC.md:84-89`:

> IPv6 literals must be in RFC 5952 canonical form: lowercase hex
> digits, suppressed leading zeros within each 16-bit group, and a
> single `::` collapsing the longest run of all-zero groups (or the
> first such run if multiple runs tie). Non-canonical input
> (`2001:DB8::1`, `2001:db8:0:0::1`, `2001:db8:0000::1`) is a compile
> error. The compiler does not silently canonicalize — the source has
> to be canonical so diffs and grep are well-behaved.

Three rules:

1. lowercase hex digits;
2. suppressed leading zeros within each 16-bit group;
3. single `::` collapsing the longest run of all-zero groups.

These rules, taken at face value, would canonicalize the IPv4-mapped
prefix `::ffff:0102:0304` to itself (it already passes all three).
They would *reject* the dotted-quad form `::ffff:1.2.3.4` because
`1.2.3.4` is not "lowercase hex digits"; it's a dotted-decimal
quad outside the 16-bit-group structure the rules describe.

### What RFC 5952 actually says about IPv4-mapped addresses

RFC 5952 §5 ("Text Representation of Special Addresses") states:

> When expressing an IPv4-mapped IPv6 address, the form
> `::ffff:172.16.254.1` SHOULD be used. ... The dotted-decimal
> notation MUST be used for the IPv4 portion ...

So under RFC 5952, the canonical form for an IPv4-mapped IPv6
address is the *dotted-quad* form, **not** the all-hex form. The
rule the spec restates (rules 1–3 above) is a strict subset of
RFC 5952 that does not cover §5. The two-rule sets disagree on
what is canonical for IPv4-mapped addresses.

### The Edge case uses the dotted-quad form

`FWL_V02_SPEC.md:185-189`:

> *IPv4-mapped IPv6 addresses (`::ffff:1.2.3.4`).* Treated as IPv6
> literals; they match `pkt.src_ip6` only when the actual frame is
> IPv6 carrying a v4-mapped address. They do **not** match an IPv4
> packet whose source is `1.2.3.4`. (Cross-family equivalence is a
> v0.3 question.)

`::ffff:1.2.3.4` is presented as a valid IPv6 literal a user can
write. By rules 1–3 from line 84, this literal is non-canonical
(the trailing `1.2.3.4` violates rule 1 — those characters aren't
"lowercase hex digits within a 16-bit group"). By RFC 5952 §5, the
all-hex form `::ffff:0102:0304` is non-canonical and the dotted-quad
form is canonical. The spec's three-rule reformulation and its Edge
cases example therefore disagree on the canonical form.

### The grammar is silent

`FWL_V02_SPEC.md:1142`:

```
ipv6          = (* RFC 5952 canonical form *) ;
```

The grammar punts to RFC 5952 with no further detail. Whether the
dotted-quad form is admitted as a `cc_code`/literal is left to the
prose, which is internally contradictory.

## Why this matters

Three independent implementer paths land on three different
behaviours:

| Implementer | Reading | Effect |
|---|---|---|
| (a) Reads only the three-rule rephrasing | rejects `::ffff:1.2.3.4` as non-canonical | Edge case example is uncompilable; the cross-family Edge case discussion is unreachable |
| (b) Reads the prose example as authoritative | accepts `::ffff:1.2.3.4` and rejects `::ffff:0102:0304` (per RFC 5952 §5) | needs an extra parser branch the three-rule restatement does not document |
| (c) Reads "RFC 5952 canonical form" literally | accepts the dotted-quad form and rejects the hex form for IPv4-mapped addresses *only* | requires recognising IPv4-mapped patterns before deciding which form is canonical — a check the spec's three rules don't describe |

The corpus oracle implications are concrete: a `.pkt` test that
asserts `pkt.src_ip6 == ::ffff:0102:0304` (compiled per (a) or (b))
produces three different verdicts under (a)/(b)/(c). And an
implementer who chooses (a) makes the Edge cases section's
IPv4-mapped paragraph dead spec (the user can't write the literal it
discusses).

The same problem is latent in the corresponding CIDR rule
(`FWL_V02_SPEC.md:193-196`):

> *CIDR with non-canonical address.* `2001:DB8::/32` is a compile
> error per the canonicalization rule; `2001:db8:0::/32` is also a
> compile error. The CIDR's address half is canonicalized
> identically to a literal.

If the address half is canonicalized "identically to a literal,"
then `::ffff:1.2.3.4/128` and `::ffff:0102:0304/128` are *both* under
the same ambiguity.

## Proposed fix

Pick one of two choices and edit the spec to match.

### Option 1 — Honour RFC 5952 §5 (dotted-quad canonical)

Add a fourth rule to `FWL_V02_SPEC.md:84-89`:

> 4. For IPv4-mapped IPv6 addresses (`::ffff:0:0/96`) and
>    IPv4-compatible IPv6 addresses (`::/96`, deprecated but still
>    representable), the trailing 32 bits MUST be expressed as a
>    dotted-quad. Hex form is non-canonical.

Add a row to the Compile errors table (`FWL_V02_SPEC.md:204-214`):

| `::ffff:0102:0304` (hex form of an IPv4-mapped address) | `error: IPv6 literal must be in canonical (RFC 5952) form; expected '::ffff:1.2.3.4'` |

### Option 2 — Diverge from RFC 5952; lock the all-hex form

Edit `FWL_V02_SPEC.md:185-189` to change the example from
`::ffff:1.2.3.4` to `::ffff:0102:0304`, drop the dotted-quad form
from the prose, and add a note that v0.2 deviates from RFC 5952 §5
in requiring all-hex form. This is consistent with the
"three-rule" rephrasing but makes the spec text "RFC 5952 canonical
form" misleading; the note is required.

Either option is fine as long as it's stated. The status quo —
"RFC 5952 canonical" plus a three-rule restatement that doesn't
match RFC 5952 plus an Edge cases example that uses a fourth form —
is ambiguous.

## Class

Class B — spec ambiguity. Spec layer. Pre-implementation, so the
fix is editorial. Without it, parser, AST interpreter, and BPF
emitter teams will independently choose among (a)/(b)/(c) and the
three oracles will disagree on `::ffff:0102:0304` / `::ffff:1.2.3.4`
inputs.
