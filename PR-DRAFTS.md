# PR drafts + handoff

Copy-paste material for opening the fork branches as pull requests against
`MichaelOwenDyer/petrivet`, and a cover message for Michael. All branches are on
`danielbdyer/petrivet`, prefixed with their merge order. Each is green except two
inherited test failures that are Michael's in-progress work
(`general_net_reachability_fallback`, `t_net_source_transition_l4`); clippy is at
or below the baseline (31 lib warnings) on every branch.

---

## Order and dependencies

| Branch | Base it stacks on |
|--------|-------------------|
| `pv/01-correctness-fixes` | `main` |
| `pv/02-exact-arithmetic` | `pv/01-correctness-fixes` |
| `pv/03-cluster-rank` | `pv/02-exact-arithmetic` |
| `pv/04-dot-export` | `pv/01-correctness-fixes` |
| `pv/05-printable-types` | `pv/01-correctness-fixes` |
| `pv/06-net-to-dot` | `pv/01-correctness-fixes` |
| `pv/07-fire-sequence` | `pv/01-correctness-fixes` |
| `pv/08-hygiene` | `pv/01-correctness-fixes` |
| `pv/09-pnml-strict-import` | `pv/01-correctness-fixes` |
| `pv/10-abstain-api` | `pv/01-correctness-fixes` |

`01` merges first. `02 → 03` are a chain. `04`–`10` each depend only on `01` and
are independent of one another (the numbers are a suggested sequence, not a hard
order). Until `01` merges, the stacked PRs' diffs also show its commits.

---

# COVER MESSAGE (paste to Michael)

Hey Michael,

I put together a set of small branches on my fork, each a single self-contained
change, numbered in the order they'd merge. What each one is:

1. **`pv/01-correctness-fixes`** — fixes three bugs, each with a regression test.
   `fire_unchecked` discarded its result, so firing was a no-op (`try_fire` /
   `fire_any` left the marking unchanged). The petgraph mirror was built in
   hash-map order but indexed by dense order, so `circuits()`, marked-graph /
   state-machine liveness, and strong-connectivity could be wrong. And
   `deadlocks()` never tested the initial marking, so a net that starts deadlocked
   was reported deadlock-free. Takes the library suite from six failures to two
   (the remaining two are in your latest commit — diagnosed, not touched).

2. **`pv/02-exact-arithmetic`** (on 01) — the negative reachability, coverability,
   and boundedness verdicts rested on a floating-point LP/ILP merely failing,
   where a degenerate float could mint a false "unreachable" / "uncoverable" /
   "bounded". They now rest on an exact rational check, with the float path kept
   as a suggester. Adds a small exact-ℚ linear-algebra core; no new dependency.

3. **`pv/03-cluster-rank`** (on 02) — adds the free-choice cluster partition
   (`clusters()`, giving the count `c` your Rank-Theorem doc names) and the
   `rank(C) = c−1` relation, gated as necessary (not sufficient) and checked
   against an independent oracle. No decider yet — just the keystone.

4. **`pv/04-dot-export`** (on 01) — `StateGraph::to_dot`, a Graphviz export of the
   reachability / coverability graph. Closes #9.

5. **`pv/05-printable-types`** (on 01) — `Display` for `Omega`, `Marking`, and
   `Boundedness`, which previously only had `Debug`.

6. **`pv/06-net-to-dot`** (on 01) — `Net::to_dot` and `PetriNet::to_dot` to render
   the net structure (and, for a `PetriNet`, its current marking).

7. **`pv/07-fire-sequence`** (on 01) — `PetriNet::fire_sequence`, to replay a list
   of transitions; returns `Err(NotEnabled)` at the first one that is blocked.

8. **`pv/08-hygiene`** (on 01) — fixes the `playground` example (which no longer
   compiled) and two clippy lints.

9. **`pv/09-pnml-strict-import`** (on 01) — rejects non-unit-weight arcs and
   markings above `u32::MAX` on PNML import instead of silently importing a
   different net. A behaviour change — the policy is your call.

10. **`pv/10-abstain-api`** (on 01) — `is_covered_by_s_components` returns
    `Option<bool>` (abstains) instead of a hardcoded `false`, and the structural
    deadlock-free path drops a `Some(false)` it couldn't justify. Breaking, and a
    policy call — yours to ratify.

`DELIVERY.md` has the per-branch detail; `RESTRUCTURE.md` the fuller map. Take
what's useful.

— Daniel

---

# PR DESCRIPTIONS

## `pv/01-correctness-fixes` → `main`

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

## `pv/02-exact-arithmetic` → `main` (base: `pv/01-correctness-fixes`)

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

## `pv/03-cluster-rank` → `main` (base: `pv/02-exact-arithmetic`)

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

## `pv/04-dot-export` → `main` (base: `pv/01-correctness-fixes`)

**Title:** Add StateGraph::to_dot — Graphviz export of state graphs (#9)

Renders a reachability or coverability graph as Graphviz DOT: nodes labelled with
their marking (`p<id>:<tokens>`, `∅` for empty), edges with the fired transition.
Works for both `u32` and `Omega` token types. Rendering only — no analysis surface
touched. Tests cover a bounded reachability graph and an unbounded coverability
graph (the ω path). Closes #9.

**Verification:** lib 111 pass / 2 inherited; clippy at baseline.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

---

## `pv/05-printable-types` → `main` (base: `pv/01-correctness-fixes`)

**Title:** Add Display for Omega, Marking, and Boundedness

These public types could only be printed via `Debug` (a marking showed as
`Marking { support: UniqueSortedSlice([...]) }`). Adds the natural `Display`
impls, matching the existing `Display` for `NetClass` and `LivenessLevel`:
`Omega` → number / `ω`; `Marking<T: Display>` → `{p1: 2, p3: 1}` / `∅`;
`Boundedness` → `bounded (k)` / `unbounded`. Rendering only; unit tests for each.

**Verification:** lib pass / 2 inherited; clippy at baseline.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

---

## `pv/06-net-to-dot` → `main` (base: `pv/01-correctness-fixes`)

**Title:** Add Net::to_dot and PetriNet::to_dot — visualize the net structure

`StateGraph::to_dot` renders the state space; there was no way to render the net
itself. `Net::to_dot` emits Graphviz DOT for the structure (places as circles,
transitions as boxes, arcs as edges, labelled with public ids); `PetriNet::to_dot`
additionally labels each place with its current token count and draws enabled
transitions bold. Rendering only; tests for both.

**Verification:** lib 111 pass / 2 inherited; clippy at baseline.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

---

## `pv/07-fire-sequence` → `main` (base: `pv/01-correctness-fixes`)

**Title:** Add PetriNet::fire_sequence — replay a sequence of transitions

Fires each transition of a sequence in order, returning `Err(NotEnabled(t))` at
the first that is not enabled when its turn comes (marking left at the last
successful firing). The natural way to replay a reachability `FiringSequence`
witness against a system. Tested for full replay and mid-sequence block.

**Verification:** lib 110 pass / 2 inherited; clippy at baseline.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

---

## `pv/08-hygiene` → `main` (base: `pv/01-correctness-fixes`)

**Title:** Build hygiene: fix the playground example and three clippy lints

`examples/playground.rs` no longer compiled (`Marking::support()` now yields
`Place` by value but the example destructured `&(place, tokens)`); it reads the
count via `Marking::get`. Also documents the `# Errors` case of
`commoner_hack_criterion` and makes `NetBuilder::place_count` / `transition_count`
`const fn`. Clippy 31 → 28; lib tests unchanged.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

---

## `pv/09-pnml-strict-import` → `main` (base: `pv/01-correctness-fixes`) — your call

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

## `pv/10-abstain-api` → `main` (base: `pv/01-correctness-fixes`) — your call, breaking

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
