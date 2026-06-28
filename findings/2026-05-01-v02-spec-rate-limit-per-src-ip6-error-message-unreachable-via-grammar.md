---
id: finding/2026-05-01-v02-spec-rate-limit-per-src-ip6-error-message-unreachable-via-grammar
type: finding
protocol: []
builtins: [rate_limit]
severity: medium
layer: spec
pattern_tags: [spec-contradiction, grammar, error-message, rate_limit, per-field, ipv6, unreachable-error]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related:
  - finding/2026-05-01-v02-spec-rate-limit-per-src-ip-with-v6-only-predicate-dead-rule
---

# 2026-05-01-v02-spec-rate-limit-per-src-ip6-error-message-unreachable-via-grammar

## Summary

The v0.2 spec talks about `per=src_ip6` (and other invalid `per=`
fields) as a compile error in two places — `FWL_V02_SPEC.md:192-195`
and the compile-errors table at `:244` — and prescribes a specific
analyser-level diagnostic:

> error: rate_limit per= must be src_ip, dst_ip, src_port, or dst_port

But the grammar at `:1153-1155` defines `rl_field` as a four-element
literal alternation:

```ebnf
rate_limit_call = "rate_limit" "(" integer "," "per" "=" rl_field ")" ;
rl_field        = "src_ip" | "dst_ip" | "src_port" | "dst_port" ;
```

`rl_field` accepts no other terminal. Any input where `per=` is
followed by anything else is rejected by the **parser**, not the
analyser, and the user sees the parser's "unexpected token" message
— not the prescribed analyser string.

This is structurally analogous to the `def f(pkt):\n    pass` issue:
the spec promises a dedicated analyser error string for an input
that the parser preempts. The promised string is unreachable under
the grammar as written.

The case is more acute for `per=src_ip6` than for `per=foo` because
of the lexer's longest-match behaviour: `src_ip6` tokenises as
`RL_FIELD("src_ip")` followed by `INTEGER(6)`, so the parser blows
up *inside* the parens, not at `per=`. The user sees a confusing
"unexpected INTEGER 6" error pointing at what looks like the wrong
position. The actual error string the v0.1 implementation produces
on the equivalent v0.1 source is:

```
error: 3:63: Unexpected token Token('INTEGER', '6') at line 3, column 63.
Expected one of:
        * RPAR
```

The spec's "rate_limit per= must be src_ip, dst_ip, src_port, or
dst_port" message is structurally unreachable under the grammar.

Class B (spec/grammar contradiction; spec layer; pre-implementation,
fix is editorial). Inherits from a v0.1 issue but is freshly
amplified by `:192-195`'s explicit discussion of `per=src_ip6`.

## Demonstration

### v0.1 implementation, mirroring the v0.2 grammar

```sh
$ cat > /tmp/test_rl_v6.fw <<'EOF'
@xdp(eth0)

allow if pkt.proto == tcp limited by rate_limit(10, per=src_ip6)
default allow
EOF

$ f-hone/.venv/bin/fwl compile /tmp/test_rl_v6.fw -o /tmp/test_rl_v6.bpf.c
error: 3:63: Unexpected token Token('INTEGER', '6') at line 3, column 63.
Expected one of:
        * RPAR
```

The lexer matches the four-character RL_FIELD token `src_ip` greedily
(it is the longest valid prefix) and then re-enters the lexer at `6`,
emitting INTEGER. The parser expects RPAR after `rl_field` and
chokes on INTEGER. The user never sees the spec's prescribed
"rate_limit per=" message.

### Other invalid `per=` inputs go the same way

`per=foo` lexes as `per`, `=`, `IDENTIFIER("foo")`. The parser
expects RL_FIELD, sees IDENTIFIER, errors. Again no analyser
diagnostic.

`per=src_ip5` (typo): same as `src_ip6` — `src_ip` + INTEGER(5),
parser error.

`per=src_addr` (the unified form deferred to v0.3 per `:191-194`):
lexes as `per`, `=`, `IDENTIFIER("src_addr")`, parser expects
RL_FIELD, errors with "unexpected IDENTIFIER".

In all four cases the user sees a parser-level "unexpected token"
diagnostic, not the spec's prescribed analyser-level message.

## The two paragraphs that contradict

### `:192-195` and `:244` — promise a specific analyser error

`:192-195`:

> ... v0.3 will introduce `per=src_ip6` (and a unified `per=src_addr`
> form). Specifying `per=src_ip6` in v0.2 is a compile error:
> `rate_limit per= must be src_ip, dst_ip, src_port, or dst_port`.

`:244`:

| Condition | Error |
|---|---|
| `per=src_ip6` in `rate_limit` | `error: rate_limit per= must be src_ip, dst_ip, src_port, or dst_port` |

Both promise a single analyser-level string keyed off the *intent*
of the user (they wanted `src_ip6`).

### `:1153-1155` — grammar admits only four exact terminals

```ebnf
rate_limit_call = "rate_limit" "(" integer "," "per" "=" rl_field ")" ;

rl_field        = "src_ip" | "dst_ip" | "src_port" | "dst_port" ;
```

This is a strict alternation. The parser cannot reach the analyser
with a fifth field — the lexer/parser combination produces a
syntax error first.

## Why this matters

Two oracles checking the same `.pkt` test:

```yaml
program: |
  @xdp(eth0)
  allow if pkt.proto == tcp \
    limited by rate_limit(10, per=src_ip6)
  default allow

expected:
  compiles: false
  compile_error: "rate_limit per= must be src_ip, ..."
```

- Oracle A (the analyser): produces a parser error
  ("Unexpected token INTEGER 6"). Test fails. Author has to look at
  the spec, see the compile-error string, and conclude the spec is
  wrong.
- Oracle B (a hand-written reference parser that interprets the spec
  literally): tries to enforce the prescribed error. Cannot —
  there's no analyser pass that ever sees a non-whitelisted
  `rl_field` because the grammar already filtered it.

The v0.2-specific damage is in the `per=src_ip6` discussion: the
spec specifically calls out this case at `:192-195` because v0.3
will eventually allow it, and an operator migrating from v0.2 to
v0.3 needs a clear error at the seam. The error string the spec
promises is the right operator-facing message, but the grammar
prevents it from ever firing.

## Proposed fix

Two clean options.

**(a) Loosen the grammar; let the analyser produce the error.**

Change `:1155` from a four-way alternation to a generic identifier:

```ebnf
rl_field        = identifier ;
```

And add an analyser pass: "if the rl_field identifier is not one of
{src_ip, dst_ip, src_port, dst_port}, emit `error: rate_limit per=
must be src_ip, dst_ip, src_port, or dst_port`." This puts the
prescribed message under the analyser's control and yields a clear
diagnostic for `per=src_ip6` (lexed as IDENTIFIER `src_ip6`),
`per=src_addr`, etc.

The lexer-priority issue with `src_ip6` (eager longest-match against
`src_ip`) goes away because `rl_field` is now an open identifier
slot. Implementation cost: one `if` in the analyser; nothing else
moves.

**(b) Update the spec to acknowledge the parser-level diagnostic.**

Edit `:192-195` and the `:244` row to reflect what the user actually
sees:

> Specifying `per=src_ip6` (or any `per=` value other than the four
> listed) is a *parser-level* error. The user-visible message is the
> parser's normal "unexpected token" form (e.g. `unexpected token
> '6', expected ')'` for `per=src_ip6`; `unexpected token
> 'src_addr', expected one of: src_ip, dst_ip, src_port, dst_port`
> for `per=src_addr`). The pre-existing "rate_limit per= must be
> src_ip, dst_ip, src_port, or dst_port" wording is not produced by
> the v0.2 parser.

(b) is honest but operator-hostile — `unexpected token '6', expected
')'` doesn't tell the user that `src_ip6` was the intended target,
and the column number points at the `6` not the `src_ip6`.

**(a) is the operator-friendly fix and the smaller spec change.**
The v0.1 grammar inherits the same bug; (a) fixes both at once.

## Cross-reference: same shape elsewhere

The exact same shape appears in `:245` for the IPv6 range syntax
`a..b`:

| IPv6 range syntax `a..b` | parser-level syntax error — `..` is the port-range operator and the parser does not accept an IPv6 literal on either side. The user-visible message is the parser's normal "unexpected token `..`" form. |

Note that this row *does* acknowledge the parser-level path:
"the user-visible message is the parser's normal `unexpected token
..` form." The `per=src_ip6` row at `:244` should be edited to use
the same language, OR the grammar should be loosened per option
(a). Pick one and apply it consistently.

## Class

Class B (spec/grammar contradiction; the prescribed error is
unreachable). Spec layer.
