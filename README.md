# f-knowledge-base

The institutional memory of the FWL/f security harness. Every confirmed
bug becomes a finding here. Every hypothesis tested-and-falsified
becomes a miss. Every recurring bug class becomes a pattern. Every
confirmed bug grows the regression corpus monotonically — tests never
get removed.

This repo is data. No code. The harness ([`hone`](https://github.com/KRuskowski/hone))
reads from it and writes to it; the public security website renders
from it.

## Layout

```
findings/    Confirmed bugs. One markdown file per bug.
misses/      Hypotheses that were tested and didn't pan out.
             Recorded so future agent rounds don't waste tokens
             reproposing them.
patterns/    Abstracted bug classes. Emerge from clustering N+
             related findings. Tell the harness where to look next.
corpus/      Regression test cases (.pkt files). Seeded from FWL's
             own corpus; grows whenever a finding produces a PoC.
triage/      Human decisions on findings.
  fixed/         findings whose bug has been fixed
  false-positive/ findings the human ruled invalid
  wontfix/       acknowledged but deliberate
  duplicate/     same root cause as another finding
stats/       Generated reports — bugs/month, cost/finding, coverage.
meta/        Agent self-critique output, scheduler tuning notes.
hooks/       Git hooks (post-commit indexer trigger, etc.).
```

## File format

Findings, misses, and patterns are markdown with a YAML frontmatter
block. The frontmatter fields drive Solr indexing; the body is what
humans (and LLMs) read.

```markdown
---
id: finding/2026-04-25-cidr-prefix-overflow
type: finding
protocol: [ipv4]
builtins: []
severity: medium
layer: compiler
pattern_tags: [range-validation]
status: fixed
source_file: examples/internal_network.fw
created: 2026-04-25
pkt_path: corpus/cidr_prefix_overflow.pkt
---

# 2026-04-25-cidr-prefix-overflow

## Summary
CIDR prefix > 32 crashed the parser instead of producing a clean
compile error.

## Root Cause
[...]
```

The `harness.knowledge` package in `hone` does the round-trip — both
ends of the harness use the same dataclasses for read/write so the
on-disk shape stays consistent.

## Seed contents

`corpus/` is seeded from the FWL repo's main test corpus (62 cases
covering @xdp/allow, proto matching, multi-rule + default, IP/port
fields, rate_limit, log+count, edge cases). These form the floor —
any harness run in this kb starts above this baseline.

`findings/`, `misses/`, `patterns/` start empty. They populate as
`hone fuzz` and `hone hunt` discover bugs.
