# Contribution 1: Support zero-operand instructions in circuit drawers

**Contribution Number:** 1 
**Student:** Stan Riane Nelson  
**Issue:** https://github.com/Qiskit/qiskit/issues/9962  
**Status:** Phase II Complete

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

### Affected Components
- `qiskit/visualization/circuit/matplotlib.py` (specifically `_get_coords` and `_zero_qubit_gate`)
- `qiskit/visualization/circuit/text.py` (specifically `_set_zero_qubit_operandbox`)
- `NodeData` class (coordinate storage)

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
- **Screenshots/logs:** ![Matplotlib Bug](https://github.com/KevesDev/su26-ai301-contribution/blob/main/mpl_bug.png?raw=true)
- **My findings:** Confirmed that the upstream layer assignment (`_LayerSpooler`) works perfectly, meaning the core engine is sound. The bug is localized entirely to the rendering loops in the `text` and `matplotlib` visualization modules.

---

## Solution Approach

### Analysis
By reviewing the git log and the commit history of the stalled PR (#12922), I traced the exact origin of the rendering bug. The root cause in `matplotlib` is a breach of a structural invariant. The code currently attempts to inject all qubits into the `q_indxs` array for zero-operand gates so the renderer has something to draw. However, `q_xy` is contractually obligated to map 1:1 with explicit `qargs`. This hack breaks the coordinate system. In `text.py`, the `bit_indices` loop uses a tautological condition (`x in self.qubits`) and relies on hardcoded string padding that misaligns on circuits with 10+ qubits.

### Proposed Solution
Instead of mutating the active operand arrays, we will explicitly calculate the bounding top and bottom wire coordinates for zero-operand gates and store them safely in new, dedicated attributes. The rendering functions will then be updated to read from these new attributes, preserving the core engine invariants.

### Implementation Plan
Using UMPIRE framework:

**Understand:** Zero-operand gates need a visual span across all wires without breaking data invariants tied to explicit operands.

**Match:** The `latex.py` drawer handles this gracefully by deriving coordinates directly from `self._qubits` rather than `node.qargs`. We will replicate this isolation pattern in `matplotlib` and `text`.

**Plan:**
1. Modify `NodeData.__init__` in `matplotlib.py` to add `zero_wire_top` and `zero_wire_bot` variables.
2. Update `_get_coords` in `matplotlib.py` to safely compute these boundary coordinates for zero-operand nodes without modifying `q_xy` or `q_indxs`.
3. Rewrite `_zero_qubit_gate` in `matplotlib.py` to draw the spanning box using the new attributes.
4. Update `Layer._set_zero_qubit_operandbox` in `text.py` to iterate explicitly over `range(len(self.qubits))` and dynamically calculate padding (`wire_label_len`) to fix the visual borders.

**Implement:** [Link to branch/commits as I work in Phase III]

**Review:** - [ ] Does it maintain Qiskit's `q_xy <-> qargs` invariant?
- [ ] Do zero-operand gates correctly reserve their own width/column?
- [ ] Does the text drawer alignment hold up on circuits with >10 qubits?

**Evaluate:** Execute the local reproduction script to verify consecutive `GlobalPhaseGate`s occupy distinct columns and render clean, connected boxes in both drawers.

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
