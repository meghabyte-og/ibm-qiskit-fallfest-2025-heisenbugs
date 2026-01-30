# BCC Protein Folding on a Body-Centered Cubic Lattice  
**QUBO Construction, Constraint Encoding, and One-Hot Simulated Annealing**

---

## Overview

This repository implements a **body-centered cubic (BCC) lattice protein folding model** with explicit **QUBO (Quadratic Unconstrained Binary Optimization)** construction suitable for classical and quantum optimization workflows.

The project demonstrates:
- Exact geometric encoding of self-avoiding protein chains on a BCC lattice
- Turn-based and coordinate-based encodings
- Constraint-preserving QUBO formulations (one-hot, adjacency, collision)
- HP and Miyazawa–Jernigan (MJ96) contact energy models
- Export to `qiskit_optimization.QuadraticProgram`
- A custom **block-move simulated annealer** that maintains one-hot feasibility
---

## Mathematical Model

### Lattice
- **Geometry**: 3D Body-Centered Cubic (BCC)
- **Step set**: 8 body-diagonal directions  
  \[
  (\pm1, \pm1, \pm1)
  \]
- **Angle spectrum** (non-backtracking):
  - 70.53°
  - 109.47°

### Chain Representation
- Protein of length \(N\)
- Each residue occupies exactly one lattice position
- Consecutive residues must be BCC neighbors
- Self-avoidance enforced globally

---

## Encodings

### 1. Turn Encoding
Each bond is encoded by:
- **Index** \( \in \{0,\dots,7\} \)
- **3-bit representation** (one register per bond)

Features:
- Backtracking forbidden via XOR symmetry
- Exact reconstruction of Cartesian coordinates
- Angle computation between successive bonds

### 2. Coordinate Encoding
- Absolute lattice positions reconstructed from turns
- Enables:
  - Self-avoidance checks
  - Contact detection
  - Energy evaluation

---

## Energy Models

### HP Model
- Binary hydrophobic/polar mapping
- Energy:
  \[
  E_{ij} =
  \begin{cases}
  -1 & \text{H–H contact} \\
  0 & \text{otherwise}
  \end{cases}
  \]

### MJ96 Model
- Full **Miyazawa–Jernigan (1996) Table 3**
- 20×20 symmetric contact energy matrix
- Real-valued pairwise energies (RT units)
- Only applied to non-consecutive BCC neighbors

---

## QUBO Formulation

### Variables
Binary variable:
\[
x_{i,p} =
\begin{cases}
1 & \text{residue } i \text{ occupies position } p \\
0 & \text{otherwise}
\end{cases}
\]

Total variables:
\[
N \times |\mathcal{P}|
\]
where \(\mathcal{P}\) is the set of reachable lattice positions in \(N-1\) steps.

---

### Constraints (Penalty Terms)

#### 1. One-Hot Constraint
Each residue occupies exactly one position:
\[
\left(\sum_p x_{i,p} - 1\right)^2
\]

#### 2. Adjacency Constraint
Consecutive residues must be neighbors:
\[
x_{i,p} \cdot x_{i+1,q} = 0 \quad \text{if } q \notin \mathcal{N}(p)
\]

#### 3. Collision Constraint
No two residues share a position:
\[
x_{i,p} \cdot x_{j,p} = 0 \quad (i \neq j)
\]

---

### Objective Function

\[
\min \left(
\lambda_{\text{onehot}} P_{\text{onehot}} +
\lambda_{\text{adj}} P_{\text{adj}} +
\lambda_{\text{coll}} P_{\text{coll}} +
\sum_{i<j} E_{ij}^{\text{contact}}
\right)
\]

Penalty weights are tunable and must dominate contact energies.

---

## Optimization Strategy

### One-Hot Preserving Simulated Annealing

Key properties:
- Maintains feasibility at all times
- Moves residues between lattice positions
- Supports:
  - Local neighbor moves
  - Occasional random jumps (ergodicity)
- Efficient Δ-energy updates using sparse QUBO structure

Algorithm:
1. Initialize from a random self-avoiding walk
2. Iterate annealing schedule
3. Perform block moves per residue
4. Accept/reject via Metropolis criterion
5. Track best solution

---

## Qiskit Integration

- QUBO exported as `QuadraticProgram`
- Fully compatible with:
  - QAOA
  - VQE
  - Classical optimizers via Qiskit Optimization
- Turn encoding allocates:
  - 3 qubits per bond
  - Explicit register naming (`b0`, `b1`, …)

---

## Example Scale (From Notebook)

Model: MJ
Sequence length (N): 9
Reachable positions: 1241
Binary variables: 11,169
Nonzero QUBO terms: ~19.4 million
QuadraticProgram variables: 11,169


---

## Repository Structure (Conceptual)



bcc_lattice/
├── lattice.py # BCC geometry, directions, angles
├── encoding.py # Turn & coordinate encodings
├── energy.py # HP & MJ energy models
├── qubo.py # QUBO construction
├── annealing.py # One-hot simulated annealer
├── qiskit_export.py # QuadraticProgram interface
└── notebook.ipynb # Full experimental pipeline


---

## Intended Use

- Research prototyping of lattice protein folding
- Benchmarking QUBO construction strategies
- Hybrid classical–quantum workflows

---

## Limitations

- QUBO size grows rapidly with sequence length
- No advanced polymer moves (pivot, crankshaft)
- Not intended for realistic protein prediction
- MJ table embedded for convenience (not learned)

---

## References

- Miyazawa, S. & Jernigan, R. L. (1996)  
  *Residue–residue potentials with a favorable contact energy interpretation*
- Lau & Dill (1989) HP lattice protein model
- Lucas (2014) Ising formulations of NP problems
- Qiskit Optimization documentation

