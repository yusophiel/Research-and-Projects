## Project Overview
This project focuses on the design and implementation of a **Graph-Enhanced Reinforcement Learning Hyper-heuristic (Graph-RL-HH)** for solving **Vehicle Routing Problems (VRP)** and **VRP with Time Windows (VRPTW)**.

This project implements a **full end-to-end learning-to-optimise framework for VRP/VRPTW**, combining **graph representation learning** with **reinforcement-driven operator control**. The system reproduces classical baselines while integrating novel **GNN-based state features** and **risk-sensitive PPO training**.

The model integrates a **Graph Neural Network (GNN) encoder** with a **PPO-based hyper-heuristic**, enabling the agent to learn when and which destroy/repair or local-search operator to apply.
The state representation leverages the inherent graph structure of VRP instances, addressing the limitations of traditional RL-HHs that rely on handcrafted scalar features only.

This framework supports:
- Graph-based state encoding
- Masked-action PPO with feasibility checks
- Classical VRP heuristic operators (destroy/repair + local search)
- Evaluation across CVRP and VRPTW benchmarks
- Ablation studies & generalisation testing

---

## Features

- **Graph Neural Network Encoder**
    - Node features: coordinates, demand, route ID, position index, residual capacity
    - Edge features: distance & solution-edge indicator
- **PPO-based Reinforcement Learning Hyper-heuristic** 
    - Supports 7+ operators (`Random Removal, Worst Removal, Regret Insertion, Swap, Relocate, 2-op`)
    - Action masking to avoid infeasible choices
- **Risk-Sensitive Reward Function**
    - IQR-normalised improvement
    - Rolling variance penalty for training stability
- **Heuristic-Hint Mechanism** 
    - KL-based soft imitation of operator improvements
- **Baseline Framework** 
    - `Nearest Neighbor` – simple constructive heuristic that always visits the closest unserved customer
    - `Cheapest Insertion` – inserts nodes greedily based on minimal increased cost
    - `Clarke–Wright Savings` – classical heuristic merging routes based on savings values
    - `OR-Tools VRP Solver` – Google's high-performance solver with guided local search
    - `Classical PPO-RL-HH` – RL hyper-heuristic using handcrafted state features (no graph encoder)
    - `ALNS baseline` – adaptive large neighbourhood search, a strong metaheuristic reference
- **Evaluation Tools** 
    - Training curve visualisation
    - Distance progression plots
    - Operator-usage heatmaps
    - Instance-level benchmarking

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

Preliminary experiments on Solomon C1 and R1 instances show that Graph-RL-HH achieves lower final tour cost and more stable convergence compared to classical RL-HH baselines.

- Training curves (Distance vs. Episode)
- Cumulative reward curves
- Ablation study:
    - Remove edge features
    - Remove route-position features
    - Disable heuristic hints
- Comparison:
    - Graph-RL-HH vs RL-HH vs ALNS vs OR-Tools

---

## Future Work

1. **Full VRPTW Integration** 
    - Add time window feasibility masks
    - Extend node features with `[ready_time, due_time, service_time]`
2. **Parallel Rollout Engine** 
    - Speed up training via multi-instance parallel collection
3. **Improved GNN Architectures** 
    - Explore GINEConv / attention-based variants for deeper relational encoding
4. **Generalisation Benchmarks** 
    - Test transfer between `(C, R, RC)` distributions
    - Training on small instances → inference on large instances
4. **Deployment & Packaging** 
    - Convert into pip-installable library
    - Add detailed documentation & plotting dashboards

---

## Requirements

- Python >= 3.9
- PyTorch
- PyTorch Geometric (PyG)
- NumPy
- OR-Tools
- Matplotlib
