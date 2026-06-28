---
id: pattern/ipv6-literal-spec-incompleteness
type: pattern
protocol: []
builtins: []
severity: medium
layer: [spec]
pattern_tags: [ipv6, lexical, canonicalization, rfc-5952, parser-ambiguity, spec-ambiguity, spec-incompleteness]
status: hypothesis
created: 2026-05-04
instance_count: 2
instances:
  - finding/2026-05-01-v02-spec-ipv4-mapped-ipv6-canonical-form-ambiguous
  - finding/2026-05-01-v02-spec-ipv6-literal-trailing-colon-ambiguity-with-if-suffix
related_patterns:
  - example-contradicts-rule
  - grammar-vs-body-text-drift
related_findings:
  - finding/2026-05-01-v02-spec-rfc5952-rule4-rejects-bare-zero-and-loopback-canonical-examples
---

# ipv6-literal-spec-incompleteness

## Description

The v0.2 spec introduces IPv6 literals and CIDRs, defers their
canonical form to "RFC 5952", but only *partially* restates the
RFC's rules — and where it does restate, the restatement diverges
from the RFC. The grammar production is comment-only:

```ebnf
ipv6  = (* RFC 5952 canonical form *) ;
```

so every concrete decision about which IPv6 strings the parser
accepts lives in prose. The prose has at least three classes of
gap:

- **Canonicalization rules incomplete.** The spec restates three
  RFC 5952 rules (lowercase hex, suppressed leading zeros,
  longest `::` collapse) but omits §5 (IPv4-mapped form). The
  three-rule restatement, taken at face value, rejects
  `::ffff:1.2.3.4` (dotted-quad outside the 16-bit-group regime)
  and accepts `::ffff:0102:0304` — the opposite of RFC 5952 §5
  and the opposite of the spec's own Edge case example.
- **Canonicalization rules contradict their own examples.** The
  bare-zero (`::`) and loopback (`::1`) literals are listed as
  canonical-form examples in tables, but rule 4 (added to fix
  the IPv4-mapped ambiguity) rejects them. (This third instance
  is already filed under `example-contradicts-rule` — included
  here as a sibling.)
- **Lexical termination unspecified.** RFC 5952 describes the
  literal in isolation. The spec doesn't say what character
  classes terminate the literal in the host language — and the
  natural interaction with the new Tier 2 `if`-suffix `:` token
  produces a literal-vs-suffix collision (`if c == 2001:db8::1:`)
  the lexer must resolve.

The shape is uniform: **the canonical-form claim defers to RFC
5952 in the grammar, but the prose makes selective decisions
about which RFC rules to restate, and the selection is incoherent
with the RFC, with the prose's own examples, or with the host
language's lexical surface**. Two parser implementors land on
different accept/reject sets for the same `.fw` source.

The downstream effect is **parser-layer cross-oracle drift**: the
AST interpreter and BPF emitter share a common `parser._canonical_ipv6`
helper today (per the related findings), but the helper's
behaviour relative to the RFC's full rule set is implementation-
defined. A reader auditing the spec gets one accept set; an
implementer reading the RFC gets a different one; an implementer
following the prose's worked examples gets a third.

This is a hypothesis pattern (N=2 unique uncovered, plus one
sibling already in `example-contradicts-rule`) — the shape is
clear and high-leverage but the surface count is low. The IPv6-
literal surface in v0.2 has more sub-questions still unaudited
(see "Where to Look Next"), and each is a candidate for this
pattern.

## Check Strategy

1. **Walk RFC 5952 rule-by-rule against `FWL_V02_SPEC.md:84-89`
   and `:185-189`.** For every RFC rule (§4 rules 1–4 plus §5
   on special addresses), confirm the spec either restates it
   verbatim or explicitly waives it. Gaps and divergences are
   findings of this pattern.
2. **For every IPv6-literal example in the spec**, confirm it
   passes both the spec's restated rules and RFC 5952. Examples
   that pass one but fail the other are findings (cross-pattern
   with `example-contradicts-rule`).
3. **For every Tier 2 / IPv6 / lexical position where an IPv6
   literal can appear immediately before another token**,
   enumerate the trailing characters and confirm the lexer's
   termination rule produces a deterministic split. The
   trailing-colon + if-suffix case is one; trailing `]` (in a
   list literal — already disambiguated by `]` not being in the
   IPv6 character class), trailing `,`, trailing `)`, trailing
   NEWLINE are all candidates.
4. **For every CIDR form** (`<ipv6>/<prefix>`), confirm the
   canonicalization rule applies to the address half identically
   to a bare literal. The spec at `:193-196` makes this claim
   verbatim; if the canonicalization rule is incomplete, the CIDR
   form inherits the same gap.
5. **Run the v0.1 corpus through the v0.2 parser.** Any IPv6
   literal in a v0.1 program (none exist — v0.1 has no v6) is
   moot, but any IPv6 literal in a v0.2-only `.pkt` test must
   parse the same way under interpreter and emitter. Cross-test
   with the `parser._canonical_ipv6` helper as the single source
   of truth.
6. **Cross-walk `PKT_V02_SPEC.md` builder fields**. The PKT spec's
   `tcp6`/`udp6`/`icmp6` builders accept IPv6 strings; whether
   they apply the same canonicality enforcement as the FWL parser
   is the surface for `pkt-loader-silent-acceptance`'s
   v6-noncanonical finding. Cross-pattern: the PKT spec must
   either inherit the FWL canonicality rule or document its own.

## Known Instances

- `finding/2026-05-01-v02-spec-ipv4-mapped-ipv6-canonical-form-ambiguous`
  — the spec's three-rule restatement rejects `::ffff:1.2.3.4`
  and accepts `::ffff:0102:0304`; RFC 5952 §5 says the opposite;
  the Edge case prose example uses `::ffff:1.2.3.4`. Three
  implementer paths, three accept sets.
- `finding/2026-05-01-v02-spec-ipv6-literal-trailing-colon-ambiguity-with-if-suffix`
  — the lexer cannot determine whether the trailing `:` in `if c
  == 2001:db8::1:` belongs to the IPv6 literal or the if-suffix.
  Three lexer behaviours possible; spec gives no rule.

Sibling (already in `example-contradicts-rule`):

- `finding/2026-05-01-v02-spec-rfc5952-rule4-rejects-bare-zero-and-loopback-canonical-examples`
  — canonicalization rule 4 (added to address the IPv4-mapped
  ambiguity) rejects `::` and `::1` while the section's own
  tables list both as canonical. Same shape: rule restated
  selectively, restatement contradicts examples.

## Where to Look Next

The IPv6-literal surface has several still-unaudited corners.
Each is a candidate for the same pattern.

- **RFC 5952 §4 rule 1 (uppercase hex letters rejected).** The
  spec says "lowercase hex digits" but doesn't enumerate the
  reject case. A `.pkt` test asserting `2001:DB8::1` is rejected
  needs the spec's compile-error string; the table at `:204-214`
  has a generic "non-canonical IPv6 literal" row but no
  case-specific message.
- **RFC 5952 §4 rule 2 (suppressed leading zeros).** Same shape:
  the spec rejects `2001:0db8::1` but the rejection-message
  granularity isn't pinned. Likely same compile-error row, but
  worth confirming.
- **RFC 5952 §4 rule 3 (longest `::` run; first run on tie).**
  The spec restates "longest run, first such run if multiple
  runs tie" but doesn't enumerate ties. A literal like
  `2001:0:0:1:0:0:1:1` has two two-zero runs and either could
  be `::`'d. Whether the spec's "first such run" wins under the
  lexer's longest-match-with-validity-backtrack rule (proposed
  in the trailing-colon finding) is unaudited.
- **CIDR-with-non-canonical-address.** The spec at `:193-196`
  says "the address half is canonicalized identically to a
  literal." Every gap in the literal rule is a gap here too —
  e.g. `::ffff:0102:0304/128` ambiguity is the IPv4-mapped
  finding applied to CIDRs.
- **Zone-ID suffix (`fe80::1%eth0`).** RFC 4007 / RFC 6874 admit
  zone IDs on link-local addresses. The spec doesn't mention
  them; whether the parser rejects (and with what error) is
  unenumerated. Operator NDP firewalls touching `fe80::/10`
  surface this — cross-pattern with `user-rule-intent-mismatch`'s
  v6-internal NDP finding.
- **Embedded IPv4 (`::1.2.3.4`, the IPv4-compatible form,
  deprecated per RFC 4291).** The spec's IPv4-mapped paragraph
  mentions "IPv4-mapped" but is silent on IPv4-compatible. RFC
  4291 deprecates the form; v0.2 should explicitly reject or
  accept.
- **Lexer trailing-`,` case in list literals.** `[2001:db8::1,
  ::1]` — the closing `]` is unambiguous, but the comma after
  `2001:db8::1` is in a position where the lexer might consume
  it as part of an unfinished IPv6 literal under naive longest-
  match. This is the same shape as the trailing-colon finding;
  audit list contexts.
- **`PKT_V02_SPEC.md` builder field for IPv6 strings.** The
  builder uses Python's `ipaddress` module which accepts every
  RFC 4291-valid form (already a fixed `pkt-loader-silent-
  acceptance` finding). Whether the FWL parser's canonicalizer
  is bug-compatible with the builder's parser is the cross-
  oracle surface; either parser is a candidate for new findings
  as RFC 5952 corner cases are audited.
- **Future v0.3 typed-annotation surface (`x: ipv6 = ...`).** The
  trailing-colon finding flagged this as a v0.3 amplification.
  When v0.3 adds explicit type annotations using `:`, every
  v0.2-decision about IPv6-literal lexical termination needs to
  be re-validated against the new colon position.
