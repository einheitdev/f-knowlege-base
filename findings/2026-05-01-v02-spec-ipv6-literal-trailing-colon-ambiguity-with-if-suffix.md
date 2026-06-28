---
id: finding/2026-05-01-v02-spec-ipv6-literal-trailing-colon-ambiguity-with-if-suffix
type: finding
protocol: []
builtins: []
severity: medium
layer: spec
pattern_tags: [spec-ambiguity, lexical, ipv6, tier2, parser-ambiguity, example-vs-grammar]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: [finding/2026-05-01-v02-spec-tier2-example5-violates-grammar-and-dominator-rule, finding/2026-05-01-v02-spec-rfc5952-rule4-rejects-bare-zero-and-loopback-canonical-examples]
---

# 2026-05-01-v02-spec-ipv6-literal-trailing-colon-ambiguity-with-if-suffix

## Summary

The fixed Tier 2 stack-budget Example 5
(`FWL_V02_SPEC.md:805-836`) contains the line:

```python
      if c == 2001:db8::1:
```

This line has **two** distinct lexical roles for the `:`
character: the *internal* colons of the IPv6 literal `2001:db8::1`
and the *terminal* colon of the `if`-statement suffix. The spec's
lexical rules give no rule for distinguishing them. A
naive longest-match lexer fed `2001:db8::1:` cannot decide whether
the trailing `:` belongs to the IPv6 literal (extending it to
`2001:db8::1:` followed by more hextets) or to the `if`-suffix.

The same issue recurs every time an IPv6 literal terminates an
`if`/`elif` condition, an `assign_stmt` line whose RHS is an IPv6
literal in a list (`[…, ::1]:` is unaffected because the closing
`]` separates), or any other production where `:` immediately
follows the literal.

Class B (lexical underspecification) — load-bearing because
multiple worked examples depend on the form parsing.

## Locations

### Examples that exhibit the form

`FWL_V02_SPEC.md:822-825` (Example 5, the rewritten stack-budget
test):

```python
    if c == 2001:db8::1:
      allow
    if d == 2001:db8::2:
      drop
```

`FWL_V02_SPEC.md:255-257` (Example "Drop one specific host"):

```python
drop if pkt.src_ip6 == 2001:db8:dead:beef::1
default allow
```

(In Tier 1 the line ends with NEWLINE rather than `:`, so the
ambiguity is dormant here. But see the Tier 2 form below.)

`FWL_V02_SPEC.md:264` (Example "Allow only a single /48
prefix"):

```python
allow if pkt.src_ip6 in 2001:db8:cafe::/48
```

(Same — Tier 1, no trailing `:`. Dormant.)

The Tier 2 examples that reach an `if`-suffix `:` immediately
after an IPv6 literal are where the ambiguity bites.

### What the spec says about IPv6 lexical structure

`FWL_V02_SPEC.md:55-63` enumerates "New literal kinds":

> | IPv6 literal | RFC 5952 canonical form | `2001:db8::1`, `::1`, `::`, `::ffff:1.2.3.4` |
> | IPv6 CIDR    | IPv6 literal + `/` + prefix | `2001:db8::/32`, `::/0`, `fc00::/7` |

There is no statement about how the lexer terminates an IPv6
literal — i.e. what character classes are *not* part of an IPv6
literal. RFC 5952 itself describes the literal in isolation; it
doesn't specify how a lexer in a host language demarcates one.

The grammar at `FWL_V02_SPEC.md:1189`:

```ebnf
ipv6  = (* RFC 5952 canonical form *) ;
```

is a comment-only production. It defers entirely to RFC 5952 prose
and gives no regex-style or DFA-style description.

### What a naive lexer does

A standard regex covering RFC 5952 IPv6 literals — e.g. Lark's
common
`[A-Fa-f0-9]{1,4}(:[A-Fa-f0-9]{1,4}){0,7}|::|((:[A-Fa-f0-9]{1,4}){0,7})::((?:[A-Fa-f0-9]{1,4}){0,7})`
or the more careful Python `ipaddress`-derived form — uses greedy
matching by default. Fed the input

```
2001:db8::1:
```

the lexer attempts to extend the match as long as the next
character class is a hex digit or `:`. The trailing `:` *is* in
the IPv6 literal's character class, so the lexer consumes it. The
lexer then looks at the next character (a NEWLINE in Example 5,
or whitespace if the user added a space) and finds it does not
extend an IPv6 literal — so it tries the longest match it has
seen, which is `2001:db8::1:`. That string is **not** valid RFC
5952 (a trailing `:` with no following hextet is an unfinished
group), so the lexer either:

- **(i)** Backtracks one character to `2001:db8::1`, accepts the
  literal, and pushes `:` back. The if-suffix colon is recovered.
  This is what a careful longest-match-with-validity-backtrack
  lexer does (Lark with `priority` and `backtrack` features, hand-
  rolled lexer with a final-state validity check).

- **(ii)** Fails the IPv6 literal match, falls back to a
  shorter prefix match. The shorter `2001:db8::1` is valid; the
  trailing `:` becomes the if-suffix. Same outcome as (i), but
  the failure path is different.

- **(iii)** Reports a lex error at `2001:db8::1:` ("invalid IPv6
  literal: trailing colon"), without realising the trailing `:`
  is the if-suffix. The user sees a confusing error pointing at
  the IPv6 literal when the actual cause is the parser's
  expectation of `:`.

Implementations (i) and (ii) work but require lexer features that
not every parser-generator front-end exposes by default. (iii) is
the failure mode the spec must rule out by giving an explicit
disambiguation rule.

### What about `2001:db8:dead:beef::1` (no trailing `:`)?

The Tier 1 rule form `drop if pkt.src_ip6 == 2001:db8:dead:beef::1`
ends with NEWLINE. The lexer's character class for IPv6 does not
include NEWLINE, so the literal is unambiguous. No issue. The
ambiguity is specifically when the literal is *immediately
followed* by a token whose character is in the IPv6 character
class — `:` is the only such token in v0.2 (the `if`-suffix, the
function-`:` after the param list, the `def`-suffix, and the
`elif`/`else`-suffix).

In Tier 2 every condition that ends with an IPv6 literal hits this
case: `if pkt.src_ip6 == ::1:`, `elif x == ::ffff:1.2.3.4:`, and
the assignment-RHS form `local = ::1` followed on a later line by
`if local == ::1:`.

## Why this matters

Three reasons:

1. **Worked Example 5 demonstrably depends on the resolution.**
   The rewritten Example 5 (per the fixed-finding
   `…-tier2-example5-violates-grammar-and-dominator-rule`) was
   chosen to be the *positive* test case for the analyser's
   stack-budget estimator. If the example doesn't parse, the
   stack estimator never runs against it and the Phase 2 corpus
   loses the case it was supposed to lock. The example currently
   sits in the spec as a positive program; the spec must give a
   rule that makes it parse.

2. **The corpus author cannot write Tier 2 IPv6-equality cases
   without knowing the rule.** Any `.pkt` test that follows the
   pattern "Tier 2 if-statement comparing a v6 local against a
   v6 literal" hits the ambiguity. Without a spec rule, the test
   either:
   - parses and produces a verdict (under lexer choice (i)/(ii))
   - fails to parse with a confusing error (under (iii))
   The cross-oracle disagreement is exactly the Class A shape
   the corpus is supposed to detect, applied to lexing.

3. **A future v0.3 surface that adds another `:` token (e.g.
   typed annotations `x: ipv4`) would amplify the ambiguity.**
   v0.2 has the chance to nail this rule down before adding more
   colon-bearing surfaces.

## Three resolutions

| Resolution | Effect |
|---|---|
| (a) Longest-match-with-RFC5952-validity-backtrack. | Specify that the lexer tries the longest character sequence whose character class is `[0-9a-fA-F:.]`, then validates against RFC 5952; if the longest match is invalid, it backs off one character at a time until a valid prefix is found, and the remainder is re-lexed. This is what (i) above describes; it is implementable in Lark/ANTLR with backtracking but requires explicit configuration. |
| (b) Whitespace-required around `:` after an IPv6 literal. | Require a space (or NEWLINE) between an IPv6 literal and a following `:` token. `if c == 2001:db8::1:` becomes `if c == 2001:db8::1 :`. This is a minor stylistic change; it eliminates the lexical ambiguity entirely. The example's prose would say "the space before the if-suffix `:` is required". |
| (c) Parenthesise the comparison. | Require `if (c == 2001:db8::1):` for any condition whose final operand is an IPv6 literal. The grammar already admits `( condition )` as a `primary`, so parsing falls out for free. The cost is verbosity in worked examples. |

(a) is the most user-friendly but most implementation-burdensome.
(b) is one character of source per literal-vs-suffix collision and
is the simplest spec rule. (c) is the most explicit but heaviest.

## Proposed fix (option a, with option b as fallback)

Add a paragraph to **Lexical Structure** at `FWL_V02_SPEC.md:51`:

> *IPv6-literal termination.* The lexer treats an IPv6 literal as
> the longest sequence of characters in `[0-9a-fA-F:.]` that is a
> valid RFC 5952 form. A trailing `:` that would make the
> sequence invalid (a hextet group with no following digits, or
> a non-canonical placement of `::`) is **not** consumed by the
> literal — it is returned to the parser. So
> `if c == 2001:db8::1:` lexes as `IF`, `IDENTIFIER(c)`, `EQ`,
> `IPV6(2001:db8::1)`, `COLON`, `NEWLINE` — the trailing `:` is
> the `if`-suffix.

Add a corresponding Edge case under "IPv6 Fields":

> *IPv6 literal immediately followed by `:`.* In a Tier 2
> `if`-condition like `if c == 2001:db8::1:`, the trailing `:`
> belongs to the `if`-suffix, not to the IPv6 literal. The lexer
> backtracks the longest IPv6 prefix that is RFC 5952-valid and
> emits the literal `2001:db8::1`; the trailing `:` is parsed as
> the statement-suffix. Worked Example 5 in the Tier 2 section
> exercises this rule.

Add corpus cases that lock the form:

```yaml
# corpus/from_hunt/<date>/v6-literal-trailing-if-colon.pkt
program: |
  @xdp(eth0)
  def firewall(pkt):
    if pkt.src_ip6 == 2001:db8::1:
      drop
    allow

description: |
  Tier 2 if-condition whose RHS is an IPv6 literal immediately
  followed by the if-suffix colon. Locks the lexer's
  longest-match-with-RFC-5952-validity-backtrack rule.
packets:
  - icmp6(src_ip="2001:db8::1", dst_ip="2001:db8::2")
expected:
  - {action: drop}
```

Add a complementary `expected.compiles: true` test for the
lexer-ambiguity-edge and an `expected.compiles: false` test for
a deliberately-malformed case (`2001:db8:::1` — three colons —
should remain a lex error). The corpus locks both directions.

## Class

Class B at the spec layer (lexical rule unspecified). Downstream
behaviour depends on lexer implementation choice; cross-oracle
agreement on Tier 2 v6-literal cases is contingent on a single
rule the spec must commit to.
