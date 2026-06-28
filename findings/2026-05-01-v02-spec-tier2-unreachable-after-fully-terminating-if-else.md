---
id: finding/2026-05-01-v02-spec-tier2-unreachable-after-fully-terminating-if-else
type: finding
protocol: [tcp, udp]
builtins: []
severity: medium
layer: spec
pattern_tags: [spec-ambiguity, tier2, reachability, if-else, terminal-action, control-flow]
status: fixed
source_file: docs/FWL_V02_SPEC.md
created: 2026-05-01
related: [finding/2026-05-01-v02-spec-tier2-stmts-after-terminal-example-violates-proto-guard, finding/2026-05-01-v02-spec-tier2-empty-body-example-uses-pass-which-fires-different-error]
---

# 2026-05-01-v02-spec-tier2-unreachable-after-fully-terminating-if-else

## Summary

The Tier 2 unreachable-statement rule (`FWL_V02_SPEC.md:666-682`,
table row at `:730`) defines exactly two cases:

1. *Reachable.* A statement after an `if` whose body terminates but
   whose else-branch (implicit fall-through) does not. The trailing
   statement is reachable on the if-false path. (The example.)
2. *Unreachable.* A statement directly after a bare terminal action
   (`drop\nallow` — the second line is `error: unreachable statement
   after terminal action 'drop'`).

A third case is left undefined: the statement after an `if/else`
(or `if/elif/else` chain) where **every** branch ends in a terminal
(`allow` or `drop`):

```python
def firewall(pkt):
  if pkt.proto == tcp:
    drop
  else:
    drop
  allow            # spec doesn't say
```

Reading 1 (textual): the previous statement at the function-body
indent level is `else_clause`, not a bare terminal. The textual
"statement after terminal action" rule from `:730` does not match,
so the program compiles. The `allow` is dead code that the emitter
silently never reaches.

Reading 2 (semantic): the spec at `:682` says "Reachability is
computed per branch." Every branch through the if/else terminates,
so no path reaches `allow`. Per the unreachable rule, this should
be a compile error.

The two readings produce different analyser verdicts on the same
source. The Compile-errors table row at `:730` is keyed on the
single-statement form, so a literal-textual analyser accepts the
fully-branched form.

Class B (spec ambiguity at the analyser layer) with a downstream
Class A surface: under reading 1 the BPF emitter generates dead
code that the verifier may accept or reject (verifier rules on
unreachable code are version-dependent), and the AST interpreter
silently never visits the dead statement. Under reading 2 the
program never compiles, so the discrepancy never reaches runtime.
A `.pkt` corpus author has no spec text to write `compiles: true`
or `compiles: false` against.

## What the spec says (and doesn't)

### The reachable example (`:666-674`)

> ```python
> if pkt.proto == tcp:
>   if pkt.tcp.syn:
>     drop
> allow
> ```
>
> is fine — the trailing `allow` is reachable whenever either the
> outer or the inner `if` does not match.

This is the explicit positive case: an `if` whose body terminates
on the true path, but whose implicit (no `else`) fall-through path
does not. `allow` is reachable on the implicit fall-through.

### The unreachable example (`:677-682`)

> ```python
> drop
> allow
> ```
>
> (the second statement is unconditionally dead) is a compile
> error: `error: unreachable statement after terminal action
> 'drop'`. Reachability is computed per branch; the inner
> `if pkt.tcp.syn:` lives inside the proto-guarded block per the
> dominator rule.

This is the explicit negative case: a bare terminal followed by a
statement, where there is no branching to consider.

### The Compile-errors row (`:730`)

> | Unreachable statement after terminal | `error: unreachable
> statement after terminal action '<allow|drop>'` |

The row's wording — "after terminal action" — names a *single
action* as the predecessor. It does not name an `if/else` block
whose every branch ends in a terminal.

### The "reachability is computed per branch" sentence (`:682`)

> Reachability is computed per branch; the inner `if pkt.tcp.syn:`
> lives inside the proto-guarded block per the dominator rule.

This is a single sentence shoehorned into the negative example's
explanation. It hints at semantic reachability analysis (every
branch matters, not just the lexically previous statement). But
the spec does not develop the rule; the only concrete examples are
the two simple cases above.

The dominator rule at `:626-633` is polarity- and disjunction-
aware. The reachability rule's "per branch" wording is consistent
with that style, but the unreachable-statement table row is keyed
textually. The two are not reconciled.

## Concrete cases the spec leaves undefined

### Case A — if/else, both arms `drop`

```python
@xdp(eth0)
def firewall(pkt):
  if pkt.proto == tcp:
    drop
  else:
    drop
  allow                 # what?
```

Every branch terminates with `drop`. The trailing `allow` is
unreached on every packet.

- Reading 1: compiles; `allow` is dead but legal.
- Reading 2: compile error per "Reachability is computed per branch".

### Case B — if/elif/else, all three arms terminate

```python
@xdp(eth0)
def firewall(pkt):
  if pkt.proto == tcp:
    drop
  elif pkt.proto == udp:
    drop
  else:
    drop
  count fell_through    # unreached on every packet
```

`count fell_through` is unreached on every packet.

- Reading 1: compiles; `count fell_through` is dead but legal.
- Reading 2: compile error.

### Case C — nested if/else where one branch's inner if/else fully terminates

```python
@xdp(eth0)
def firewall(pkt):
  if pkt.proto == tcp:
    if pkt.dst_port == 22:
      drop
    else:
      drop
    log                 # unreached because both inner arms terminate
  allow
```

The inner if/else fully terminates; `log` after it is unreached.
But `allow` after the outer `if` is reached on the if-false path.

- Reading 1: compiles; `log` is dead.
- Reading 2: compile error on `log`.

### Case D — if without else, both arms terminate via inner blocks

This case is actually the spec's positive example modified:

```python
if pkt.proto == tcp:
  drop                  # terminates
allow                   # reached when proto != tcp
```

This is the spec's explicit case 1 — `allow` is reachable. (Listed
for contrast.)

## Why this matters

Three concrete failure modes:

1. **Implementation drift.** A literal-textual analyser (Reading 1)
   accepts Cases A, B, C; a semantic-reachability analyser (Reading
   2) rejects them. Two conformant FWL compilers will land on
   different verdicts. The user has no spec text to point at.

2. **BPF verifier interaction.** The XDP verifier's tolerance for
   unreachable code is kernel-version dependent. Older verifiers
   reject programs with truly-unreachable instructions; newer ones
   may accept them but emit warnings. Under Reading 1, the FWL
   compiler emits dead instructions that reach the verifier;
   whether the BPF program loads becomes a kernel detail. Under
   Reading 2, the FWL compiler stops the program at analysis time
   and the user sees a clean error message naming their source
   line.

3. **Interpreter / BPF oracle drift on Cases A/B/C.** The AST
   interpreter under Reading 1 silently never executes the
   trailing statement (it's never reached at runtime). The BPF
   emitter under Reading 1 emits the trailing statement's bytecode
   and relies on the verifier to skip it. If the bytecode has any
   side effect the verifier doesn't elide (a counter increment, a
   ring-buffer log event), the emitted program disagrees with the
   interpreter. Hone's three-oracle check fires.

4. **Compose with the existing `proto-guard-violating example`
   finding.** The fixed example at `:666-674` (per
   `finding/.../tier2-stmts-after-terminal-example-violates-proto-
   guard`) shows an `if` with a single terminal body and an
   implicit fall-through. It does not show the if/else case. The
   reachability rule's "per branch" claim is unexercised by the
   spec's worked examples.

## Three resolutions

| Resolution | What changes |
|---|---|
| (a) Promote the textual rule to a semantic rule. | Replace `:730`'s table row with a precise definition: "A statement is unreachable if every control-flow path from the function entry that could reach it terminates first. Specifically: (1) any statement after a bare terminal action `allow`/`drop`, (2) any statement after an `if`/`elif`/`else` chain where every branch (including the implicit fall-through when no `else` is present) terminates. Both cases produce `error: unreachable statement after terminal action '<allow|drop>'`." Cases A, B, C become compile errors. |
| (b) Restrict the rule to the textual case only. | Edit `:682` to drop the "Reachability is computed per branch" sentence (which the textual rule does not actually implement) and explicitly note: "The unreachable check considers only the lexically-immediate predecessor; an `if`/`else` whose every branch terminates does NOT make the next statement unreachable." Programs A, B, C compile; the trailing statements are dead but legal. The corpus must include them as `compiles: true; expected: <whatever-was-the-pre-fall-through-action>`. |
| (c) Add the if/else case as a separate compile-error row. | Keep the existing single-statement row, add a second row: "Unreachable statement after `if`/`elif`/`else` chain where every branch terminates | `error: unreachable statement after fully-terminating if/elif/else block`". This is the analyser-friendliest answer; the existing dominator-style polarity-aware control-flow walk already computes the per-branch termination state. |

(a) and (c) produce the same accept/reject set; (a) is the smaller
spec change. (b) is the lowest-friction for users (no new errors
to surprise them) but leaves the BPF-verifier interaction
unresolved.

## Proposed fix (option a)

Edit `FWL_V02_SPEC.md:730` (Compile-errors table row) to:

> | Unreachable statement, where unreachable means every control-
> flow path from the function entry that could reach the statement
> terminates first (either via a bare `allow`/`drop` predecessor or
> via an `if`/`elif`/`else` chain where every branch terminates) |
> `error: unreachable statement after terminal action '<allow|drop>'` |

Append to the example at `:666-682`:

> A third case the rule covers, by extension of the "per branch"
> analysis: a statement after an `if`/`elif`/`else` chain where
> every branch (and the implicit fall-through when no `else` is
> written) ends in a terminal action.
>
> ```python
> if pkt.proto == tcp:
>   drop
> else:
>   drop
> allow                  # error: unreachable
> ```
>
> Both arms of the `if/else` terminate, so `allow` is unreached on
> every packet. Same error message as the bare-terminal case.

## Suggested corpus item

```yaml
# corpus/from_hunt/tier2-unreachable-after-full-if-else.pkt
program: |
  @xdp(eth0)
  def firewall(pkt):
    if pkt.proto == tcp:
      drop
    else:
      drop
    allow

description: |
  Both arms of the if/else terminate.
  Reading 1 (textual): compiles; `allow` is dead.
  Reading 2 (semantic): compile error.

expected:
  compiles: false
  compile_error_pattern: "unreachable statement after.*'drop'"
```

A second discriminator (Case C) for the nested form:

```yaml
program: |
  @xdp(eth0)
  def firewall(pkt):
    if pkt.proto == tcp:
      if pkt.dst_port == 22:
        drop
      else:
        drop
      log                # unreached: both inner arms terminate
    allow                # reached: outer-if-false path

expected:
  compiles: false
  compile_error_pattern: "unreachable statement after.*'drop'"
```

## Class

Class B at the spec layer (the unreachable rule's "per branch"
clause is hinted but not formalized). Downstream Class A risk: the
BPF emitter emits dead bytecode that the AST interpreter never
visits; the verifier's tolerance for unreachable instructions is
kernel-version-dependent; the three-oracle check disagrees on
programs that exercise the gap.
