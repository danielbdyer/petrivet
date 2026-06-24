# Delivery: branches and changes

Per-branch decomposition of the work on the fork. Each entry states what the
change is and why it was made. Branches stack in this order; each is green except
two inherited failures (`general_net_reachability_fallback`,
`t_net_source_transition_l4`) which are the maintainer's in-progress work, left
untouched. Clippy is at or below the baseline (31 lib warnings) on every branch.

```
origin/main
 └─ pv/01-correctness-fixes
     ├─ pv/02-exact-arithmetic ── pv/03-cluster-rank
     ├─ pv/04-dot-export
     ├─ pv/05-printable-types
     ├─ pv/06-net-to-dot
     ├─ pv/07-fire-sequence
     ├─ pv/08-hygiene
     ├─ pv/09-pnml-strict-import
     └─ pv/10-abstain-api
```

Branch names are prefixed with their merge order: `01` first, `02 → 03` a chain,
`04`–`10` each depending only on `01`.

---

## `pv/01-correctness-fixes` (off `origin/main`)

Three independent bug fixes, each with a regression test. On `origin/main` six
library tests fail; this branch fixes four of them (the other two are the
maintainer's WIP).

**Fix inert `fire_unchecked`** — `api/system/mod.rs`.
`fire_unchecked` computed `marking[p].checked_sub(1)` / `checked_add(1)` but
discarded the result, so the marking was never updated and every firing path
(`try_fire`, `fire_any`) was a no-op. The result is now assigned back. Greens the
maintainer's `basic_firing` and `into_parts` tests.

**Fix petgraph mirror node order** — `api/builder.rs`.
In `NetBuilder::build` the petgraph node arrays were filled in `HashMap`
iteration order but indexed by dense index, so the place/transition → node
correspondence was scrambled and edges were wired between the wrong nodes. This
corrupted `circuits()`, marked-graph / state-machine liveness, and
strong-connectivity in a hash-order-dependent way. The arrays are now built in
dense-index order. Greens two liveness tests; a structural invariant test
(`petgraph_mirror_agrees_with_dense_adjacency`) pins the edge set against the
dense adjacency.

**Fix m0-deadlock blind spot** — `api/system/deadlock_freedom.rs`.
The deadlock search evaluated its predicate only on newly-discovered *successor*
markings, never the initial marking. A net whose initial marking is itself a
deadlock was therefore reported deadlock-free. `deadlocks()` now tests the seed
marking first, exactly once. New test `initial_marking_deadlock_is_detected`.

---

## `pv/02-exact-arithmetic` (off `pv/01-correctness-fixes`)

Makes the negative verdicts of reachability, coverability, and boundedness rest on
exact rational arithmetic instead of the floating-point LP/ILP merely failing.
Stacked on `pv/01-correctness-fixes` because the realized-positive path (below) calls
`try_fire`, which is inert until that branch's fire fix lands.

**Exact-rational linear-algebra core** — `core/analysis/rational.rs`,
`core/analysis/exact_matrix.rs`, `core/analysis/mod.rs`.
`rational.rs`: an exact `Rational` over `i128` (reduced, sign-normalized; checked
arithmetic). `exact_matrix.rs`: a `Matrix` over `Rational` with RREF, kernel /
left-kernel (P/T-semiflows), `incidence_over_rationals`, and three decision
functions used below. No new crate dependency.

**Exact negative reachability** — `api/system/reachability.rs`.
The `Unreachable` verdict previously came from the f64 LP or ILP failing to find a
solution; a degenerate float could then report a reachable target as unreachable.
It now rests on `marking_equation_exact` over ℚ (`target − m₀ ∉ col(C)`), and
anything not exactly infeasible escalates to the state-space search. The
live-marked-graph *positive* previously returned the f64-rounded ILP vector
directly (which need not satisfy the equation); it is now *realized* — the
suggested firing vector is replayed and accepted only if it actually reaches the
target. Adds `IdxMarking::<u32>::wide_sum` (a token sum that cannot wrap) and two
tests.

**Exact negative coverability** — `api/system/coverability.rs`.
The `Uncoverable` verdict now rests on `covering_invariant_exact` over ℚ (a
non-negative place sub-invariant `y` with `yᵀ·C ≤ 0`, `yᵀ·target > yᵀ·M₀`)
instead of the f64 covering LP/ILP failing; otherwise it escalates to the
Karp–Miller coverability graph.

**Retire the superseded f64 solvers** — `core/analysis/semi_decision.rs`.
The four f64 solver functions whose only callers were the replaced verdict arms
are kept (they may seed a future guided path) but carry a localized
`#[allow(dead_code)]` with a note, restoring clippy parity.

**Exact structural boundedness** — `api/net/boundedness.rs`,
`core/analysis/exact_matrix.rs`, `core/net/mod.rs`.
`Net::is_structurally_bounded` / `is_place_structurally_bounded` returned the bare
f64 LP result as a verdict; a near-boundary `yᵀ·C` rounded to ≤ 0 could mint a
false `true`. The f64 LP now only suggests a weight vector, which is rationalized
(over a few scalings) and re-verified over ℚ by
`exact_matrix::is_{positive,semipositive}_place_subinvariant` before any `true`.
The now-orphaned f64 verdict methods on `DenseNet` are removed.

---

## `pv/03-cluster-rank` (off `pv/02-exact-arithmetic`)

**Cluster partition and rank relation** — `core/analysis/cluster.rs`,
`core/analysis/mod.rs`.
`class.rs`'s Rank-Theorem documentation names a cluster count `c` but the repo
never defined it. `clusters()` computes the Desel–Esparza consumption-arc
partition (union-find) giving `c`; `rank_cluster_relation` pairs it with the exact
ℚ incidence rank and `holds()` tests `rank + 1 == c`. Gated as a *necessary* (not
sufficient) condition of free-choice well-formedness, validated against an
independent BFS-flood oracle. Registers no decision procedure; carries a
module-level `#[allow(dead_code)]` because the consumers are future deciders.
Depends on `exact_matrix` for the rank.

---

## `pv/09-pnml-strict-import` (off `pv/01-correctness-fixes`)

**Reject silent misimports** — `api/pnml/convert.rs`.
The P/T converter parsed but ignored a non-unit arc `<inscription>` weight
(collapsing weight-k to weight-1) and saturated an `<initialMarking>` above
`u32::MAX`. Both import a structurally different net than the file describes. Two
`PnmlConversionError` variants are added — `NonUnitWeightArc { arc_id, weight }`
and `MarkingOverflow { place_id, value }` — and each case is now an error. Absent
inscription still means weight 1; a marking exactly at `u32::MAX` is still
accepted. Four boundary tests. This is an interim posture, not real weighted-arc
support (issue #57); the reject-vs-clamp policy is the maintainer's to set.

---

## `pv/10-abstain-api` (off `pv/01-correctness-fixes`)

Two predicates returned an invented negative rather than abstaining. Breaking API
change; a deliberate soundness-policy stance the maintainer ratifies.

**`is_covered_by_s_components → Option<bool>`** — `api/net/mod.rs`,
`api/system/boundedness.rs`.
The method was a `// todo` stub returning a hardcoded `false`, consumed at
`is_efficiently_bounded` for live free-choice nets and reported as unboundedness
with no evidence. It now returns `Option<bool>` and abstains (`None`) until a
certifying S-component decomposition exists; `is_efficiently_bounded` propagates
the abstention so `is_bounded` falls through to the exact coverability graph.

**Drop the fabricated deadlock negative** — `api/system/deadlock_freedom.rs`.
`is_efficiently_deadlock_free` returned `Some(false)` for non-live free-choice
nets. Non-liveness does not prove a reachable deadlock, so that was a fabricated
negative; the arm is dropped (abstain instead). New test
`efficient_deadlock_free_never_fabricates_false`. Two further arms in the same
method (the `MarkedGraph` non-strongly-connected case, and the `AsymmetricChoice`
Commoner–Hack case) are left unchanged and flagged in the commit message — they
turn on marking-dependent theory that is the maintainer's call.

---

## Usability and completeness additions (each off `pv/01-correctness-fixes`)

Independent of the original PR; small, additive, no analysis/verdict surface
touched.

**`pv/08-hygiene`** — `examples/playground.rs` no longer compiled (`Marking::support`
now yields `Place` by value); fixed to read the count via `Marking::get`. Plus a
`# Errors` doc on `commoner_hack_criterion` and `const fn` on
`NetBuilder::place_count` / `transition_count`. Clippy 31 → 28.

**`pv/04-dot-export`** — `StateGraph::to_dot` (issue #9): Graphviz export of the
reachability/coverability graph (nodes labelled with their marking, edges with the
fired transition), for both `u32` and `Omega` token types.

**`pv/05-printable-types`** — `Display` for `Omega` (`ω` / number), `Marking<T>`
(`{p1: 2, p3: 1}` / `∅`), and `Boundedness` (`bounded (k)` / `unbounded`). These
public types could previously only be `Debug`-printed; this matches the existing
`Display` for `NetClass` and `LivenessLevel`.

**`pv/06-net-to-dot`** — `Net::to_dot` (places as circles, transitions as boxes, arcs
as edges) and `PetriNet::to_dot` (additionally labels each place with its token
count and draws enabled transitions bold). Complements the state-graph export.

**`pv/07-fire-sequence`** — `PetriNet::fire_sequence` fires a sequence of transitions
in order, returning `Err(NotEnabled(t))` at the first that is not enabled (marking
left at the last successful firing). The natural way to replay a reachability
`FiringSequence` witness.

---

## `pv/restructure-plan` (docs only, off `origin/main`)

`RESTRUCTURE.md` (the unit decomposition map, extraction paths for unbuilt units,
and the two-failure hand-back) and this file. No code.
