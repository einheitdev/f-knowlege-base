---
id: finding/2026-05-01-v02-spec-icmp6-counter-name-breaks-strict-superset-carveout
type: finding
protocol: [icmp6]
builtins: []
severity: medium
layer: spec
pattern_tags: [spec-contradiction, strict-superset, lexical, reserved-words, counter-names, v01-compat, icmp6]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related:
  - finding/2026-05-01-v02-spec-new-reserved-words-break-strict-superset-claim
---

# 2026-05-01-v02-spec-icmp6-counter-name-breaks-strict-superset-carveout

## Summary

The v0.2 strict-superset claim at `FWL_V02_SPEC.md:9` enumerates
exactly **three** new reservations relative to v0.1 — `def`, `elif`,
and `else` — and pledges that every other v0.1 program compiles
unchanged under v0.2. The Lexical Structure section at
`FWL_V02_SPEC.md:64-79` then introduces a **fourth** new reservation
the carve-out never mentions: `icmp6` (`:73`, "*Proto keywords:*
`tcp`, `udp`, `icmp`, `icmp6`").

`tcp`, `udp`, `icmp` were already proto keywords (and therefore
already excluded from identifier positions) in v0.1, so reserving
those is a no-op for v0.1 programs. `icmp6` was **not** a v0.1
keyword: it is just a `[a-z_][a-z0-9_]*` identifier per
`FWL_V01_SPEC.md:39`, and the v0.1 implementation accepts it
verbatim as a counter name, an interface name, or anywhere else
v0.1 admits a generic identifier.

A purely-v0.1 program that uses `icmp6` as a counter name therefore
compiles cleanly in v0.1 but fails in v0.2 with the
reserved-keyword error from `:79`. The strict-superset carve-out at
`:9` is wrong by exactly one entry.

Class B (spec contradiction; spec layer; pre-implementation, fix is
editorial). Will surface as a Class-A regression in the
`hone regress` v0.1-corpus oracle the moment a v0.1 case using
`icmp6` as a counter name (or interface name) is run through the
v0.2 toolchain.

## Demonstration

The shipped v0.1 compiler (`f-hone/.venv/bin/fwl`, current master)
accepts `icmp6` as a counter name with no diagnostic:

```sh
$ cat > /tmp/test_count_icmp6.fw <<'EOF'
@xdp(eth0)

count icmp6 if pkt.proto == icmp
allow
EOF

$ f-hone/.venv/bin/fwl compile /tmp/test_count_icmp6.fw \
    -o /tmp/test_count_icmp6.bpf.c
$ echo $?
0
```

For comparison, the same harness reports an explicit parser error
for `count tcp`:

```
error: 3:7: Unexpected token Token('PROTO_KEYWORD', 'tcp') at line 3, column 7.
Expected one of:
        * IDENTIFIER
```

The asymmetry is exactly what `:73` has just promised to remove —
v0.2 elevates `icmp6` to the same PROTO_KEYWORD class as `tcp`.

## The two paragraphs that disagree

### `:9` — the carve-out lists exactly three keywords

> v0.2 is a near-superset of [v0.1](FWL_V01_SPEC.md): every v0.1
> program is a valid v0.2 program with identical packet semantics,
> **except** v0.1 programs that use one of the three Tier 2 syntactic
> keywords (`def`, `elif`, `else`) as a counter name. Such programs
> require a one-time rename of the offending counter — a syntactic
> adjustment, not a semantic change. The three keywords are reserved
> at the lexical layer because Tier 2 needs them as statement-leading
> tokens; **no other v0.2 reservation breaks v0.1 backward
> compatibility.**

The closing sentence is unambiguous: "no other v0.2 reservation
breaks v0.1 backward compatibility."

### `:64-79` — the reserved-words list adds `icmp6`

> The following identifiers are reserved and may not appear as a
> Tier 2 function name, a Tier 2 local name, or a counter name.
> Lexically they tokenize as their respective keyword classes
> regardless of context.
>
> ...
> - *Proto keywords:* `tcp`, `udp`, `icmp`, `icmp6`.

`icmp6` is in the list. `:64`'s sentence "tokenize as their
respective keyword classes regardless of context" means a v0.1
source with `count icmp6` no longer parses under v0.2 — the lexer
emits PROTO_KEYWORD for `icmp6`, the parser's `count IDENTIFIER`
rule fails, and the analyser raises the reserved-word error from
`:79`.

### `:46` — the carve-out paragraph

> A v0.1 program runs unchanged under the v0.2 compiler — modulo the
> `def`/`elif`/`else`-as-counter-name carve-out described above. The
> [Phase 1 corpus](../../f-knowlege-base/corpus/) is the regression
> oracle for this guarantee; any corpus case that names a counter
> `def`, `elif`, or `else` requires a one-time rename when the corpus
> is migrated to v0.2.

`:46` repeats the three-keyword carve-out as the migration rule.
A v0.1 corpus case using `count icmp6` requires a one-time rename
too, but `:46` doesn't list it. An author writing the migration
script that walks the corpus is told to rewrite three keywords and
will miss the fourth.

## Surfaces affected, beyond counter names

`:64`'s "tokenize as keyword regardless of context" rule means
`icmp6` is forbidden in **every** v0.1 identifier position, not
just counter names:

| v0.1 surface | v0.1 program | v0.2 status |
|---|---|---|
| Counter name | `count icmp6 if pkt.proto == icmp` | reserved-word error |
| Interface name | `@xdp(icmp6)` | parser error: PROTO_KEYWORD where IDENTIFIER expected |
| (v0.2-only) Tier 2 local | `icmp6 = pkt.proto` | reserved-word error |
| (v0.2-only) Tier 2 function name | `def icmp6(pkt): ...` | reserved-word error |

The first two are v0.1 surfaces; the program shapes existed and were
admissible in v0.1. The last two are v0.2-only surfaces and don't
participate in the strict-superset claim.

## Why this matters for Gate 1

Gate 1 of `planning/PHASE_2_PLAN.md` requires zero open `layer: spec`
findings. This contradiction is one such — `:9` and `:73`
contradict each other on a concrete, demonstrable v0.1 program.

Knock-on impact on Gate 2 (corpus migration):

1. The carve-out paragraph at `:46` instructs the corpus migrator to
   rename three counter names. Anyone writing the migration grep
   misses the fourth (`icmp6`) and ships a "migration complete"
   patch that breaks on the first run of `hone regress` against the
   migrated corpus.
2. An operator running `fwl compile` against an existing v0.1 `.fw`
   that names a counter `icmp6` is told by the strict-superset
   paragraph that the program will compile unchanged. It does not.

`icmp6` is also the *most likely* of the new keywords to be a
counter name in operator-written v0.1 programs — it is a natural
descriptor for an ICMPv6 traffic counter (`count icmp6 if pkt.proto
== icmp`). The other three new reservations (`def`/`elif`/`else`)
are uncommon counter names by virtue of being statement keywords in
unrelated languages; `icmp6` is the one a network engineer is most
likely to have written.

## Proposed fix

Two clean options:

**(a) Update the carve-out at `:9` and `:46` to list four keywords.**

Edit `:9`:

> ...**except** v0.1 programs that use one of the four new v0.2
> reserved words (`def`, `elif`, `else`, `icmp6`) in an identifier
> position. Such programs require a one-time rename — a syntactic
> adjustment, not a semantic change. The first three are reserved
> because Tier 2 grammar needs them as statement-leading tokens;
> `icmp6` is reserved because v0.2 promotes it to a proto keyword.

Edit `:46` symmetrically — rename the three-keyword phrasing to a
four-keyword one. (The carve-out for `def`/`elif`/`else` was
narrowed in a prior fix to drop `for`/`while`/`pass`/`return`; this
edit re-broadens it to include `icmp6`.)

**(b) Drop `icmp6` from the lexical reserved set and detect it
contextually.**

`:73`'s reservation of `icmp6` is for the lexer to identify it as a
proto-keyword token. An alternative is to keep `icmp6` lexing as
IDENTIFIER and have the proto-equality analyser pass treat the
literal `icmp6` (in proto-keyword position) as the proto-keyword.
This preserves v0.1 counter-name compatibility but adds analyser
complexity and breaks the symmetry with `tcp`/`udp`/`icmp` (which
are already PROTO_KEYWORD-tokenized in v0.1 by the implementation).

**(a) is the smaller spec change and matches the existing
treatment of `tcp`/`udp`/`icmp`** (which are PROTO_KEYWORD-tokenized
already, so v0.1 programs using them as counter names are *already*
rejected by the v0.1 lexer — the reservation is just being made
explicit in v0.2). For `icmp6` this is genuinely new: v0.1 admits
it as a counter name; v0.2 should not silently break that
compatibility while the strict-superset paragraph claims otherwise.

## Suggested corpus item

A `.pkt` test that locks the chosen resolution:

```yaml
program: |
  @xdp(eth0)
  count icmp6 if pkt.proto == icmp
  allow
# Strict-superset reading (option b above):  expected.compiles: true
# Reserved-word reading (option a above):    expected.compiles: false
expected:
  compiles: false   # picked once spec resolves
```

## Class

Class B (spec self-contradiction). Spec layer.
