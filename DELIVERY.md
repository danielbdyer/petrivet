# petrivet — delivery narrative

Status: a plain-language account of the work done on the fork, written so the
**why** is as legible as the **what**. Companion to [`RESTRUCTURE.md`](RESTRUCTURE.md)
(which is the terse decomposition map). Every "the code does X" here is backed by a
commit and a passing test; every judgement call is named as such.

This is offered as material for the maintainer to read, accept, refine, or reject
— not as a decision already taken. Nothing has been pushed to the upstream
repository; all work sits on branches of the fork.

---

## 1. The two problems this work answers

The maintainer gave two pieces of feedback on the original large pull request:

1. **"I don't see how *certifying* the net adds value to someone using the
   library."** A fair objection — and §4 answers it directly.
2. **"I couldn't make sense of one massive commit."** Also fair: ~15k lines mixing
   bug fixes, an arithmetic layer, a certificate framework, and a thesis proposal
   cannot be evaluated as a unit.

Underneath both sits a discovery that reframes everything (§2): the shared
baseline was already broken. So the work was re-organised around a single
principle — **decompose into independent units, each of which is either a
bug the maintainer can confirm against his own tests, or a self-contained
capability with one statable reason to exist** — and around a single technical
doctrine: **never let the tool state something it cannot back up** (§3).

---

## 2. The reframing discovery: `origin/main` was red

Before adding anything, running the maintainer's own test suite on the baseline
(`origin/main`, commit `fdcf7fb`) showed **six failing library tests**. The
breakage reached the README's headline features:

- **Simulation was inert.** `fire_unchecked` computed the new token counts and
  *threw them away* — `try_fire` / `fire_any` left the marking unchanged. The
  maintainer's own `basic_firing` test was failing.
- **The graph mirror was miswired.** The petgraph copy of the net was built in
  hash-map order but indexed by dense order, so `circuits()`, marked-graph /
  state-machine liveness, and strong-connectivity returned hash-order-dependent
  wrong answers. Three liveness tests were failing.

This changes the framing of the whole contribution. Most of the original PR's
bug-fix value is best stated as: **"your main branch is red; here are small,
single-cause commits that make it green."** That is the most reviewable possible
form — each fix is confirmed by a test the maintainer already wrote.

(Two of the six failures are *not* ours to fix — see §6.)

---

## 3. The organising doctrine: suggest, then verify; abstain, never fabricate

A Petri-net analyser answers questions like "can this bad state happen?" The
dangerous failure is not slowness — it is a **confident wrong answer**: reporting
a state unreachable when it is reachable tells a user their system is safe when it
is not. Every unit below serves one rule:

> The fast, heuristic, or floating-point path may **suggest** an answer, but a
> verdict is only emitted when an **exact** check confirms it. When nothing can
> confirm it, the tool **abstains** (returns "don't know") rather than inventing a
> result.

This rule is what makes the rest coherent — and, as §4 explains, it is also the
honest core of what "certifying" was reaching for.

---

## 4. The certification-value question, answered

The maintainer's objection is correct *as stated*, because the original PR bundled
two unrelated things under the word "certificate":

**(a) "The answer is checked, not guessed" — real, delivered value.**
This is the exact-arithmetic work (§5, unit 4). Today a near-boundary
floating-point result can make the solver report a marking *unreachable* when it
is not — a silent false "your system is safe." The exact-rational guard removes
that whole class of wrong answers while keeping the fast path as a suggester. The
value needs no certificate vocabulary to state: **petrivet does not hand you a
wrong analysis answer, even when the fast path would.** This is on the end-user's
call path now.

**(b) Certificate *objects* handed back to the user — value not yet real.**
The framework of `Verdict`/`Certificate`/ledger objects was on no public call
path; the analysis methods still returned their ordinary results. An object like
that earns its place only when it answers a question the *user* asks — and there
is exactly one such question in the repository, written by the maintainer himself:
**issue #45**, "verify an SMT-solver result." That is the genuine end-user story:
an untrusted fast/SMT producer proposes an answer, and petrivet **cheaply and
exactly checks it.** Everything beyond that — a "trusted-base ledger," a coverage
number, a "firewall" figure of merit — measures a property of the *implementation*
(how much of the answer space is proof-backed). That is a thesis question, not a
user one. It has been set aside as a separate proposal, not deleted; the
user-facing residue (the exact guard, and a future #45-shaped checker) is what
remains in scope.

So: **certification delivers real value in exactly one mode — correct-because-
checked — and that value is captured by the exact guard plus, eventually, a
checkable-witness API for #45.** The rest is reframed as thesis material the
maintainer owns.

---

## 5. What was delivered, and why each piece matters

Each unit is an independent branch on the fork. "Why" is stated in end-user /
maintainer terms. All are green except two inherited failures (§6); clippy is at
or below the baseline warning count throughout.

### Units 1–3 — correctness fixes (`pv/correctness-fixes`)
*Why: the library's advertised features (simulation, liveness, deadlock-freedom)
were silently wrong. These turn the maintainer's red suite green, one cause each.*

- **Inert `fire_unchecked`.** Assign the computed token counts back. Simulation
  mutates the marking again. (Greens `basic_firing`, `into_parts`.)
- **Petgraph mirror order.** Build the node arrays in dense-index order so the
  graph matches how it is indexed. (Greens two liveness tests; a structural
  invariant test guards it.)
- **m0-deadlock blind spot.** The deadlock search never tested the *initial*
  marking, so a net that starts deadlocked was reported deadlock-free — a false
  "your system always makes progress." Now the seed marking is tested first.

### Unit 4 — exact-arithmetic guard (`pv/exact-arithmetic`)
*Why: this is "correct-because-checked" made real (see §4a) — the single most
important correctness feature.*

A small exact-rational linear-algebra core (`rational.rs`, `exact_matrix.rs`; no
new dependency) now backs every **negative** verdict:

- **Reachability** "unreachable" rests on an exact ℚ check (`target − m₀ ∉
  col(C)`), not on the float solver failing. The live-marked-graph *positive* is
  *realised* — the suggested firing vector is replayed and accepted only if it
  truly reaches the target — instead of returning a rounded vector that might not.
- **Coverability** "uncoverable" rests on an exact non-negative place
  sub-invariant, not on the float covering-LP failing.
- **Structural boundedness** "bounded" rests on an exact place sub-invariant
  (`y > 0`, `yᵀ·C ≤ 0` over ℚ); the float LP only *suggests* the weights.

In every case the float path is kept as a suggester; when ℚ cannot certify, the
tool escalates to the exact, terminating state-space search rather than guessing.
Honest caveat carried in the code: the exact core is plain RREF over ℚ, not the
fraction-free (Bareiss) schedule, so on very large instances it may be slower than
the path it guards — a measured follow-on, not a regression on the suggestion.

### Unit 7 — cluster partition + rank relation (`pv/cluster-rank`)
*Why: `class.rs`'s Rank-Theorem documentation names a cluster count `c` but the
repo never defined it. This supplies the missing definition, tested.*

`clusters()` computes the Desel–Esparza consumption-arc partition (giving `c`),
and `rank(C) = c − 1` is paired with the exact incidence rank. It is gated
honestly as a **necessary** (not sufficient) condition of free-choice
well-formedness, validated against an independent flood-fill oracle. It registers
no decision procedure yet — it is the keystone the future structural deciders
build on.

### Unit 5 — strict PNML import (`pv/pnml-strict-import`)
*Why: loading a file should not silently change the net it describes.*

A non-unit arc weight was parsed and ignored (collapsing weight-k to weight-1),
and a token count above `u32::MAX` was saturated. Both import a structurally
*different* net than the file. Now each is a typed error. This is an interim
posture, not a substitute for real weighted-arc support (issue #57); the policy —
reject vs. clamp vs. wait — is the maintainer's to set.

### Unit 6 — abstain instead of fabricate (`pv/abstain-api`)
*Why: two predicates returned an invented negative rather than admitting they
could not decide.*

- `is_covered_by_s_components` was a stub returning a hardcoded `false`, consumed
  as an *unboundedness* verdict. It now returns `Option<bool>` and abstains
  (`None`); the boundedness check propagates the abstention to the exact
  coverability graph. (Breaking API change — the maintainer's to ratify.)
- `is_efficiently_deadlock_free` returned `Some(false)` for non-live free-choice
  nets; non-liveness does not prove a reachable deadlock, so that was a fabricated
  negative. The arm is dropped (abstain instead).

Two further arms in the same method (`MarkedGraph` non-strongly-connected, and the
`AsymmetricChoice` Commoner–Hack arm) were **deliberately left alone and flagged**
— they turn on marking-dependent Petri-net theory that is the maintainer's call,
not ours to guess.

---

## 6. What is deliberately *not* touched, and why

Discipline here is itself part of the contribution: the deep Petri-net theory is
the maintainer's.

- **Two inherited test failures are left for him**, with a precise diagnosis
  rather than a guessed fix:
  - `general_net_reachability_fallback` asserts a net is `General`, but the net he
    wrote is genuinely extended-free-choice — **the classifier is correct and the
    test's expectation is wrong.** Correcting it means redesigning the fixture,
    his call.
  - `t_net_source_transition_l4` is a marked-graph liveness-with-source-transition
    question — his theory.
- **The thesis framing** (the ledger, the coverage number `f_struct`, the
  "firewall," the learned-selection ladder) is reframed as a separate proposal he
  owns (§4b), not merged into library PRs and not deleted.
- **Theory-dependent verdict arms** (above) are flagged, not changed.

---

## 7. How it all fits together

Branch stack on the fork (each layer green but for the two inherited §6 failures;
clippy at baseline parity throughout):

```
origin/main  (the maintainer's baseline — 6 failing tests)
 └─ pv/correctness-fixes      units 1–3   bug fixes (6 → 2 red)
     ├─ pv/exact-arithmetic    unit 4      exact ℚ guards (reach / cover / bound)
     │   └─ pv/cluster-rank    unit 7      cluster partition + rank relation
     ├─ pv/pnml-strict-import  unit 5      reject silent misimports
     └─ pv/abstain-api         unit 6      Option<bool>, drop fabricated negatives
pv/restructure-plan            RESTRUCTURE.md + DELIVERY.md (these notes)
```

The dependency shape, stated plainly: the correctness fixes are the floor
everything stands on (the exact guard's *realised* positive needs working
`fire`). The exact core is a root the cluster keystone builds on. The import and
abstention units are independent policy choices. The certificate-objects work
(issue #45) is a separate stack not yet started.

The throughline: the parts that map to a **user** question — correctness,
simulation that works, verifying an external result — are real, small, and
mergeable in isolation. The parts that mapped to a **thesis** question are lifted
out and handed back. That is the whole delivery.
