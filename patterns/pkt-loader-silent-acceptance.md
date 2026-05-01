---
id: pattern/pkt-loader-silent-acceptance
type: pattern
protocol: [tcp, udp, icmp, icmp6]
builtins: [rate_limit, geoip]
severity: medium
layer: [loader, runner]
pattern_tags: [pkt-loader-validation, oracle-skew, silent-acceptance, test-harness, spec-conformance]
status: hypothesis
created: 2026-05-01
instance_count: 4
instances:
  - finding/2026-04-25-pkt-loader-state-rule-idx-not-validated
  - finding/2026-04-25-pkt-loader-unknown-builder-fields
  - finding/2026-05-01-pkt-loader-accepts-noncanonical-ipv6-in-builder-fields
  - finding/2026-05-01-bpf-runner-skips-geoip-map-population
---

# pkt-loader-silent-acceptance

## Description

The `.pkt` loader (`fwl/fwl/pkt.py`) and the runner (`fwl/fwl/runner.py`)
silently accept inputs the PKT spec mandates rejecting. Each
instance erodes the verification methodology in a slightly different
way:

- **Unknown builder field accepted** — `icmp(src_port=12345)` adds
  `src_port: 12345` to the decoded dict for an ICMP frame whose wire
  bytes have no L4 ports. Interpreter and BPF runtime then read
  *different* fields for the same packet (interpreter reads decoded
  dict, BPF reads bytes), producing Class A oracle skew on rate-limit
  bucket keys.
- **state.rate_limit references a non-existent or no-modifier rule
  index** — silently ignored. Test "passes" with the bucket count
  the author wrote silently discarded.
- **Non-canonical IPv6 string in builder field** — accepted by
  `_ipv6_to_bytes`, which uses Python's lenient `ipaddress` module.
  The FWL parser enforces RFC 5952 canonicality on `.fw` literals;
  the .pkt loader doesn't, so a test can document a literal the .fw
  source can't even contain.
- **Test-harness skips populating BPF maps** — `_build_map_init`
  translates `state.rate_limit` to BPF map writes but never touches
  `case.geoip_data`, so every BPF run sees an empty geoip trie.
  Without root the BPF oracle is `skip`'d and the divergence stays
  hidden; with root, ten geoip cases fail.

The unifying root cause: **the .pkt schema is enforced by ad-hoc
parsing rather than by validation against a declared schema**. Each
field on `PktCase` has its own validation path, and any path that
doesn't exist or doesn't enforce the spec accepts arbitrary input.

The downstream effect is **oracle skew of the wrong sign**: the
test infrastructure makes the corpus *look* green when it's
exercising the wrong test (or no test at all). The methodology
treats `.pkt` agreement as the verification authority; if the .pkt
loader can produce two oracles seeing different inputs, agreement
loses meaning.

This is a hypothesis pattern because N=4 is below the
high-confidence threshold; but the shape is mechanically clear and
each instance points to a missing validation hook. The runner
finding overlaps with `helper-skips-tier2-function-body` (both are
"helper missed a v0.2 input source") but is included here because
its surface symptom is harness-level silent acceptance.

## Check Strategy

1. **Audit `pkt.load` against `PKT_V01_SPEC.md` and `PKT_V02_SPEC.md`
   error tables.** Every row of the form "load-time error: ..."
   should be reachable from a `.pkt` input. Synthetic .pkt files
   that should fail to load but currently load are findings.
2. **Audit each builder constructor (`tcp`, `udp`, `icmp`, `tcp6`,
   `udp6`, `icmp6`) against the documented field table.** Unknown
   fields, missing required fields, type mismatches must each fire
   the prescribed error.
3. **Cross-check FWL-source validation against PKT-builder
   validation for shared concepts.** RFC 5952 canonicality, CIDR
   prefix bounds, country-code casing — anywhere the parser
   validates a syntactic form, the .pkt builder for the same concept
   should validate the same way (and ideally call the same helper).
4. **Audit `_build_map_init` and other harness helpers for every
   `case.<field>` they read.** Every PktCase field must flow into
   both oracle paths or the harness silently produces oracle skew.
5. **For every existing `.pkt` corpus case, deliberately mutate the
   builder to inject an invalid field, and confirm load-time
   rejection.** Cases that currently pass with the mutation are
   findings.
6. **Run the corpus under sudo on a dev VM to expose `bpf=skip`-masked
   divergences.** The geoip-runner finding only surfaced under root;
   anything else hidden by `skip` is a candidate.

## Known Instances

- `finding/2026-04-25-pkt-loader-state-rule-idx-not-validated` —
  silent acceptance of `state.rate_limit` referencing missing or
  modifier-less rule indices.
- `finding/2026-04-25-pkt-loader-unknown-builder-fields` —
  `parse_builder` accepts `name=value` pairs without consulting
  per-constructor field tables; latent Class A on icmp.
- `finding/2026-05-01-pkt-loader-accepts-noncanonical-ipv6-in-builder-fields`
  — `_ipv6_to_bytes` accepts every RFC 4291-valid form, not just
  RFC 5952-canonical.
- `finding/2026-05-01-bpf-runner-skips-geoip-map-population` —
  `_build_map_init` only knows about `state.rate_limit`;
  `case.geoip_data` is consumed by the interpreter but never wired
  into the BPF map.

## Where to Look Next

- **`PKT_V02_SPEC.md` schema entries that don't have a v0.1
  analogue** — `tcp6`/`udp6`/`icmp6` builders, `geoip_data` block,
  the v0.2-extended `expected:` keys (`compiles`, `loads`,
  `compile_error_pattern`). Each was added without the
  cross-validation discipline of `parser._canonical_ipv6`.
- **CIDR prefix-length validation in the v6 builders.** v4 builders
  check `0 <= prefix <= 32`; v6 builders don't necessarily check
  `0 <= prefix <= 128` consistently. The .pkt corpus hasn't been
  fuzzed at the v6 prefix boundary.
- **`expected.bpf_action` vs `expected.action`** — the .pkt schema
  has accreted multiple keys for the same concept across versions.
  An author who writes `bpf_action` when the schema expects `action`
  may get silent acceptance of a never-checked expectation.
- **`expected.compiles: false` cases that don't include
  `compile_error_pattern`** — silently accept any compile error,
  including the wrong one. The empty-body / `pass` finding shows
  this matters.
- **`expected.counter_changes`** is documented in the spec but the
  runner doesn't drain the per-CPU counter map (per the
  rate-limit-inversion finding's "What was NOT tested" section).
  Any `.pkt` that asserts counter deltas is silently accepted with
  no actual check.
- **`truncate_to`** is spec-only — the runner skips with a clear
  reason. But the skip is at the BPF oracle; the interpreter still
  accepts the case. This is the kind of half-implementation that
  produces oracle skew (interpreter agrees with itself, BPF skips,
  corpus appears green).
- **The fd daemon's bundle-attach path.** The .pkt runner is the
  test analogue. The runner-skips-geoip finding pinned the
  test-side gap; the daemon-side hasn't been audited and may have
  the same shape.
- **Cross-runner: `hone fuzz` adversarial inputs.** The fuzzer
  generates `.pkt`s; if it produces inputs the loader silently
  accepts, hone marks them green and moves on. Audit the fuzzer's
  output against the spec error tables — anything the loader
  accepts that the spec rejects is a fuzzer-side false negative.
