# Restructuring PR #56 into independently-mergeable units

Status: working note for Daniel and Michael. Analysis + a first delivered slice.
Plain register; every "the code does X" is checked against a line or a test run.

PR [#56](https://github.com/MichaelOwenDyer/petrivet/pull/56) bundles ~15k lines
across four very different kinds of change. This note separates them, states the
end-user value of each, and proposes an order in which each can be evaluated and
merged on its own. Nothing here is deprecated; the heavier pieces are
re-substantiated as smaller, self-justifying units rather than dropped.

---

## 0. The finding that reframes everything: `origin/main` is red

Before any new feature, the shared baseline (`origin/main`, commit `fdcf7fb`)
fails **6 of its own library tests**, and the broken behavior reaches the
README's headline features:

```
cargo test -p petrivet --lib   →   103 passed; 6 failed
  api::system::tests::basic_firing                         FAILED  (simulation is inert)
  api::system::tests::into_parts                           FAILED  (same root cause)
  api::system::liveness::tests::s_net_non_sc_mixed_levels  FAILED  (petgraph mirror)
  api::system::liveness::tests::t_net_dead_predecessor_…   FAILED  (petgraph mirror)
  api::system::liveness::tests::t_net_source_transition_l4 FAILED  (liveness WIP — yours)
  api::system::reachability::tests::general_net_reach…     FAILED  (test fixture — yours)
```

This is the most ingestible possible framing of the work: **most of PR #56's
bug-fix value is "your main branch is red and these small commits make it
green," one root cause per commit.** The two failures tagged "yours" sit in your
most recent commit (`fdcf7fb`, the optimized-liveness work) and are your call —
see §5.

---

## 1. Delivered now — `pv/correctness-fixes` (3 commits, pushed to the fork)

Branch off `origin/main`; each commit is one root cause with its regression test;
each is independently cherry-pickable (disjoint files). Verified:
`cargo test -p petrivet --lib` → **108 passed; 2 failed** (the two remaining are
the WIP in §5). The branch takes the suite from 6 red to 2 red.

| Commit | One-line value | Greens |
|---|---|---|
| `a82b0bf` Fix inert `fire_unchecked` | Simulation actually mutates the marking; `try_fire`/`fire_any` were silent no-ops. | `basic_firing`, `into_parts` |
| `5fa7e80` Fix petgraph mirror order | `circuits()`, SM/MG liveness, strong-connectivity stop returning hash-order-dependent wrong results. | `s_net_non_sc_mixed_levels`, `t_net_dead_predecessor_propagates` |
| `c5f99e8` Fix m0-deadlock blind spot | A net whose initial marking is itself a deadlock is no longer reported deadlock-free (a false safety verdict). | latent bug; new test |

`fire_unchecked` is the most consequential: `m[p].checked_sub(1).expect(..)`
computed the decremented count and **discarded it**, so the entire firing surface
was inert. The fix assigns the result back. The petgraph fix builds the node-index
arrays in dense-index order instead of `HashMap` iteration order, so the dense↔
NodeIndex correspondence matches how the arrays are later indexed.

These three carry **no policy decisions and no API changes** — safe to merge as-is.

---

## 2. The decomposition (full unit sequence)

Ordered by standalone value × reviewability. Status: ✅ delivered, ◻ specced
(extraction path given), ⚠ your-call (changes behavior/API — needs your decision).

| # | Unit | Value statement | Status |
|---|---|---|---|
| 1 | Fix inert `fire_unchecked` | Simulation works again. | ✅ §1 |
| 2 | Fix petgraph mirror order | Graph-derived analyses stop being wrong/flaky. | ✅ §1 |
| 3 | Fix m0-deadlock blind spot | No false "deadlock-free" on an m0 deadlock. | ✅ §1 |
| 4 | Exact-arithmetic guard on negative reach/cover verdicts | A near-boundary `f64` can no longer mint a false "unreachable/uncoverable" — the correctness headline. | ◻ §3 |
| 5 | PNML strict import | Non-unit-weight arcs and >u32 markings error instead of silently importing a different net. | ⚠ §3 (rejects in-tree fixtures) |
| 6 | Abstain-not-fabricate API | `is_covered_by_s_components → Option<bool>`; drop the structural `Some(false)` deadlock arm. | ⚠ §3 (breaking) |
| 7 | B2 cluster partition + `rank(C)=c−1` | Supplies the cluster count `c` that `class.rs`'s Rank-Theorem doc already names; tested, no decider. | ◻ §3 |
| 8 | M3 decider registry | Per-class dispatch becomes a pluggable seam for #42/#44; reproduces today's cascade exactly. | ◻ §3 |
| 9 | Checkable-witness API (the #45 reframe) | Verify an external/SMT-proposed marking cheaply — the one real end-user story for "certificates." | ◻ §3, §4 |
| 10 | Doctest + `model` module | Make `cargo test --doc` compile and run again. | ◻ §3 (depends on #9) |

Units 4–10 are **not yet rebuilt** here; §3 gives each an extraction path from the
`workflow-2` branch so they can be carved one at a time. They were left un-built
deliberately: shipping 1–3 first earns trust without asking you to accept any
architecture, and keeps each subsequent unit a single reviewable decision.

---

## 3. Extraction paths for the remaining units

All references are to branch `workflow-2` (the original PR head). `git diff
origin/main...workflow-2 -- <path>` shows each unit's slice.

- **Unit 4 — exact guard.** Files: `core/analysis/rational.rs` (exact ℚ),
  `core/analysis/exact_matrix.rs` (marking equation over ℚ), and the wiring in
  `api/system/reachability.rs` / `coverability.rs` / `api/net/boundedness.rs`.
  The principle is small and statable: keep the `f64` LP/ILP path as a *suggester*,
  and rest every **negative** verdict ("unreachable/uncoverable") on an exact
  certificate, escalating to the (terminating) state-space explorer when ℚ cannot
  decide. Honest caveat to carry into the PR: the exact core is RREF-over-ℚ, not
  the fraction-free Bareiss schedule, so at large scale it may be slower than the
  path it guards — a measured follow-on, not a regression on the suggestion.
  This is the single highest-value feature; it should be its own PR.

- **Unit 5 — PNML strict import.** File: `api/pnml/convert.rs` (two new
  `PnmlConversionError` variants: `NonUnitWeightArc`, `MarkingOverflow`).
  ⚠ It rejects 3 in-tree fixtures, so it changes import behavior — your policy to
  set. Advances #34 and the spirit of #57. Present with an explicit
  "reject vs. clamp vs. wait-for-real-weight-support" choice.

- **Unit 6 — abstain-not-fabricate.** Files: `api/net/mod.rs`
  (`is_covered_by_s_components: bool → Option<bool>`), `api/system/deadlock_freedom.rs`
  (remove the `Some(false) if FreeChoice` arm). ⚠ Breaking and a policy stance
  (abstain rather than return a possibly-correct-but-unwitnessed `false`). The
  m0-deadlock fix (unit 3) was deliberately split out of this so the bug fix
  doesn't ride on the policy decision.

- **Unit 7 — B2 cluster.** File: `core/analysis/cluster.rs`. Self-contained;
  registers no decider. Lands the consumption-arc (Desel–Esparza) cluster
  partition and the tested relation `rank(C) = c − 1` (gated as *necessary*, not
  the full equivalence — flagged in-code).

- **Unit 8 — M3 registry.** Files: `api/model/registry.rs` + `registry/*.rs`.
  Default policy reproduces the current cascade exactly (corpus regression +
  order-invariance test). Value is future extensibility, not visible behavior.

- **Unit 9 — checkable-witness API.** Carve from `api/model/`: keep
  `FiringSequenceCert` / `ParikhVectorCert` and a public `check`; **drop** the
  ledger / frontier / interchange-format / firewall accounting (those are thesis
  instrumentation — §4). Reframe as "verify a proposed marking/firing-vector,"
  i.e. issue #45.

- **Unit 10 — doctests + `model`.** `literature.rs:409` imports `crate::model`,
  which does not exist on `origin/main`, so `cargo test --doc` fails to compile.
  Fixing the doc examples therefore depends on unit 9 landing the module (or on
  editing `literature.rs`). Bundle with unit 9.

---

## 4. The certification-value question, in end-user terms

Michael's objection ("how does *certifying* add value to someone using the
library?") is correct as stated, because "certification" is doing two unrelated
jobs in the PR. Separated:

- **Job A — "the answer is checked, not guessed."** This is real, and it is unit 4.
  Today a near-boundary `f64` can mint a false "unreachable," which to a
  practitioner checking a safety property reads as "your bad state can't happen" —
  a system reported safe when it is not. The exact guard removes that class of
  silent wrong answers while keeping the fast path. The value needs no certificate
  vocabulary to state: *petrivet does not give you wrong analysis answers, even
  when the fast path would.*

- **Job B — certificate *objects* returned to the user.** Here the objection
  holds. The machinery (the 1706-line `checkers.rs`, the ledger, the frontier, the
  interchange format) is not on any public call path — the `analyze_*` methods
  still return the old result types and the registry reproduces the cascade. So
  the user sees no certificate today. A certificate earns its place only when it
  answers a question the **user** asks, and there is exactly one such question in
  the repo, in your own words: **issue #45** — guided enumeration to *verify an
  SMT-solver result*. That is the genuine story: an untrusted fast/heuristic/SMT
  producer proposes an answer; petrivet cheaply and exactly **checks** it. That is
  unit 9, and it pairs with #44 (SMT).

Everything beyond Job A and the #45 checker — the trusted-base ledger, the
`f_struct` coverage number, the firewall-as-figure-of-merit, the learned-selection
ladder — measures a property of the *implementation* (how much of the answer space
is proof-backed). That is a thesis question, not an end-user one. **Re-substantiation,
not deprecation:** it moves out of the library PR and into a short proposal
(§6) you own, and the user-facing residue (units 4 and 9) stays.

---

## 5. Hand-back: the two failures that are yours

Both are in commit `fdcf7fb` (your optimized-liveness work). Precise diagnosis so
you can decide, rather than a guess at your theory:

- **`general_net_reachability_fallback`** asserts `net.class() == General`, but the
  fixture it builds (`p0→t0→p1`, `p0→t1→p2`, `p1→t2→p0`, `p2→t2→p0`) is genuinely
  extended-free-choice: the only shared input place `p0` feeds `t0,t1` with equal
  presets, and `t2`'s inputs `p1,p2` each have `t2` as their sole output. **The
  classifier returning `FreeChoice` is correct; the test's expectation is wrong.**
  (Independently, `workflow-2` reached the same conclusion and rewrote the fixture.)
  Fixing it means redesigning the fixture to be genuinely general while keeping its
  reachability assertions valid — a careful net-construction choice that is yours.

- **`t_net_source_transition_l4`** expects every transition L4 in a marked graph
  with a source transition and empty marking; it gets L0 for the circuit
  transitions. By the standard marked-graph criterion (live iff every directed
  circuit carries a token) the token-free circuit `t0→p0→t1→p1→t0` is dead, so
  whether the expectation or the algorithm is right depends on how source
  transitions are meant to interact with that criterion — your liveness theory.
  `workflow-2` reworked `liveness.rs` substantially here; that rework should be
  yours to author or ratify.

---

## 6. Issues — taken up, and proposed

**Effectively taken up by the delivered fixes (worth filing so the record exists):**

- *Simulation is inert* — `fire_unchecked` discards its result (unit 1). New.
- *Graph mirror wired in hash order* — corrupts `circuits()`/liveness/SCC (unit 2). New.
- *False deadlock-free on an m0 deadlock* (unit 3). New.

**Existing issues these units advance:**

- **#34 Full PNML (de)serialization** — unit 5 (strict import) is a down payment.
- **#44 SMT solver** / **#45 guided enumeration to verify a result** — unit 9
  (checkable-witness API) is exactly the verifier #45 describes.
- **#42 structural reductions** / classifier-driven dispatch — unit 8 (registry)
  is the seam new per-class algorithms plug into.

**Proposed new issues, clearly in scope (README + existing direction):**

1. **Make `cargo test --lib` green on `main`** — track the 6-red baseline as a
   correctness regression gate; wire CI to fail on red. (Highest leverage; the
   delivered branch closes 4 of 6.)
2. **Exact-arithmetic guard on negative verdicts** — unit 4 as its own tracked
   feature, with the Bareiss-vs-RREF performance follow-on noted.
3. **Property-test the petgraph mirror against the dense adjacency** — the bug in
   unit 2 was hash-order-dependent; a `proptest`/invariant test over random nets
   would have caught it and guards against regressions in RCM ordering.
4. **Replay-checkable reachability witnesses (#45 groundwork)** — return a
   firing word from positive reachability and expose a public `check` that
   replays it against the original net. Small, self-justifying, and the concrete
   seam SMT/guided-enumeration would target.
5. **Decide the abstention policy** — settle, repo-wide, whether structurally
   undecided predicates return `Option<bool>` (abstain) or a best-effort `bool`.
   Unit 6 assumes the former; this should be one explicit decision, not many
   scattered ones.
