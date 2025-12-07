> **Status**: CVRP framework complete with validated results. 
> VRPTW extension in active development (expected completion Mar 2025).

## Project Overview
This project proposes **Graph-RL-HH**, an integrated framework that learns adaptive operator-selection policies for solving Vehicle Routing Problems (VRP) and Vehicle Routing Problems with Time Windows (VRPTW). By combining:

- **GraphSAGE** for graph-structured state encoding
- **Proximal Policy Optimization (PPO)** for policy learning
- **Adaptive Large Neighbourhood Search (ALNS)** for local optimization

The framework achieves competitive solution quality on CVRP with strong robustness, and is being extended to VRPTW.

---

## Key Contributions

1. **Unified Framework** combining graph-structured state encoding, learning-based operator selection, and risk-sensitive reward shaping for VRP

2. **Novel Graph Encoding** for routing: KNN adjacency with distance + solution-edge duality enables spatial and topological understanding

3. **Risk-Sensitive Training** using IQR normalization and variance penalty provides stable convergence

4. **Demonstrated Robustness** across random seeds (CV < 1%)

---

## Framework Architecture

```
Graph-RL-HH Framework
├── Data Processing Layer
│   ├── Solomon instance parsing
│   └── KNN graph construction (k=10)
│
├── Graph Encoding Layer (GraphSAGE)
│   ├── Node feature embedding (6 dimensions)
│   ├── 2-layer GNN with mean aggregation
│   └── Graph-level pooling
│
├── Environment Layer (ALNS)
│   ├── State representation (graph embedding + search features)
│   ├── Action space (6 operators: 3 destroy, 3 local-search)
│   ├── Phase transition (destroy ↔ local-search)
│   └── Reward shaping (IQR normalization + variance penalty)
│
└── Learning Layer (PPO)
    ├── Policy network with masked action selection
    ├── Value function for advantage estimation
    ├── Differentiable heuristic hints (KL divergence)
    └── Risk-sensitive loss combining 4 components
```

--- 

## Features

- **Graph-Based State Representation**
    - Captures spatial dependencies and route structure
    - KNN adjacency matrix with distance and solution-edge information
    - Enables better cross-instance generalization
- **Learning-Based Operator Selection**
    - Learns to select destroy/repair and local-search operators dynamically
    - Incorporates heuristic hints via differentiable KL-divergence loss
    - Masked-action PPO ensures infeasible operators are never selected
- **Risk-Sensitive Reward Shaping**
    - IQR-normalized reward prevents overconfidence on sparse signals
    - Variance penalty encourages stable and reproducible policies
    - Stabilized training even in noisy optimization environments
- **Reproducible Research**
    - Modular architecture with clear separation of concerns
    - Multi-seed evaluation with low variance
    - Comprehensive baseline comparisons

---

## Core Workflow (Pseudo code)

```python
for each training episode:
    
    # Step 1. Construct graph state
    build node features (x, y, demand, route_id, pos, residual_cap)
    build edge features (distance, solution_edge_flag)
    encode gnn_state = GNN(graph)

    # Step 2. Augment with search dynamics
    state = concat(gnn_state, iteration_index, acceptance_rate)

    # Step 3. PPO policy selects operator
    mask infeasible operators
    action = πθ(state)

    # Step 4. Apply destroy/repair or local search
    new_solution = operator(action, current_solution)
    compute Δcost

    # Step 5. Compute risk-sensitive reward
    reward = Δcost/IQR - λ * rolling_variance(Δcost)

    # Step 6. PPO update with heuristic hints
    compute q(o | s) from operator improvements
    L_total = L_PPO + λhint * KL(q || π)
    
    update policy parameters
```

---

## Module Justification

### `data_processing.py`
- Loads Solomon (1987) VRPTW benchmark data
- Converts raw TXT format into unified `.npz` graph format
- Computes distance matrices and kNN neighbourhoods
- Validates CVRP/VRPTW feasibility components

### `vrp_environment.py`
- Maintains routing state: routes, loads, feasibility
- Implements cost evaluation & temporal constraints
- Provides step-wise interface for RL training

### `encoder.py`
- Implements 2–3 layer GNN with mean pooling
- Encodes structural route relationships (following Johnn et al., 2024)
- Supports both CVRP and VRPTW features

### `operators/`
- Destroy operators:
Remove customers from the current solution to create diversification.
    - `Random Removal` – removes customers uniformly at random
    - `Worst Removal` – removes customers contributing highest marginal cost
- Repair operators:
Reinsert removed customers using greedy heuristics.
    - `Regret Insertion` – chooses insertion with maximum regret difference
- Local search operators:
Perform neighbourhood improvements to refine existing routes.
    - `Swap` – exchanges two customers
    - `Relocate` – moves a customer to a different position
    - `2-opt` – reverses a segment of a route to reduce distance

### `ppo_agent.py`
- Clipped PPO objective
- Generalized Advantage Estimation (GAE)
- KL-based heuristic hint loss
- Action masking for infeasible operators
- Stable reward & policy update mechanisms

### `experiments/`
- Includes baseline comparisons against:
    - NN / Insertion / CW / OR-Tools
    - Classical PPO-based RL-HH
    - ALNS
- Contains scripts for ablations and parameter sweeps

---

## Example Output

### CVRP Results: Solomon R101 Instance

| Method | Best Distance (km) | Mean Distance (km) | Computation Time (s) | Gap to OR-Tools (%) |
|--------|--------------------|--------------------|----------------------|-------------------|
| **Graph-RL-HH** | **846.77** ✨ | **853.99** | ~150 episodes | **-0.8%** |
| OR-Tools (5s) | 860.49 | 860.49 | 5.1 | 0.0 |
| Nearest Neighbor | 1174.36 | - | 0.01 | 36.5 |
| 2-opt | 1140.23 | - | 0.014 | 32.5 |
| Clarke-Wright | 2404.96 | - | 0.023 | 179.5 |

**Key Results on CVRP:**
- **Stability**: Standard deviation 5.07 km (CV = 0.59%), demonstrating robust convergence
- **Quality**: Mean performance within 0.8% of OR-Tools baseline
- **Generalization**: Consistent performance across 10 independent seeds
- **Learning**: PPO losses converge smoothly; no divergence observed

### Training Stability (10 Seeds on R101 CVRP)

```
Mean:                 853.99 km
Std Dev:               5.07 km
Coefficient of Var:    0.59%  ← Excellent robustness
Min:                 846.77 km (best performance)
Max:                 862.45 km
Median:              853.23 km
```

**Conclusion**: Framework demonstrates robust convergence regardless of random initialization.

---

## Research Status

### CVRP Framework: Completed
- Full framework design and implementation
- Comprehensive baseline evaluation and comparison
- Multi-seed robustness validation
- Early cross-validation on R101 demonstrates strong potential for generalization across instance distributions

**Current Findings (CVRP):**
- Robust learning with minimal inter-seed variance (CV < 1%)
- Variance-penalized reward shaping effectively stabilizes training
- Graph-based state representation enables consistent operator selection

### VRPTW Extension: In Progress
- Framework adaptation for time-window constraints (temporal features, feasibility checks)
- Phase 2 implementation: Adding time-window handling to operators and reward function
- **Status**: Code implementation in progress; no quantitative results yet
- **Timeline**: Expected baseline comparison completion by Jan 2025

**Important Note**: VRPTW results will be released once the extension is completed and thoroughly validated. Current documentation focuses on validated CVRP achievements.

---

## Methodology & Innovation

### Graph-Based State Encoding
Unlike traditional RL-HH methods that use hand-crafted numerical features, Graph-RL-HH leverages spatial and topological structure through GraphSAGE encoding. This enables better understanding of:
- Customer spatial clustering
- Current route configurations
- Neighborhood relationships

### Risk-Sensitive Learning
The variance-penalized reward combines:
- **IQR normalization**: Prevents outlier improvements from dominating
- **Variance penalty**: Encourages reproducible policy behavior
- **Result**: Stable training even with sparse reward signals

### Adaptive Operator Selection
Instead of static operator weights (traditional ALNS), the PPO policy learns to select operators based on:
- Current state (graph embedding)
- Search progress (iteration index)
- Recent performance (acceptance rate)

---

## Requirements

- Python >= 3.9
- PyTorch
- PyTorch Geometric (PyG)
- torch>=1.9.0      
- numpy>=1.19.0
- scikit-learn>=0.24.0
- ortools>=9.0.0
- matplotlib>=3.3.0
