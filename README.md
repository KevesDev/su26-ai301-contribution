# Contribution 1: Support zero-operand instructions in circuit drawers

**Contribution Number:** 1 
**Student:** Stan Riane Nelson  
**Issue:** https://github.com/Qiskit/qiskit/issues/9962  
**Pull Request:** https://github.com/Qiskit/qiskit/pull/16453  
**Status:** Phase IV Complete — Awaiting review

---

## Why I Chose This Issue
I chose issue #9962, 'Support zero-operand instructions in circuit drawers' in Qiskit, because it allows me to work within a high-performance Python/Rust ecosystem, aligning with my systems-level development background. The scope is highly specific, targeting the visualization layer rather than the core quantum matrix algorithms. Furthermore, analyzing the stalled 2024 PR for this issue will provide an excellent architectural starting point for my Phase II reproduction.

---

## Understanding the Issue

### Problem Description
Zero-operand instructions (such as `GlobalPhaseGate`) fail to render correctly in the visual circuit drawers. Because these gates act on the entire circuit state rather than specific target qubits, they lack the standard operand arguments (`qargs`) that the UI layout engines rely on to calculate coordinate placement and box spans.

### Expected Behavior
A zero-qubit gate should occupy its own dedicated vertical column to indicate its place in the circuit's timeline, and it should be drawn as a unified box visually spanning all circuit wires (without explicit target markers).

### Current Behavior
In the `matplotlib` drawer, a previous architectural hack forces zero-qubit gates to reset their internal x-index to 0, causing multiple zero-qubit gates to render directly on top of each other at the beginning of the circuit. In the `text` drawer, the gate attempts to render but fails to draw the vertical box lines connecting the wires, and suffers from string misalignment on the bottom wire.

> **Phase III correction:** once I actually instrumented the pipeline (rather than reasoning about it from the stalled PR), the real behavior was different from both of the above descriptions, and worse: the gate doesn't render badly, it doesn't render *at all*. Both the text and matplotlib drawers silently dropped `GlobalPhaseGate` instructions completely — no box, no error, nothing on `print(circ.draw('text'))`. See "Solution Approach → Phase III correction" below for the actual root cause.

### Affected Components
- `qiskit/visualization/circuit/_utils.py` (`_get_layered_instructions`, `_LayerSpooler`) — **this is the actual root cause, identified in Phase III; it is not mentioned at all in my original Phase II plan.**
- `qiskit/visualization/circuit/matplotlib.py` (`_get_coords`, `_multiqubit_gate`)
- `qiskit/visualization/circuit/text.py` (`_set_multibox`, `set_qu_multibox`, `_node_to_gate`)
- `qiskit/visualization/circuit/latex.py` (`_build_latex_array`, `_build_multi_gate`) — not in my original plan; see below.

---

## Reproduction Process

### Environment Setup
Cloned my fork (`https://github.com/KevesDev/qiskit`) locally using GitHub Desktop. Created a clean Python virtual environment and installed the Qiskit visualization development dependencies (`pip install --no-cache-dir -e '.[visualization]'`). Corrected a cross-compilation toolchain mismatch by explicitly targeting the 64-bit Windows MSVC Rust toolchain to match the Python interpreter.

### Steps to Reproduce
1. Initialize a `QuantumCircuit(5)`.
2. Append standard single/multi-qubit gates, followed by consecutive `GlobalPhaseGate(np.pi)` insertions.
3. Call `circ.draw('mpl')` and `circ.draw('text')` and observe the output.

### Reproduction Evidence
- **Commit showing reproduction:** https://github.com/KevesDev/qiskit/blob/92da65f7cd60959a26da0592d1d945ba8af2382d/reproduce.py (linked by commit SHA rather than branch name — the script itself was removed before opening the PR, since a loose root-level script isn't idiomatic for a Qiskit contribution and the real unit tests now cover the same reproduction; see Phase IV notes below.)
- **Screenshots/logs:** ![Matplotlib Bug](https://github.com/KevesDev/su26-ai301-contribution/blob/main/mpl_bug.png?raw=true) (this screenshot is the *original*, pre-fix bug — both `GlobalPhaseGate` calls are completely invisible; there is no "stacked boxes" artifact like I'd originally assumed.)
- **My findings (Phase II):** Confirmed that the upstream layer assignment (`_LayerSpooler`) works perfectly, meaning the core engine is sound. The bug is localized entirely to the rendering loops in the `text` and `matplotlib` visualization modules.
- **My findings (Phase III, corrected):** That Phase II finding was wrong. `_LayerSpooler` never sees zero-qarg nodes at all — `dag.layers()` (which it iterates) only ever visits nodes reachable by walking the DAG's wires, and a `GlobalPhaseGate` has no wires. The node is dropped before the rendering loops ever get a chance to run. I confirmed this directly:
  ```python
  >>> dag.op_nodes()             # includes the global_phase node
  [h, global_phase, x, global_phase]
  >>> list(dag.layers())          # global_phase nodes are simply absent
  [{h}, {x}]
  ```

---

## Solution Approach

### Analysis (Phase II)
By reviewing the git log and the commit history of the stalled PR (#12922), I traced the exact origin of the rendering bug. The root cause in `matplotlib` is a breach of a structural invariant. The code currently attempts to inject all qubits into the `q_indxs` array for zero-operand gates so the renderer has something to draw. However, `q_xy` is contractually obligated to map 1:1 with explicit `qargs`. This hack breaks the coordinate system. In `text.py`, the `bit_indices` loop uses a tautological condition (`x in self.qubits`) and relies on hardcoded string padding that misaligns on circuits with 10+ qubits.

### Phase III correction
The above analysis described the stalled PR's code, which I had mistaken for the *current* upstream code — it isn't merged, so none of that code exists on `main`. The actual current behavior is much simpler and more upstream: zero-qarg nodes never survive `dag.layers()`/`_LayerSpooler` in the first place, so they never even reach `q_indxs`, `_get_coords`, or any rendering loop. I also re-checked PR #12922's actual diff (not just its description) and found it had the same root-cause gap I'd have hit: it patched `_LayerSpooler.__init__` to insert zero-qarg nodes by walking `dag.topological_op_nodes()` and inserting each one **at the very start of the circuit** (`self.insert(0, ...)`). Since topological sort has no edges to order independent (zero-qarg) nodes by, this is exactly the "stacks at the beginning" bug described in my original Phase II notes — so that bug is real, just not yet shipped; it's what *would have* happened had #12922 been merged as-is.

I also checked `unrelated issue #14538` (a different bug about an identity-violating template) where a commenter explicitly avoided using `GlobalPhaseGate` and called it out as having "visualization issues" — useful independent confirmation that this gap is a live, felt pain point blocking other parts of the codebase from using a legitimate API.

### Proposed Solution (revised)
1. Fix the actual root cause in `_get_layered_instructions` (`qiskit/visualization/circuit/_utils.py`): walk `dag.op_nodes()` for zero-operand nodes and re-insert each one as its own dedicated layer, positioned immediately after the layer of the last real instruction that precedes it **in original circuit order** — not via `dag.topological_op_nodes()`, which has no notion of circuit order for nodes with no dependency edges. `DAGOpNode` identity isn't stable across separate `dag.layers()`/`dag.op_nodes()` calls (confirmed empirically — same logical node, different `_node_id` each call), so nodes are matched by their `(qargs, cargs)`, which *is* stable.
2. Render the now-surviving node in each drawer as a box spanning every qubit wire, with no per-wire operand index (per the issue's own suggested resolution). Implemented for **text, matplotlib, and LaTeX** (latex.py was not in my original plan — see Challenges Faced).
3. Reuse each drawer's existing multi-qubit box-drawing code path (`set_qu_multibox`/`_multiqubit_gate`/`_build_multi_gate`) with a `skip_wire_labels` option, rather than writing new, parallel "zero-qubit gate" drawing functions that duplicate that logic (which is what the stalled PR #12922 did, even labeling its own matplotlib version a "hack" in the commit message).

### Implementation Plan (as executed)
1. `_utils.py`: fix `_get_layered_instructions` (see above).
2. `text.py`: render via `set_qu_multibox(..., skip_wire_labels=True)`; fix a latent bug this surfaced (below).
3. `matplotlib.py`: extend `_get_coords`/`_multiqubit_gate` the same way.
4. `latex.py`: extend `_build_latex_array`/`_build_multi_gate` the same way (added once I found it crashed too — see below).
5. Tests for all of the above (see Testing Strategy).
6. Release note (`releasenotes/notes/fix-zero-operand-drawers-5b92cd1d081d2aeb.yaml`).

### Review checklist
- [x] Does it maintain Qiskit's `q_xy <-> qargs` invariant? — Yes; `qargs` itself is never mutated. The drawers compute a *separate* "effective span" only for layout purposes, the same way `_get_gate_span` already does elsewhere in `_utils.py`.
- [x] Do zero-operand gates correctly reserve their own dedicated column? — Yes, verified with circuits containing two consecutive `GlobalPhaseGate`s; each gets its own column, in declaration order.
- [x] Does the text drawer alignment hold up on circuits with >1 qubit, including the single-qubit boundary case? — Yes; see Testing Strategy.

### Evaluate
Ran `reproduce.py` after each drawer's fix; all three drawers now render both `GlobalPhaseGate` calls as distinct, correctly-positioned, full-height boxes with no per-wire numbering, matching the issue's requested resolution.

---

## Testing Strategy

### Unit Tests
All added to the existing Qiskit test suite (not a separate test file), modeled directly on neighboring tests in each file:

- `test/python/visualization/test_utils.py` — 4 new tests on `_get_layered_instructions`:
  - `test_get_layered_instructions_zero_operand_gate` — a `GlobalPhaseGate` lands in its own layer between the real ops before/after it.
  - `test_get_layered_instructions_zero_operand_gates_preserve_order` — two zero-operand gates in a row keep their relative order (don't collapse into one layer).
  - `test_get_layered_instructions_zero_operand_gate_idle_wires_false` — survives `idle_wires=False` pruning.
  - `test_get_layered_instructions_only_zero_operand_gate` — a circuit with *only* a zero-operand instruction still produces one layer.
- `test/python/visualization/test_circuit_text_drawer.py` — 3 new tests, asserting exact string output (same pattern as `test_basic_box`):
  - `test_text_zero_operand_gate` — the full reproduce.py-style scenario.
  - `test_text_zero_operand_gate_single_qubit` — the n=1 boundary case (this is exactly the case that exposed the latent `top_connect=None` bug below).
  - `test_text_only_zero_operand_gate` — only-zero-operand circuit.
- `test/python/visualization/test_circuit_drawer.py` — 1 new test:
  - `test_mpl_zero_operand_gate_does_not_raise` — asserts `circuit.draw("mpl")` doesn't raise and that a patch taller than a single-qubit gate box exists (a real, if intentionally non-pixel-exact, regression check — see note below on why I didn't add a snapshot test here).
- `test/python/visualization/test_circuit_latex.py` — 2 new tests, using the project's existing `assertEqualToReference` text-file-diff mechanism (not image diffing):
  - `test_zero_operand_gate`, `test_zero_operand_gate_single_qubit`, with committed reference `.tex` files generated and reviewed by hand.

All 8 new tests pass. Existing suites: ran all four touched test modules (`test_circuit_text_drawer`, `test_utils`, `test_circuit_drawer`, `test_circuit_latex` — 245 tests total) before and after my changes; the same 4 pre-existing failures occur both before and after (confirmed via `git stash`), and are caused by a stale local Rust build in an unrelated transpiler module (`Optimize1qGatesDecompositionState` missing from the compiled extension) — not by anything in this diff.

### Integration Tests
N/A — this is a self-contained visualization-layer change with no cross-module integration surface beyond the unit tests above.

### Manual Testing
For matplotlib specifically, I did not add a pixel-diff snapshot test to `test/visual/mpl/circuit/test_circuit_matplotlib_drawer.py`, because CONTRIBUTING.md's snapshot-testing process requires generating the reference image via the project's mybinder Jupyter notebook flow (to match the exact matplotlib/font rendering environment CI uses) — I can't do that from a local clone. Instead I:
- Manually rendered and visually inspected `circ.draw('mpl')` for: the full reproduce.py circuit, a single-qubit circuit, an only-zero-operand circuit, `idle_wires=False`, and `justify='right'` — all rendered correctly (box spans all qubits, no per-wire numbers, correct relative position).
- Added the non-snapshot regression test described above, which exercises the actual code path the snapshot test would and would have caught the crash regression I introduced and fixed during development (see Challenges Faced).
- **Follow-up for Phase IV:** add a proper snapshot test + reference image via the mybinder flow before this is PR-ready for final review.

---

## Implementation Notes

### Week 3 Progress (2026-06-15 – 2026-06-19)

**What I built:**
- Root-caused the actual bug (Phase II's theory was wrong — see "Solution Approach" above) by instrumenting `dag.layers()` vs `dag.op_nodes()` directly rather than reasoning from the stalled PR's description.
- `qiskit/visualization/circuit/_utils.py`: added `_insert_zero_operand_layers`, called from `_get_layered_instructions`, to re-insert zero-operand DAG nodes (dropped by `dag.layers()`) into the right layer, matched by `(qargs, cargs)` rather than node identity (which I confirmed empirically is unstable across separate DAG traversals).
- `qiskit/visualization/circuit/text.py`: render zero-operand nodes via `set_qu_multibox(..., skip_wire_labels=True)`; fixed a latent bug in `_set_multibox`'s single-bit path that the new single-qubit test case exposed (it always forwarded `top_connect=None` straight into the box constructor, stomping the class's own default and crashing `.center()`).
- `qiskit/visualization/circuit/matplotlib.py`: extended `_get_coords` (treat a zero-operand node's qubit span as every qubit) and `_multiqubit_gate` (skip per-wire index annotations + their box padding for such nodes).
- `qiskit/visualization/circuit/latex.py`: same pattern in `_build_latex_array`/`_build_multi_gate` — **added after the fact**, once I checked whether my shared `_utils.py` fix had the same crash-regression effect on the LaTeX drawer that it initially did on matplotlib. It did; fixed the same way.
- 8 new unit tests across 4 test files (see Testing Strategy).
- 1 reno release note.
- All formatted/linted clean (`black`, `ruff`) and disclosed as AI-assisted per this repo's `PULL_REQUEST_TEMPLATE.md` requirement (inline comments at each substantively new block, plus full disclosure in the PR description draft below).

**Challenges faced:**
- **DAGOpNode identity is not stable across traversals.** My first attempt at `_insert_zero_operand_layers` built a `{node: index}` dict from `dag.op_nodes()` and tried to look up nodes pulled from `dag.layers()` in it — `KeyError`, every time, even for nodes that were logically "the same." I proved this empirically: the same `h` gate gets a different `_node_id` (and even a different `id(node.op)`) depending on whether it's fetched via `dag.op_nodes()` or via `dag.layers()`'s per-layer subgraphs. Rewrote to match by `(qargs, cargs)` instead, which *is* stable, with a justification for why that's safe (operations sharing identical operands can't be reordered relative to each other, since they conflict on every wire they touch).
- **Fixing matplotlib regressed it from "silently wrong" to "crashes."** Once the layering fix let zero-operand nodes through, matplotlib's `_multiqubit_gate` called `min()` on an empty coordinate list and raised. I caught this only because I re-ran `reproduce.py` after the layering fix instead of assuming the drawers would "just work" once the node was layered correctly. Same exact pattern then bit the LaTeX drawer for the same reason — I went looking for it deliberately the second time.
- **The single-qubit case surfaced a real, pre-existing latent bug**, unrelated to my change in spirit but only reachable because of it: `_set_multibox`'s `len(bit_indices) == 1` path passes `top_connect=None` straight into `BoxOnQuWire`/`BoxOnClWire`, overriding their own sensible default. Every existing caller happened to avoid this combination; mine didn't.
- **Compared my approach against the stalled PR #12922 directly** (its actual file diff, not just its description) at my mentor's prompting. Confirmed it has the exact "all zero-qarg nodes pile up at the start of the circuit" bug my Phase II notes predicted, that it never implemented latex.py's crash fix, that it had zero implemented tests (`pass # TODO!` placeholders only), and that its own matplotlib commit message admits "needs review" for what it calls a hack. This was a useful sanity check that my approach is a genuine improvement, not just different.

### Code Changes
- **Files modified:**
  - `qiskit/visualization/circuit/_utils.py`
  - `qiskit/visualization/circuit/text.py`
  - `qiskit/visualization/circuit/matplotlib.py`
  - `qiskit/visualization/circuit/latex.py`
  - `test/python/visualization/test_utils.py`
  - `test/python/visualization/test_circuit_text_drawer.py`
  - `test/python/visualization/test_circuit_drawer.py`
  - `test/python/visualization/test_circuit_latex.py`
  - `test/python/visualization/references/test_latex_zero_operand_gate.tex` (new)
  - `test/python/visualization/references/test_latex_zero_operand_gate_single_qubit.tex` (new)
  - `releasenotes/notes/fix-zero-operand-drawers-5b92cd1d081d2aeb.yaml` (new)
- **Key commits** (branch: [`fix-zero-operand-drawers`](https://github.com/KevesDev/qiskit/tree/fix-zero-operand-drawers); rebased onto upstream `main` before opening the PR, so these are the final SHAs, not the original Phase III ones):
  - [`05c2558fb`](https://github.com/KevesDev/qiskit/commit/05c2558fbdd66f232aa89cc0610bf32bd84f1793) — Fix zero-operand instructions being dropped from circuit layering
  - [`9a9c8479c`](https://github.com/KevesDev/qiskit/commit/9a9c8479c557b6606c553d58d1aab8c2c162fa3e) — Render zero-operand instructions in the text circuit drawer
  - [`844400b0e`](https://github.com/KevesDev/qiskit/commit/844400b0e436f76248e5f815a7c6db33dcabf0d4) — Render zero-operand instructions in the matplotlib circuit drawer
  - [`5e3f552c3`](https://github.com/KevesDev/qiskit/commit/5e3f552c3fadb35ea09512143f4ac9ddf8b12cf3) — Render zero-operand instructions in the LaTeX circuit drawer
  - [`95e23ef99`](https://github.com/KevesDev/qiskit/commit/95e23ef9963d8a2e21906dbbe974a63629bf55aa) — Add release note for zero-operand drawer fix
  - [`12b9c5e9f`](https://github.com/KevesDev/qiskit/commit/12b9c5e9fe55f4d3fee96d94fe5e53b088f9e7a1) — Remove local-only reproduction script before PR submission (Phase IV)
  - [`4118f4da2`](https://github.com/KevesDev/qiskit/commit/4118f4da245513b62ed9b568809180a9dd201a65) — Exclude classical directives from zero-operand layer insertion (Phase IV)
  - [`e1d04e8c8`](https://github.com/KevesDev/qiskit/commit/e1d04e8c8e97560855d43dbcc064bd5cf78feebb) — Fix self-introduced regression: Store/Var crashes drawers (Phase IV; amended once after pushing to remove a stray reference to this README from the commit message)
- **Approach decisions:**
  - Matched nodes by `(qargs, cargs)` instead of object identity (forced by the DAGOpNode-identity discovery above).
  - Reused each drawer's existing multi-qubit box code via a `skip_wire_labels` flag, instead of writing new parallel "zero-qubit gate" drawing functions (unlike PR #12922) — less code, no duplicated box-drawing logic to keep in sync.
  - Scoped the fix to instructions with *both* zero qargs and zero cargs (not just zero qargs), since a classical-only instruction is already reachable via its clbit wire in `dag.layers()` and doesn't need the same treatment.
  - Did not add a matplotlib snapshot/reference-image test (see Manual Testing) — flagged explicitly as a Phase IV follow-up rather than silently skipped.

### Phase IV Progress (2026-06-24)

**What I did before opening the PR:**
- Ran the full Step 1 pre-submission checklist: re-confirmed the fix still works, re-ran the full test suite, reviewed `git diff upstream/main` line by line.
- That diff review caught `reproduce.py` (a loose root-level script from an earlier commit) as an unrelated/non-idiomatic inclusion. Removed it in its own commit, keeping the original commit in history and re-linking the README's reproduction evidence by commit SHA instead of branch name so the link survives the removal.
- Rebased onto upstream `main` (36 commits ahead at the time) — confirmed first that none of those commits touched any file I'd changed, so the rebase was a clean, conflict-free fast-forward of my changes onto newer history.
- The rebase pulled in Rust-side changes, which meant the locally-built Rust extension was stale (`ImportError: cannot import name 'estimate_fidelity'`). Rebuilt it (`python setup.py build_rust --inplace`, ~70s) — this incidentally *fixed* 2 of the 4 "pre-existing" test failures I'd been tracking since Phase III, confirming they really were a stale-build artifact and not something in my diff.
- The rebuild also **surfaced a real bug my fix introduced**: a classical `Store` instruction on a `Var` (used in control-flow tests with standalone variables) also has zero qargs and zero cargs, so my generic "zero-operand" filter caught it too and tried to insert it into a dedicated layer — crashing the text drawer's barrier-handling code (`min()` on empty `qargs`), which had never had to handle a zero-qarg directive before. The correct fix isn't to render `Store` (drawers have never shown it, and shouldn't start now — that's out of scope for #9962) but to exclude directives from the zero-operand layering fix entirely, so they stay exactly as invisible as before. Initially fixed only the *insertion* path in [`4118f4da2`](https://github.com/KevesDev/qiskit/commit/4118f4da245513b62ed9b568809180a9dd201a65); my first instinct that the LaTeX drawer's matching crash was a separate, pre-existing bug turned out to be wrong.
- Before considering this done, verified rigorously rather than trusting a `git stash` check on a partially-committed branch: built unmodified upstream `main` from scratch in an isolated git worktree and confirmed the `Store`/`Var` repro does **not** crash there. That proved the bug was introduced by my own fix, not pre-existing. Root cause: a separate filter line in `_get_layered_instructions` (which I'd changed to stop dropping real zero-operand gates) was also rescuing `Store` from being filtered out, since `Store` reaches that code through its own pre-existing path, independent of my insertion logic. Fixed by extracting one shared predicate used consistently by both code paths, in [`e1d04e8c8`](https://github.com/KevesDev/qiskit/commit/e1d04e8c8e97560855d43dbcc064bd5cf78feebb), pushed before any maintainer review.
- Caught one more thing on a final pass: that fix's first commit message referenced this README by name, which doesn't mean anything to a Qiskit reviewer and has no business in a public commit history. Amended the message to stand on its own as a purely technical explanation and force-pushed again. Worth remembering for next time: keep the two audiences (this README vs. the actual project) separate from the first draft, not just on review.
- Force-pushed the rebased branch with `--force-with-lease`.
- Picked a reviewer data-driven rather than by guessing: `git log` on the exact files I changed showed Edwin Navarro (`@enavarro51`) as the single most frequent author (17 commits vs. the next-highest at 8) — confirmed his GitHub handle by checking one of his commits directly.
- Opened the PR and left a review-request comment tagging `@enavarro51`, following the project's own `PULL_REQUEST_TEMPLATE.md` (filled in the AI/LLM disclosure checkboxes naming Claude Code/Sonnet 4.6, since this contribution was built with heavy AI assistance throughout, which I reviewed and take responsibility for).

---

## Pull Request

**PR Link:** https://github.com/Qiskit/qiskit/pull/16453

**PR Description:**
What does this PR do?: Renders zero-operand instructions (most commonly a stand-alone `GlobalPhaseGate`) in the text, matplotlib, and LaTeX circuit drawers, as a box spanning every qubit wire with no per-wire operand index, positioned in original circuit order.

Why was this PR needed?: All three drawers were silently dropping these instructions entirely, with no error — `circ.draw('text')` would simply omit a `GlobalPhaseGate` call as if it had never been appended. Investigation revealed the root cause is upstream of all three drawers: `_get_layered_instructions` builds its layers from `dag.layers()`, which only visits DAG nodes reachable via wires, and a zero-operand instruction has none.

What are the relevant issue numbers?: Fixes #9962

Does this PR meet the acceptance criteria?:
- [x] Tests added for new/changed behavior (8 new unit tests, 4 files)
- [x] All tests passing (245/245 locally, full suite of touched modules)
- [x] Follows project style guide (`black`, `ruff` both clean)
- [x] No breaking changes introduced (purely additive — previously-invisible instructions now render)
- [ ] Documentation updated (n/a beyond the reno release note; no public API changed)

**Maintainer Feedback:**
- 2026-06-24: PR opened, requested review from @enavarro51 (most frequent contributor to the exact files changed, by git history).
- 2026-06-24: Before any maintainer response, found and fixed a regression in my own change (see Phase IV Progress above) and pushed the fix with a comment explaining it. No maintainer response yet.

**Status:** Awaiting review.

---

## Learnings & Reflections

### Technical Skills Gained
- How to read and debug a Rust-backed Python data structure (`DAGCircuit`) from the Python side — specifically, that wrapper-object identity/equality for `DAGOpNode` is not something you can rely on across separate accessor calls, even when "logically" referring to the same node.
- The internal layering model Qiskit's circuit drawers share (`_get_layered_instructions` → `_LayerSpooler` → per-drawer rendering), and how a gap in the shared layer ought to be fixed once, centrally, rather than patched around in each of the three drawers separately.
- How to extend existing multi-qubit box-drawing code via an explicit, narrowly-scoped option flag instead of writing parallel one-off drawing functions for a new case.

### Challenges Overcome
See "Challenges faced" above — the DAGOpNode identity issue and the two crash regressions (matplotlib, then LaTeX) were the two biggest. Both were caught by re-running the actual reproduction script after each change rather than trusting that a fix in one layer would "just work" everywhere it mattered.

### What I'd Do Differently Next Time
I'd check all three drawers for the crash-regression pattern in one pass right after the `_utils.py` fix, rather than discovering the LaTeX one separately, later, only because it was specifically prompted. The pattern ("does anything downstream of this shared fix assume non-empty `qargs`?") was identifiable in advance. The same lesson repeated itself in Phase IV with the `Store`/`Var` directive case — a broad, generic filter (any zero-qarg, zero-carg node) caught more than I'd designed for. Next time I'd enumerate the actual *categories* of zero-operand node (real gates vs. directives) up front, rather than discovering the second category by accident during a routine rebuild.

### Phase IV-specific learning
Rebasing onto 36 upstream commits and rebuilding the Rust extension wasn't just chore work — it changed the actual behavior of my code (by exposing the `Store`/directive case) in a way that purely re-running my own existing tests on a stale build never would have. The pre-submission checklist's "rebase on latest upstream" step isn't just about avoiding merge conflicts; it's a real opportunity to catch interactions with code that didn't exist when you wrote your fix.

---

## Resources Used
- [Qiskit `dagcircuit` source](https://github.com/Qiskit/qiskit/blob/main/crates/circuit/src/dag_circuit.rs) — for understanding why `dag.layers()` is wire-reachability-based.
- Issue #14538 and its closing PR #15944 — independent confirmation that avoiding `GlobalPhaseGate` due to "visualization issues" is a real, felt workaround elsewhere in the codebase.
- Stalled PR #12922 (full diff, not just description) — useful negative example: showed the naive layering fix, the no-tests gap, and the matplotlib "hack" to avoid repeating.
