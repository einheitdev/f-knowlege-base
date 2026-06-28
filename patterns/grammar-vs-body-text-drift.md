---
id: pattern/grammar-vs-body-text-drift
type: pattern
protocol: [tcp, udp, icmp, icmp6]
builtins: [geoip, rate_limit]
severity: high
layer: [spec]
pattern_tags: [grammar, ebnf, body-text, spec-inconsistency, tier2, unreachable-error, dead-rule]
status: confirmed
created: 2026-05-01
instance_count: 12
instances:
  - finding/2026-05-01-v02-spec-grammar-rvalue-rejects-pkt-field-contradicts-mybool-eq-tcp-syn
  - finding/2026-05-01-v02-spec-grammar-set-or-range-includes-range-for-ipv6
  - finding/2026-05-01-v02-spec-grammar-operand-overpermissive-for-geoip-call
  - finding/2026-05-01-v02-spec-grammar-value-field-missing-proto-and-tcp-fields
  - finding/2026-05-01-v02-spec-grammar-no-function-call-form-promises-unreachable-errors
  - finding/2026-05-01-v02-spec-grammar-local-field-undefined-nonterminal
  - finding/2026-05-01-v02-spec-grammar-cc-code-upper-letter-with-space-illformed
  - finding/2026-05-01-v02-spec-tier2-rate-limit-condition-not-in-grammar
  - finding/2026-05-01-v02-spec-tier2-locals-not-in-condition-grammar
  - finding/2026-05-01-v02-spec-tier2-list-typed-locals-grammar-permits-but-no-type
  - finding/2026-05-01-v02-spec-tier2-scalar-expr-missing-proto-keyword
  - finding/2026-05-01-v02-spec-rate-limit-per-src-ip6-error-message-unreachable-via-grammar
---

# grammar-vs-body-text-drift

## Description

The spec contains both an EBNF grammar reference (the parser
contract) and prose rules / type tables / compile-error tables
(the analyzer and runtime contract). The two drift apart:

- **Grammar too narrow.** The grammar omits a production the body
  text relies on. Example: `rvalue` admits `operand | identifier`
  but not `value_field`, while the type rules say `my_bool ==
  pkt.tcp.syn` is well-typed; the parser rejects the well-typed
  form. Example: `condition` does not admit Tier 2 locals, but every
  Tier 2 example uses `if internal:`. Example: `value_field` lists
  the six v4/v6 address-and-port fields but omits `pkt.proto` and
  `pkt.tcp.syn`/`ack`, while the type-rules table treats those as
  legal RHS values for assignment.
- **Grammar too wide.** The grammar admits a production the body
  text forbids, with no analyser pass to catch it. Example: `set_or_range`
  admits `range` for any LHS type, but the prose forbids IPv6
  ranges. Example: the geoip-call grammar admits an `operand` over
  the wildcard, when the body text restricts arguments to bare
  country codes.
- **Promised analyzer error preempted by parser.** The compile-error
  table promises a specific string for an input shape, but the
  grammar rejects that input at the parser layer with "unexpected
  token". Example: `error: rate_limit per= must be src_ip, ...`
  never fires because `rl_field` is a four-element terminal
  alternation. Example: `error: unknown function '<name>'` never
  fires because no general call production exists.
- **Grammar-typo class.** EBNF productions with malformed terminals
  (`upper letter with space` instead of `upper-letter`) that no
  parser implementation can read literally.

The downstream effect is that two implementers reading the spec
land on different parser surfaces. The strict implementer follows
the grammar verbatim and rejects programs the prose blesses; the
liberal implementer extends the grammar to match the prose and ends
up with a surface the spec does not document. Either way, hone's
oracle disagrees on accept/reject for the same source.

## Check Strategy

1. **Round-trip every grammar production against every prose
   example** that exercises that production. A `.pkt` per example
   with `expected.compiles: true` is the lockable form.
2. **Round-trip every compile-error table row against the grammar.**
   For each promised error string, construct the minimum input
   shape that should fire it. If the parser emits a different error
   first (typically `unexpected token`), the row is unreachable —
   either the grammar must reject earlier and the row should be
   deleted, or the grammar must admit the input so the analyser can
   reach the row.
3. **For every Tier 2 type-rule entry, trace which grammar
   production carries the type.** If `pkt.tcp.syn` is a `bool` per
   the type rules but no grammar production carries it as an
   `rvalue`, the rule is unreachable.
4. **Diff the grammar against the v0.1 grammar** to confirm every
   v0.2 type/operator/keyword extension has a matching grammar
   delta. Missing deltas are the most common cause of this pattern.
5. **Search for inline EBNF terminals with unusual whitespace or
   case** (`upper letter`, `lower-letter`, etc.). They are typos
   that most readers miss because the surrounding prose disambiguates.
6. **Use the `fwl-test-agent` synthetic-corpus generator** to
   produce one program per grammar production and one program per
   prose example, then check that every program either compiles or
   fires the documented error string. Mismatches are the
   instances of this pattern.

## Known Instances

Grammar too narrow:

- `finding/2026-05-01-v02-spec-grammar-rvalue-rejects-pkt-field-contradicts-mybool-eq-tcp-syn`
  — `rvalue = operand | identifier` doesn't admit `value_field`.
- `finding/2026-05-01-v02-spec-grammar-value-field-missing-proto-and-tcp-fields`
  — `value_field` lacks `pkt.proto`, `pkt.tcp.syn`, `pkt.tcp.ack`.
- `finding/2026-05-01-v02-spec-grammar-local-field-undefined-nonterminal`
  — `expression` references undefined `local_field`.
- `finding/2026-05-01-v02-spec-tier2-rate-limit-condition-not-in-grammar`
  — `rate_limit_call` introduced as bool primary, but `condition` /
  `primary` don't admit it.
- `finding/2026-05-01-v02-spec-tier2-locals-not-in-condition-grammar`
  — `condition`'s extension list omits local-identifier reads.
- `finding/2026-05-01-v02-spec-tier2-scalar-expr-missing-proto-keyword`
  — `proto`-typed locals are one-way readable from `pkt.proto`,
  never assignable to a comparison RHS.

Grammar too wide:

- `finding/2026-05-01-v02-spec-grammar-set-or-range-includes-range-for-ipv6`
  — `set_or_range` admits `range` over IPv6 LHS; prose forbids.
- `finding/2026-05-01-v02-spec-grammar-operand-overpermissive-for-geoip-call`
  — `geoip_call` operand accepts any `operand`, not just `cc_code`.
- `finding/2026-05-01-v02-spec-tier2-list-typed-locals-grammar-permits-but-no-type`
  — `expression` admits `list`/`cidr`/`cidr_list` as RHS of local
  assignment; type table has no list-shaped local type.

Unreachable error / dead surface:

- `finding/2026-05-01-v02-spec-grammar-no-function-call-form-promises-unreachable-errors`
  — the spec promises `error: unknown function '<name>'` and
  recursion errors, but no general call production exists.
- `finding/2026-05-01-v02-spec-rate-limit-per-src-ip6-error-message-unreachable-via-grammar`
  — `rl_field` is a four-element terminal; the parser preempts the
  analyzer's `error: rate_limit per= must be ...` string.

Grammar typo:

- `finding/2026-05-01-v02-spec-grammar-cc-code-upper-letter-with-space-illformed`
  — `cc_code = upper letter , upper letter` is malformed EBNF.

## Where to Look Next

- **Every compile-error table row, audited for parser-preemption.**
  This is mechanical: enumerate each row, write the input that
  should trigger it, run through a Lark grammar implementing the
  EBNF verbatim, see which rows the parser preempts. ~30 rows
  unaudited.
- **Tier 2 condition surface** is undertested. The grammar's
  `condition` production is a v0.1 import; only ad-hoc bullet-point
  extensions are documented. A formal v0.2 `condition` rewrite is
  almost certainly needed; until it exists, every Tier 2 example
  that uses `if`/`elif`/`not`/`and`/`or` with new operand types is a
  candidate for grammar drift.
- **Geoip grammar.** The geoip section has both `geoip_call` and
  `cc_code` productions plus prose rules about call-site identity;
  the manifest format is described prose-only. This is fertile
  ground for the same pattern — the grammar may not constrain to
  what the manifest expects.
- **PKT_V02_SPEC.md `.pkt` schema.** The PKT spec describes the
  `.pkt` YAML schema in prose with example blocks; it has no
  formal grammar at all. Body-text-only specs are even more prone
  to drift between rules and examples. Audit the `state.rate_limit`
  / `state.geoip_data` schemas in particular.
- **Lexical layer.** The reserved-word list, the proto-keyword
  list, and the IPv6 canonicalization rules are stated three
  different ways (Lexical Structure, IPv6 Fields, the per-keyword
  comment). Cross-section drift is high; the canonicalization-rule-4
  finding is a confirmed instance, but the canonicalization audit
  isn't complete (RFC 5952 §4 rules 1–3 still untouched).
- **`set_or_range` and `list` productions.** These shipped as
  shared productions for IPv4 and IPv6 in v0.2; one finding already
  caught the IPv6-range bug. The complementary finding "list of v6
  literals must allow `::/0`" hasn't been written but the same
  surface is likely affected.
