# PR drafts + handoff

Copy-paste material for opening the fork branches as pull requests against
`MichaelOwenDyer/petrivet` and a cover message for Michael. All branches are on
`danielbdyer/petrivet`. Each is green except two inherited test failures that are
Michael's in-progress work (`general_net_reachability_fallback`,
`t_net_source_transition_l4`); clippy is at or below the baseline (31 lib
warnings) on every branch.

---

## Recommended order and dependencies

| # | Branch | Base it stacks on | Kind |
|---|--------|-------------------|------|
| 1 | `pv/correctness-fixes` | `main` | correctness (merge first) |
| 2 | `pv/exact-arithmetic` | `pv/correctness-fixes` | correctness |
| 3 | `pv/cluster-rank` | `pv/exact-arithmetic` | structural keystone |
| 4 | `pv/dot-export` | `pv/correctness-fixes` | usability (#9) |
| 5 | `pv/printable-types` | `pv/correctness-fixes` | usability |
| 6 | `pv/net-to-dot` | `pv/correctness-fixes` | usability |
| 7 | `pv/fire-sequence` | `pv/correctness-fixes` | completeness |
| 8 | `pv/hygiene` | `pv/correctness-fixes` | build hygiene |
| 9 | `pv/pnml-strict-import` | `pv/correctness-fixes` | policy (your call) |
| 10 | `pv/abstain-api` | `pv/correctness-fixes` | policy (your call, breaking) |

`pv/correctness-fixes` should merge first: it is the green floor the others build
on (e.g. the exact-arithmetic positive path replays firings, which needs the
firing fix). Until it merges, the stacked PRs' diffs also show its commits. #2→#3
is a chain; #4–#10 are independent of each other.

---

# COVER MESSAGE (paste to Michael)

Hey Michael,

I've been reading through petrivet closely with an AI assistant, and we ended up
preparing a set of small, self-contained branches for you to look at — each one
its own change with one reason to exist, so you can evaluate and take (or leave)
them independently rather than wading through one big diff.

A couple of things up front, because they shape everything:

- **Your `main` currently has six failing library tests.** Four are real bugs:
  firing was a no-op (`fire_unchecked` discarded its result), the petgraph mirror
  was wired in hash order so `circuits()` / liveness / strong-connectivity could
  return wrong answers, and a net that starts deadlocked was reported
  deadlock-free. The first branch (`pv/correctness-fixes`) fixes those and turns
  your suite from 6 red to 2.
- **The other two failures are yours** — in your latest (optimized-liveness)
  commit. I left them alone but wrote down a precise diagnosis (one is a test
  fixture that asserts `General` for a net that's genuinely extended-free-choice,
  so the classifier is actually right; the other is a marked-graph
  liveness-with-source-transition question that's your theory to settle).

The throughline across the branches is one rule: the fast/floating-point path may
*suggest* an answer, but a verdict is only emitted when an exact check confirms
it, and when nothing can confirm it the tool abstains rather than inventing a
result. That's what the exact-arithmetic branch is about — removing a class of
silent wrong "unreachable/unbounded" answers — and it's the honest version of the
"certifying" idea you were skeptical of: the value is *correct-because-checked*,
not a certificate object you have to care about.

Two branches are explicitly **your call**, because they change behaviour rather
than just fix a bug: rejecting non-unit-weight PNML arcs instead of silently
linearising them, and making `is_covered_by_s_components` return `Option<bool>`
(abstain) instead of a hardcoded `false`. I flagged the spots where I stopped
short because the answer depends on Petri-net theory I didn't want to guess at —
those are marked in the code and the PR text.

The rest are small usability/completeness things you'd probably have gotten to: a
DOT export for the state graph (your issue #9) and for the net itself, `Display`
for `Omega`/`Marking`/`Boundedness`, a `fire_sequence` to replay a witness, and a
fix for the `playground` example that no longer compiled.

None of this is meant to land as-is or to step on the thesis — the bigger
"thesis-shaped" framing (a coverage number, the firewall, a decider registry) I
deliberately kept *out* of these branches; it's a separate proposal you own, not
something riding in on a bug-fix PR. Take what's useful, push back on what isn't.
`DELIVERY.md` walks through every branch and why; `RESTRUCTURE.md` has the fuller
map.

— Daniel

---

# PR DESCRIPTIONS

## 1. `pv/correctness-fixes` → `main`

**Title:** Fix three correctness bugs: inert firing, petgraph mirror order, m0-deadlock

Three independent bug fixes, each with a regression test. `main` currently fails
six library tests; this branch fixes four of them (the remaining two are
in-progress work in `fdcf7fb`, untouched).

- **Inert `fire_unchecked`** — it computed `checked_sub(1)` / `checked_add(1)` for
  each pre/post place but discarded the result, so the marking was never mutated
  and `try_fire` / `fire_any` were no-ops. The checked result is assigned back.
  Greens `system::tests::basic_firing` and `into_parts`.
- **Petgraph mirror order** — in `NetBuilder::build` the petgraph node arrays were
  filled in `HashMap` iteration order but indexed by dense index, scrambling the
  node correspondence and wiring edges between the wrong nodes. This corrupted
  `circuits()`, marked-graph / state-machine liveness, and strong-connectivity in
  a hash-order-dependent way. The arrays are built in dense-index order. Greens
  two liveness tests; a structural invariant test pins the edge set against the
  dense adjacency.
- **m0-deadlock** — `deadlocks()` evaluated its predicate only on successor
  markings, never the seed, so a net whose initial marking is itself a deadlock
  was reported deadlock-free. The seed is now tested first, exactly once.

**Verification:** `cargo test -p petrivet --lib` 6-red → 2-red; clippy identical
to baseline.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

---

## 2. `pv/exact-arithmetic` → `main` (base: `pv/correctness-fixes`)

**Title:** Exact-rational guard on reachability/coverability/boundedness verdicts

Makes the negative verdicts of the three properties rest on exact rational
arithmetic instead of the floating-point LP/ILP merely failing — a degenerate
float could otherwise mint a false "unreachable" / "uncoverable" / "bounded". The
float path is kept as a *suggester*; soundness is added on the verdict.

- An exact ℚ linear-algebra core (`core/analysis/rational.rs`, `exact_matrix.rs`):
  `Rational` over `i128`, RREF, kernel / left-kernel, incidence over ℚ, and the
  marking-equation / covering-invariant decision functions. **No new dependency.**
- Reachability: `Unreachable` rests on `marking_equation_exact` over ℚ; anything
  not exactly infeasible escalates to the state-space search. The live-marked-
  graph positive is *realized* (the suggested firing vector is replayed and
  accepted only if it actually reaches the target) rather than returned as a
  rounded ILP vector.
- Coverability: `Uncoverable` rests on an exact non-negative place sub-invariant.
- Structural boundedness: `is_structurally_bounded` re-verifies the f64-suggested
  weight vector over ℚ (`y > 0`, `yᵀ·C ≤ 0`) before any `true`.

**Honest note:** the exact core is plain RREF over ℚ, not the fraction-free
(Bareiss) schedule, so on very large instances it may be slower than the path it
guards — a measured follow-on, not a regression on the suggestion.

**Verification:** lib 134 pass / 2 inherited; clippy identical to baseline.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

---

## 3. `pv/cluster-rank` → `main` (base: `pv/exact-arithmetic`)

**Title:** Add the free-choice cluster partition and the rank(C) = c−1 relation

`class.rs`'s Rank-Theorem documentation names a cluster count `c` but the repo
never defined it. This adds `clusters()` (the Desel–Esparza consumption-arc
union-find partition giving `c`) and `rank_cluster_relation` (pairs it with the
exact ℚ incidence rank; `holds()` tests `rank + 1 == c`). Gated honestly as a
**necessary** (not sufficient) condition of free-choice well-formedness, validated
against an independent BFS-flood oracle. Registers no decision procedure — it is
the keystone the future structural deciders build on — so it carries a localized
`#[allow(dead_code)]` with a note. Depends on the exact-arithmetic core for the
rank.

**Verification:** lib 138 pass / 2 inherited; clippy at baseline.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

---

## 4. `pv/dot-export` → `main` (base: `pv/correctness-fixes`)

**Title:** Add StateGraph::to_dot — Graphviz export of state graphs (#9)

Renders a reachability or coverability graph as Graphviz DOT: nodes labelled with
their marking (`p<id>:<tokens>`, `∅` for empty), edges with the fired transition.
Works for both `u32` and `Omega` token types. Rendering only — no analysis surface
touched. Tests cover a bounded reachability graph and an unbounded coverability
graph (the ω path). Closes #9.

**Verification:** lib 111 pass / 2 inherited; clippy at baseline.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

---

## 5. `pv/printable-types` → `main` (base: `pv/correctness-fixes`)

**Title:** Add Display for Omega, Marking, and Boundedness

These public types could only be printed via `Debug` (a marking showed as
`Marking { support: UniqueSortedSlice([...]) }`). Adds the natural `Display`
impls, matching the existing `Display` for `NetClass` and `LivenessLevel`:
`Omega` → number / `ω`; `Marking<T: Display>` → `{p1: 2, p3: 1}` / `∅`;
`Boundedness` → `bounded (k)` / `unbounded`. Rendering only; unit tests for each.

**Verification:** lib pass / 2 inherited; clippy at baseline.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

---

## 6. `pv/net-to-dot` → `main` (base: `pv/correctness-fixes`)

**Title:** Add Net::to_dot and PetriNet::to_dot — visualize the net structure

`StateGraph::to_dot` renders the state space; there was no way to render the net
itself. `Net::to_dot` emits Graphviz DOT for the structure (places as circles,
transitions as boxes, arcs as edges, labelled with public ids); `PetriNet::to_dot`
additionally labels each place with its current token count and draws enabled
transitions bold. Rendering only; tests for both.

**Verification:** lib 111 pass / 2 inherited; clippy at baseline.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

---

## 7. `pv/fire-sequence` → `main` (base: `pv/correctness-fixes`)

**Title:** Add PetriNet::fire_sequence — replay a sequence of transitions

Fires each transition of a sequence in order, returning `Err(NotEnabled(t))` at
the first that is not enabled when its turn comes (marking left at the last
successful firing). The natural way to replay a reachability `FiringSequence`
witness against a system. Tested for full replay and mid-sequence block.

**Verification:** lib 110 pass / 2 inherited; clippy at baseline.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

---

## 8. `pv/hygiene` → `main` (base: `pv/correctness-fixes`)

**Title:** Build hygiene: fix the playground example and three clippy lints

`examples/playground.rs` no longer compiled (`Marking::support()` now yields
`Place` by value but the example destructured `&(place, tokens)`); it reads the
count via `Marking::get`. Also documents the `# Errors` case of
`commoner_hack_criterion` and makes `NetBuilder::place_count` / `transition_count`
`const fn`. Clippy 31 → 28; lib tests unchanged.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

---

## 9. `pv/pnml-strict-import` → `main` (base: `pv/correctness-fixes`) — your call

**Title:** PNML import: reject silent misimports (non-unit weights, >u32 markings)

The P/T converter parsed but ignored a non-unit arc `<inscription>` weight
(collapsing weight-k to weight-1) and saturated an `<initialMarking>` above
`u32::MAX`. Both import a structurally *different* net than the file describes.
Adds two `PnmlConversionError` variants — `NonUnitWeightArc { arc_id, weight }`
and `MarkingOverflow { place_id, value }` — and rejects each case. Absent
inscription still means weight 1; a marking exactly at `u32::MAX` is still
accepted. Four boundary tests.

**Your call:** this is an interim posture, not real weighted-arc support (issue
#57). The policy — reject vs. clamp vs. wait for weighted arcs — is yours to set.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

---

## 10. `pv/abstain-api` → `main` (base: `pv/correctness-fixes`) — your call, breaking

**Title:** Abstain instead of fabricating: is_covered_by_s_components → Option<bool>

Two predicates returned an invented negative rather than admitting they could not
decide:

- `Net::is_covered_by_s_components` was a `// todo` stub returning a hardcoded
  `false`, consumed at `is_efficiently_bounded` for live free-choice nets and
  reported as unboundedness with no evidence. It now returns `Option<bool>` and
  abstains (`None`) until a certifying S-component decomposition exists;
  `is_efficiently_bounded` propagates the abstention to the exact coverability
  graph. **Breaking** (return type changed).
- `is_efficiently_deadlock_free` returned `Some(false)` for non-live free-choice
  nets; non-liveness does not prove a reachable deadlock, so that was a fabricated
  negative. The arm is dropped (abstain instead).

**Left for you (flagged in code):** the `MarkedGraph` non-strongly-connected arm
and the `AsymmetricChoice` Commoner–Hack arm in the same method are arguably
further fabricated verdicts, but they turn on marking-dependent theory — your call,
not guessed.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
