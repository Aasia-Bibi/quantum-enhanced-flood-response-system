
# Q-Khez: Quantum-Enhanced Flood Risk Prediction and Emergency Shelter Placement Optimization in Punjab

A hybrid classical-quantum system that supports disaster management through **flood-risk prediction** and **optimized emergency shelter placement**, using a case study of **Punjab, Pakistan**.

The project explores how classical machine learning and quantum computing can be combined to improve flood-risk assessment and support decision-making during flood emergencies.


---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Project Objectives](#project-objectives)
- [System Architecture](#system-architecture)
- [Project Modules](#project-modules)
- [Results](#results)
- [Repository Structure](#repository-structure)
- [Technologies and Tools](#technologies-and-tools)
- [Installation](#installation)
- [Usage](#usage)
- [Limitations](#limitations)
- [Future Work](#future-work)
- [Team](#team)
- [License](#license)

---

## Overview

Floods cause significant human, economic, and environmental losses. Effective disaster response requires two things: accurate, timely flood-risk prediction, and strategic placement of emergency shelters. This repository implements a two-stage hybrid pipeline:

1. **Flood Prediction**: Classical SVM (RBF kernel) vs. Quantum SVM (9-qubit entangled feature map) to classify flood susceptibility.
2. **Shelter Placement Optimization**: Shelter siting formulated as a QUBO problem and solved with the Quantum Approximate Optimization Algorithm (QAOA), with a classical 2-opt local search as baseline.

The project investigates the potential of quantum computing in disaster-response applications. **It does not claim a proven practical quantum speedup.**

## Problem Statement

Traditional approaches can face challenges when handling complex prediction tasks and large-scale optimization problems. Q-Khez explores hybrid classical and quantum techniques to address these challenges in flood-prone regions such as Punjab.

## Project Objectives

- Predict flood risk using machine learning and quantum machine learning approaches.
- Explore quantum-enhanced models for flood prediction.
- Optimize emergency shelter placement using quantum optimization techniques.
- Support disaster-response planning through data-driven decision-making.
- Compare classical and quantum-based approaches where applicable.

## System Architecture

```
            ┌──────────────────────────┐
            │  flood_preprocessed.csv  │
            └────────────┬─────────────┘
                         │
        ┌────────────────┴────────────────┐
        ▼                                 ▼
 Stage 1: Flood Prediction        Stage 2: Shelter Optimization
 - Feature selection (top 9)      - 6x6 grid of 36 Punjab districts
 - Classical SVM (RBF)            - QUBO formulation (K = 4 shelters)
 - Quantum SVM (9-qubit ZZ+RYY)   - QAOA (p = 3)  |  2-opt baseline
        │                                 │
        └────────────────┬────────────────┘
                         ▼
          Flood-risk classification + optimized shelter locations
```

## Project Modules

### 1. Flood Prediction

The Flood Prediction module identifies flood susceptibility from historical, environmental, and infrastructural indicators.

**Dataset:** `Dataset/flood_preprocessed.csv`
- Preprocessed flood dataset with 20 numerical features, including `ClimateChange`, `PoliticalFactors`, `WetlandLoss`, `DamsQuality`, `Landslides`, `Watersheds`, and `MonsoonIntensity`.
- Target variable: `FloodRisk`.

**Feature Selection:** Absolute Pearson correlation with the target is used to select the top 9 features, matching the 9-qubit quantum register.

**Quantum Preprocessing:** Quantum kernel estimation requires O(N²) circuit executions, so a balanced subset of 100 samples (80 train / 20 test) is used for the quantum model. Features are standardized with `StandardScaler` and clipped to [−π, π] for rotational gate encoding.

**Models explored:**
- Support Vector Classifier (SVC, RBF kernel), classical baseline
- Quantum Support Vector Classifier (QSVC) with a custom 9-qubit kernel
- Random Forest (RF)
- Variational Quantum Classifier (VQC)

**Quantum Kernel Design:** A custom 9-qubit, 2-layer data re-uploading feature map (depth 60, 105 gates) projects features into a 512-dimensional Hilbert space:
1. **Encoding:** Hadamard gates create superposition, followed by `Rz(2·xᵢ)` rotations.
2. **ZZ entanglement:** Linear, long-range, and cross-qubit `CNOT–Rz(2·xᵢ·xⱼ)–CNOT` blocks capture pairwise correlations.
3. **Re-uploading and RYY entanglement:** A second `Rz` layer followed by `CNOT–Ry(2·xᵢ·xⱼ)–CNOT` blocks.
4. **Kernel matrix:** Computed with compute-uncompute circuits `U(xᵢ)U†(xⱼ)` on the Qiskit Aer statevector simulator, reading out P(|000000000⟩).

**Expected Output:** A predicted flood-risk class for given input features.

### 2. Shelter Placement Optimization

The Shelter Placement module identifies suitable locations for emergency shelters, considering population coverage and flood risk.

**Problem model:** 36 Punjab districts on a 6×6 spatial grid (total population 96.6 million). Select exactly K = 4 shelters to maximize population coverage.

**Formulation (QUBO):** With binary variables xᵢ ∈ {0, 1} (1 = shelter placed in district i):

- **Objective:** maximize population coverage within a Manhattan radius of 2, penalizing double-counted overlap.
- **Constraint 1, exact budget:** (Σᵢ xᵢ − 4)², penalty λ₁ = 10⁶
- **Constraint 2, safe zones:** shelters only in districts with flood risk ≤ 0.55, penalty λ₂ = 10⁶ (implemented by reducing the register from 36 to 22 qubits)
- **Constraint 3, no adjacent shelters:** penalty λ₃ = 10⁵ on orthogonally adjacent pairs

**Why QAOA:** Choosing 4 of 22 safe zones gives C(22, 4) = 7,315 valid combinations inside a 2³⁶ (~68.7 billion) unconstrained space. The objective maps directly to an Ising Hamiltonian, and QAOA uses shallow parameterized circuits that suit NISQ hardware. This implementation uses p = 3 layers, optimized classically with COBYLA.

**Expected Output:** An optimized set of shelter locations for the defined problem instance.

## Results

### Stage 1: Classical SVM vs. Quantum SVM

| Metric | Classical SVM (RBF) | Quantum SVM (9-qubit) |
|---|---|---|
| Training samples | 28,000 | 80 |
| Test samples | 12,000 | 20 (balanced 10/10) |
| Accuracy | **83.78%** | 60.00% |
| Precision | **0.9951** | 0.6000 |
| Recall | **0.8391** | 0.6000 |
| F1-score | **0.9105** | 0.6000 |
| Total compute time | **13.62 s** | 394.44 s |

Classical SVM outperforms the quantum model on this instance. The quantum model was limited to 80 training samples by the cost of classical simulation of the kernel matrix. Kernel values were also concentrated around a mean of 0.0147, which suggests expressibility-induced barren plateaus in the 2-layer ansatz.

### Stage 2: QAOA Shelter Placement

Across 150 optimization trials on the 36-district grid, all hard constraints were satisfied (4/4 shelters placed, 0 unsafe placements, 0 adjacent pairs).

| Metric | Value |
|---|---|
| Population coverage | 92.3% (89.2M / 96.6M) |
| Average shelter flood risk | 0.445 |
| Runtime | 0.752 s |

**Selected shelter locations:**

| # | District | Grid Coord | Population | Flood Risk | Status |
|---|---|---|---|---|---|
| 1 | Gujranwala | (0, 1) | 4.5M | 0.40 | Safe, non-adjacent |
| 2 | Faisalabad | (1, 4) | 7.5M | 0.38 | Safe, non-adjacent |
| 3 | Sargodha | (2, 0) | 3.2M | 0.55 | Safe, non-adjacent |
| 4 | Hafizabad | (5, 2) | 1.2M | 0.45 | Safe, non-adjacent |

Generated figures are stored in `images/`, including the classical vs. quantum comparison, confusion matrix comparison, shelter placement map, and Punjab QAOA optimization output.

## Repository Structure

```
Qiskit/
├── Dataset/
│   └── flood_preprocessed.csv          # Preprocessed flood dataset (20 features)
├── images/                             # Result plots and figures
│   ├── classical vs qtm image.png
│   ├── confusion_matrix_comparison...
│   ├── flood_risk_shelter_placement...
│   ├── image.png
│   ├── output 1.png
│   ├── output 2.png
│   ├── output 3.png
│   ├── punjab_qaoa_shelter_optimiz...
│   ├── quantum_vs_classical_compar...
│   └── quantum_vs_classical_detailed...
├── Flood_Top4Features_CustomZZ...      # Feature selection and custom ZZ feature map code
├── qaoa.ipynb                          # Main notebook: QSVM and QAOA shelter optimization
├── requirements.txt                    # Python dependencies
└── README.md
```

## Technologies and Tools

- Python
- Machine Learning (scikit-learn)
- Quantum Machine Learning
- Qiskit, Qiskit Aer, Qiskit Optimization, Qiskit Algorithms
- QAOA
- NumPy
- Pandas

## Installation

Clone the repository and install dependencies:

```bash
git clone <YOUR_GITHUB_REPOSITORY_URL>
cd Qiskit
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

## Usage

Open the main notebook and run the cells in order:

```bash
jupyter notebook qaoa.ipynb
```

1. Load `Dataset/flood_preprocessed.csv`.
2. Run feature selection to obtain the top 9 features.
3. Train and evaluate the classical SVM and the quantum SVM.
4. Run the QAOA shelter placement solver to get the optimized shelter locations.

## Limitations

1. **Classical simulation bottleneck (QSVM):** Simulating the 9-qubit kernel took 394.44 s for 100 samples, which limited training to 80 samples (0.28% of the classical training set).
2. **Statevector scaling (QAOA):** Simulating all 36 districts as qubits needs 2³⁶ amplitudes (over 512 GB of RAM). The solver therefore pre-filters to 22 safe-zone qubits (~64 MB statevector). The notebook's `ProperQAOASolver` also uses a randomized 2-opt local search with constraint repair as a classical baseline.
3. **Coverage approximation:** Exact set-union coverage across overlapping shelters needs auxiliary slack qubits. Pairwise quadratic overlap penalties are used as a hardware-efficient approximation.
4. **Spatial abstraction:** Punjab's 36 districts are represented as a uniform 6×6 Manhattan grid. This ignores highway networks, river barriers, and shelter capacity.

## Future Work

- Scale the quantum SVM beyond 80 samples, or use hardware-efficient kernels that avoid barren plateaus.
- Use XY-mixers to keep QAOA inside the Hamming-weight-4 subspace, removing the need for penalty terms.
- Incorporate real road travel times, river barriers, and shelter capacity through a capacitated QUBO model.
- Test on real quantum hardware via IBM Quantum.

## Team

**Team QKhez**
- Aasia Bibi
- Zoya Arooj

