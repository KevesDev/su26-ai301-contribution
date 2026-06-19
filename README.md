# Contribution 1: Support zero-operand instructions in circuit drawers

**Contribution Number:** 1 
**Student:** Stan Riane Nelson  
**Issue:** https://github.com/Qiskit/qiskit/issues/9962  
**Status:** Phase III Complete

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
- **Commit showing reproduction:** https://github.com/KevesDev/qiskit/blob/fix-zero-operand-drawers/reproduce.py
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
- **Key commits** (branch: [`fix-zero-operand-drawers`](https://github.com/KevesDev/qiskit/tree/fix-zero-operand-drawers)):
  - [`b26dabe3f`](https://github.com/KevesDev/qiskit/commit/b26dabe3f3fc603855fd8b3bc22b5fe6063f6857) — Fix zero-operand instructions being dropped from circuit layering
  - [`91dc3f5a1`](https://github.com/KevesDev/qiskit/commit/91dc3f5a1fa4e78de18cad07c53a8c3917f0c736) — Render zero-operand instructions in the text circuit drawer
  - [`910642f56`](https://github.com/KevesDev/qiskit/commit/910642f565a676813d309f20d1e9d35f73b7ab7e) — Render zero-operand instructions in the matplotlib circuit drawer
  - [`5db983e11`](https://github.com/KevesDev/qiskit/commit/5db983e1187d9d1f4cc17c105ce0604f349cf126) — Render zero-operand instructions in the LaTeX circuit drawer
  - [`db78fe6fd`](https://github.com/KevesDev/qiskit/commit/db78fe6fdc13d0ec294f73aa2bbfec5418e0109b) — Add release note for zero-operand drawer fix
- **Approach decisions:**
  - Matched nodes by `(qargs, cargs)` instead of object identity (forced by the DAGOpNode-identity discovery above).
  - Reused each drawer's existing multi-qubit box code via a `skip_wire_labels` flag, instead of writing new parallel "zero-qubit gate" drawing functions (unlike PR #12922) — less code, no duplicated box-drawing logic to keep in sync.
  - Scoped the fix to instructions with *both* zero qargs and zero cargs (not just zero qargs), since a classical-only instruction is already reachable via its clbit wire in `dag.layers()` and doesn't need the same treatment.
  - Did not add a matplotlib snapshot/reference-image test (see Manual Testing) — flagged explicitly as a Phase IV follow-up rather than silently skipped.

---

## Pull Request

**PR Link:** Not yet opened — per the Phase III instructions ("Move directly to Phase IV... where you'll open a pull request"), I'm holding off on opening the actual PR against `Qiskit/qiskit` until Phase IV, but the description below is ready to use as-is.

**PR Description (draft, ready for Phase IV):**

> ### Summary
> Fixes #9962. Zero-operand instructions (most commonly a stand-alone `GlobalPhaseGate`) were being silently dropped by the text and matplotlib circuit drawers, and were never supported by the LaTeX drawer either — none of the three would show them at all, with no error. The root cause is upstream of all three drawers: `dag.layers()` (consumed by `_get_layered_instructions`/`_LayerSpooler`) only ever visits nodes reachable by walking the DAG's wires, and a zero-operand instruction has none.
>
> This PR:
> - Fixes `_get_layered_instructions` in `qiskit/visualization/circuit/_utils.py` to re-insert zero-operand nodes into their correct layer, in original circuit order, matched by `(qargs, cargs)` rather than node identity (`DAGOpNode` identity/equality isn't stable across separate `dag.layers()`/`dag.op_nodes()` calls).
> - Renders each such node in the text, matplotlib, and LaTeX drawers as a box spanning every qubit wire, with no per-wire operand index (per the resolution proposed in the issue), by extending each drawer's existing multi-qubit box-drawing code with a `skip_wire_labels` option rather than adding new parallel drawing functions.
> - Fixes a latent bug in `text.py`'s `_set_multibox` (single-bit path forwarded `top_connect=None` into the box constructor, overriding its sensible default) that the new single-qubit test case exposed.
> - Adds 8 unit tests across `test_utils.py`, `test_circuit_text_drawer.py`, `test_circuit_drawer.py`, and `test_circuit_latex.py`.
>
> A prior attempt at this issue (#12922) stalled in draft with no tests; its `_LayerSpooler` patch would have inserted every zero-operand node at the very start of the circuit regardless of where it was declared (using `dag.topological_op_nodes()`, which has no ordering for nodes with no dependency edges), and its own commit message called the matplotlib portion "a hack, needs review." This PR fixes the same underlying gap with original-circuit-order placement, reuses existing box-drawing code instead of duplicating it, and includes tests.
>
> **Known follow-up (not blocking):** no matplotlib snapshot/reference-image test was added; per CONTRIBUTING.md, generating one correctly requires the project's mybinder snapshot-testing notebook, which I'll do as part of addressing review feedback.
>
> Fixes #9962
>
> ### AI/LLM disclosure
> - [x] I used the following tool to help write this PR description: Claude Code (Claude Sonnet 4.6)
> - [x] I used the following tool to generate or modify code: Claude Code (Claude Sonnet 4.6)

**Maintainer Feedback:**
- N/A yet — PR not opened.

**Status:** Ready to open in Phase IV.

---

## Learnings & Reflections

### Technical Skills Gained
- How to read and debug a Rust-backed Python data structure (`DAGCircuit`) from the Python side — specifically, that wrapper-object identity/equality for `DAGOpNode` is not something you can rely on across separate accessor calls, even when "logically" referring to the same node.
- The internal layering model Qiskit's circuit drawers share (`_get_layered_instructions` → `_LayerSpooler` → per-drawer rendering), and how a gap in the shared layer ought to be fixed once, centrally, rather than patched around in each of the three drawers separately.
- How to extend existing multi-qubit box-drawing code via an explicit, narrowly-scoped option flag instead of writing parallel one-off drawing functions for a new case.

### Challenges Overcome
See "Challenges faced" above — the DAGOpNode identity issue and the two crash regressions (matplotlib, then LaTeX) were the two biggest. Both were caught by re-running the actual reproduction script after each change rather than trusting that a fix in one layer would "just work" everywhere it mattered.

### What I'd Do Differently Next Time
I'd check all three drawers for the crash-regression pattern in one pass right after the `_utils.py` fix, rather than discovering the LaTeX one separately, later, only because it was specifically prompted. The pattern ("does anything downstream of this shared fix assume non-empty `qargs`?") was identifiable in advance.

---

## Resources Used
- [Qiskit `dagcircuit` source](https://github.com/Qiskit/qiskit/blob/main/crates/circuit/src/dag_circuit.rs) — for understanding why `dag.layers()` is wire-reachability-based.
- Issue #14538 and its closing PR #15944 — independent confirmation that avoiding `GlobalPhaseGate` due to "visualization issues" is a real, felt workaround elsewhere in the codebase.
- Stalled PR #12922 (full diff, not just description) — useful negative example: showed the naive layering fix, the no-tests gap, and the matplotlib "hack" to avoid repeating.
