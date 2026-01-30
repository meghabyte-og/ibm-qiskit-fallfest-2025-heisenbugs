# BCC Protein Folding on a Body-Centered Cubic Lattice
## QUBO Construction, Constraint Encoding, and One-Hot Simulated Annealing

---

## Overview

This repository implements a **body-centered cubic (BCC) lattice protein folding model**
with an explicit **QUBO (Quadratic Unconstrained Binary Optimization)** formulation.

The project focuses on **correct geometric modeling**, **explicit constraint encoding**,
and **optimization-ready problem construction**, suitable for classical and quantum
optimization workflows (via Qiskit).

This is a **technical, interview-grade codebase** intended to demonstrate:
- Discrete lattice geometry modeling
- Binary optimization design (QUBO / Ising form)
- Constraint engineering with penalties
- Hybrid classical–quantum interfaces

---

## Lattice Geometry

- Lattice type: **Body-Centered Cubic (BCC)**
- Allowed steps (8 body diagonals):

  (+1,+1,+1), (+1,+1,-1), (+1,-1,+1), (+1,-1,-1),
  (-1,+1,+1), (-1,+1,-1), (-1,-1,+1), (-1,-1,-1)

- Non-backtracking bond angles:
  - 70.53 degrees
  - 109.47 degrees

These angles arise from dot products of BCC direction vectors.

---

## Chain Representation

- Protein length: `N` residues
- Each residue occupies exactly **one lattice position**
- Consecutive residues must be **BCC neighbors**
- No two residues may occupy the same lattice site (self-avoidance)

---

## Encodings

### 1. Turn Encoding

Each bond is encoded as:
- An integer direction index in `[0, 7]`
- A 3-bit binary code (one register per bond)

Properties:
- Immediate backtracking is forbidden via XOR symmetry
- Exact reconstruction of Cartesian coordinates
- Explicit bond-angle computation

---

### 2. Coordinate Encoding

- Absolute lattice coordinates reconstructed from turns
- Enables:
  - Self-avoidance checks
  - Contact detection
  - Energy evaluation

---

## Energy Models

### HP Model

Binary hydrophobic–polar model:

- H–H contact energy = `-1.0`
- All other contacts = `0.0`
- Only non-consecutive lattice neighbors are counted

---

### MJ96 Model

- Full Miyazawa–Jernigan (1996) Table 3
- 20×20 symmetric contact energy matrix
- Real-valued contact energies (RT units)
- Applied only to non-consecutive BCC neighbors

---

## QUBO Formulation

### Binary Variables

For residue `i` and lattice position `p`:

```
x[i,p] = 1  if residue i occupies position p
x[i,p] = 0  otherwise
```

Total number of binary variables:

```
N × |P|
```

where `P` is the set of lattice positions reachable in `N-1` steps.

---

## Constraints (Penalty Terms)

### 1. One-Hot Constraint

Each residue must occupy **exactly one** lattice position.

Penalty term:

```
(sum over p of x[i,p] - 1)^2
```

This enforces both:
- At least one position selected
- At most one position selected

---

### 2. Adjacency Constraint

Consecutive residues must be lattice neighbors.

For any non-neighboring positions `p` and `q`:

```
x[i,p] * x[i+1,q] = 0
```

Violations incur a positive penalty.

---

### 3. Collision Constraint (Self-Avoidance)

No two residues may occupy the same lattice site.

For all residues `i != j` and any position `p`:

```
x[i,p] * x[j,p] = 0
```

This globally enforces excluded volume.

---

## Objective Function

The QUBO minimizes the following total energy:

```
Total Energy =
  λ_onehot    * one-hot penalty
+ λ_adj       * adjacency penalty
+ λ_collision * collision penalty
+ contact energy (HP or MJ)
```

Penalty weights must dominate contact energies to ensure feasibility.

---

## Optimization Strategy

### One-Hot Preserving Simulated Annealing

A custom simulated annealer is implemented with the following properties:

- Always maintains one-hot feasibility
- Moves one residue at a time
- Supports:
  - Local neighbor moves
  - Occasional random jumps (ergodicity)
- Efficient Δ-energy updates using sparse QUBO structure

Algorithm outline:
1. Initialize from a random self-avoiding walk
2. Anneal temperature from `T_start` to `T_end`
3. Propose block moves per residue
4. Accept/reject using Metropolis criterion
5. Track best feasible solution

---

## Qiskit Integration

- QUBO export to `qiskit_optimization.QuadraticProgram`
- Compatible with:
  - QAOA
  - VQE
  - Classical optimizers
- Turn encoding allocates:
  - 3 qubits per bond
  - Named registers: `b0`, `b1`, `b2`, ...

---

## Example Scale

From the reference notebook:

- Sequence length: 9
- Reachable lattice positions: 1241
- Binary variables: 11,169
- Nonzero QUBO terms: ~19.4 million
- QuadraticProgram variables: 11,169

---

## Limitations

- QUBO size grows rapidly with sequence length
- No advanced polymer moves (pivot, crankshaft)
- Not intended for real protein structure prediction
- MJ table embedded for clarity, not learned

---

## References

- Miyazawa & Jernigan (1996), residue–residue potentials
- Lau & Dill (1989), HP lattice protein model
- Lucas (2014), Ising formulations of NP problems
- Qiskit Optimization documentation
