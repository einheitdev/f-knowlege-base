---
id: finding/2026-05-01-v02-spec-new-reserved-words-break-strict-superset-claim
type: finding
protocol: []
builtins: []
severity: high
layer: spec
pattern_tags: [spec-contradiction, strict-superset, lexical, reserved-words, counter-names, v01-compat]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: [finding/2026-05-01-v02-spec-strict-superset-vs-cross-family-pkt-proto, finding/2026-05-01-v02-spec-tier2-reserved-keyword-set-not-enumerated]
---

# 2026-05-01-v02-spec-new-reserved-words-break-strict-superset-claim

## Summary

The v0.2 spec promises that "every v0.1 program is a valid v0.2
program with identical semantics" (`FWL_V02_SPEC.md:9-12`,
`:49`, and reinforced at `:171-174`, `:932-936`). The Lexical
Structure section then introduces seven new reserved words:
`def`, `elif`, `else`, `for`, `while`, `pass`, `return`
(`FWL_V02_SPEC.md:69`). All seven are valid v0.1 identifiers per
the v0.1 lexical rule (`FWL_V01_SPEC.md:39`,
`FWL_V01_SPEC.md:608`) and may legitimately appear as v0.1
counter names (`count <n>` where `<n>` is any
`[a-z_][a-z0-9_]*`).

A v0.1 program containing `count def`, `count for`, etc.
compiles cleanly in v0.1 (verified, see "Demonstration" below)
but is a v0.2 compile error per `FWL_V02_SPEC.md:82`:

> Using a reserved word in a forbidden position is a compile
> error: `error: '<word>' is reserved and may not be used as a
> <function|local|counter> name`.

The strict-superset claim and the new-reserved-words list are
mutually inconsistent. One of them must give.

Class B (spec contradiction). Will surface as a Class A
regression the moment `hone regress` runs the v0.1 corpus
through the v0.2 toolchain — any test case that uses one of the
seven keywords as a counter name fails to compile under v0.2.

## Evidence

### The strict-superset claim

`FWL_V02_SPEC.md:9-12`:

> This document specifies FWL v0.2. v0.2 is a **strict superset
> of v0.1**: every v0.1 program is a valid v0.2 program with
> identical semantics.

Repeated at `:49`:

> A v0.1 program runs unchanged under the v0.2 compiler. The
> Phase 1 corpus is the regression oracle for this guarantee.

Repeated at `:935-936` (inside the Compilation section):

> This is the strict-superset guarantee: any v0.1 program
> compiled by the v0.2 toolchain has identical packet behaviour
> to v0.1.

### The seven new reserved words

`FWL_V02_SPEC.md:67-78` (Reserved words):

> The following identifiers are reserved and may not appear as a
> Tier 2 function name, a Tier 2 local name, or a counter name.
>
> - *Statement keywords:* `def`, `if`, `elif`, `else`, `for`,
>   `while`, `pass`, `return`, `default`.

Of these, v0.1 had only `if` (always a syntactic keyword) and
`default` (already reserved as the start of `default <action>`).
The seven NEW reservations relative to v0.1 are:

```
def, elif, else, for, while, pass, return
```

The motivation given at `:78` is forward-compat:

> `for`, `while`, `pass`, `return` are reserved even though
> v0.2 does not admit them in any production — reserving them
> now means a future v0.3 that adds them does not retroactively
> rename a v0.2 user's local.

Sound forward-compat reasoning, but the rule applies equally to
v0.1 → v0.2. Reserving them in v0.2 does retroactively break
v0.1 programs that used them as counter names.

### v0.1 admits these as counter names

`FWL_V01_SPEC.md:39`:

> Identifiers match `[a-z_][a-z0-9_]*` — lowercase letters,
> digits (not in first position), and underscores. They appear
> as: ... and **counter names in `count <n>`**.

`FWL_V01_SPEC.md:608` (grammar):

```ebnf
identifier    = letter_lower { letter_lower | digit | "_" } ;
letter_lower  = "a" | "b" | ... | "z" | "_" ;
```

`def`, `elif`, `else`, `for`, `while`, `pass`, `return` all
match `letter_lower { letter_lower | digit | "_" }`. None of
them appears as a v0.1 grammar terminal, so the v0.1 lexer
emits them as `IDENTIFIER`.

### Compile-error rule in v0.2

`FWL_V02_SPEC.md:82`:

> Using a reserved word in a forbidden position is a compile
> error: `error: '<word>' is reserved and may not be used as a
> <function|local|counter> name`.

So a `count def` rule from a v0.1 program becomes a v0.2
compile error.

## Demonstration

The shipped v0.1 compiler (`f-hone/.venv/bin/fwl`, current
master) accepts every one of the seven new reserved words as a
counter name. Sample (one of seven; the other six produce
identical exit-zero output):

```
$ cat /tmp/test_kw.fw
@xdp(eth0)
count def if pkt.proto == tcp
default allow

$ fwl compile /tmp/test_kw.fw -o /tmp/test_kw.bpf.c
$ echo $?
0
```

Repeating with `def`, `elif`, `else`, `for`, `while`, `pass`,
`return` substituted produces exit code 0 in every case (full
test loop in the agent's session log; reproducible by running
the same script).

A v0.2-conforming compiler must reject the same source per
`:82`.

## Impact

Three downstream problems:

1. **Phase 1 regression corpus is the v0.2 regression oracle.**
   `planning/PHASE_2_PLAN.md` makes the v0.1 corpus the
   regression oracle for every v0.2 PR (`Decisions: 2026-05-01
   — Gate 0 waived`). If any v0.1 corpus case uses one of the
   seven keywords as a counter name, the v0.2 compiler will
   fail it — and the user has no spec text to point at, because
   the spec says both "this v0.1 program is valid in v0.2" and
   "this counter name is forbidden in v0.2".

2. **Operator-written v0.1 deployments do not migrate.** Any
   `.fw` an operator wrote against the v0.1 spec — and the v0.1
   spec explicitly admits arbitrary lowercase identifiers as
   counter names — may use, say, `count return_traffic`
   abbreviated to `count return`. The v0.2 lexer will reject it
   on first compile. The user reads the strict-superset
   paragraph and is misled.

3. **Static analysers / IDE integrations cannot rely on the
   strict-superset framing.** Anyone building a tool that
   "validates with the v0.2 grammar but accepts every v0.1
   program" cannot do so with a single grammar; they need a
   compatibility shim.

## Three possible resolutions

| Resolution | What changes |
|---|---|
| (a) Soften the strict-superset claim. | Replace `:9-12` and `:935-936` with: "v0.2 is a near-superset of v0.1. Every v0.1 program is a valid v0.2 program **except** those that use one of the new v0.2 reserved words (`def`, `elif`, `else`, `for`, `while`, `pass`, `return`) as a counter name. The migration is mechanical: rename the counter." |
| (b) Drop the new reservations except those grammar-required. | `def`, `elif`, `else` are *grammar-required* by Tier 2 syntax — they tokenize as keywords whether or not they're declared reserved. `for`, `while`, `pass`, `return` are NOT used by any v0.2 production; the spec reserves them for forward-compat only. Remove `for`, `while`, `pass`, `return` from the reserved list and accept the v0.3-rename cost. |
| (c) Context-dependent reservation. | Allow the seven words as counter/local/function names but reject them as statement-leading tokens. The lexer disambiguates by context (Lark's `priority` + lookahead can do this). The simplest spelling: `count def` is fine because it follows `count`; bare `def` at statement-position is a function definition. Cost: more analyser complexity. |

(a) is the smallest spec change and the most honest one — the
strict-superset claim is genuinely too strong. (b) is the
operator-friendly fix that preserves the strict-superset
guarantee at the cost of forward-compat. (c) is the
language-implementer's fix that preserves both at the cost of
implementation complexity.

The v0.2 author should pick one and update the spec accordingly.

## Suggested corpus item

A `.pkt` test that locks the chosen resolution. Pre-resolution,
this case is a discriminator: it tells the operator which
reading the implementation picked.

```yaml
program: |
  @xdp(eth0)
  count def if pkt.proto == tcp
  default allow

# Strict-superset reading (b/c above): expected.compiles: true
# Reserved-word reading (a above):     expected.compiles: false
expected:
  compiles: true        # picked once spec resolves
```

## Class

Class B (spec contradiction; spec layer). Two paragraphs in the
spec — the strict-superset claim and the reserved-words list —
give incompatible answers about a concrete v0.1 program. Any
v0.1 corpus case using one of the seven keywords as a counter
name will surface the contradiction the moment `hone regress`
runs against the v0.2 compiler.
