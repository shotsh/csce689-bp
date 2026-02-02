# Proposal: Implement CBP 2025 TAGE-SC and MPP in ChampSim, and sweep capacity under fixed hardware budgets

## 1. Title

* CSCE 689 Special Topics in Branch Prediction Project Proposal

## 2. Abstract

* In this project, we implement two state-of-the-art branch predictors from the CBP 2025 workshop, TAGE-SC and the Multiperspective Perceptron Predictor (MPP), in the latest ChampSim.
* We evaluate prediction accuracy and performance across seven fixed storage budgets (24 KB to 1536 KB, doubling each step) using more than 100 benchmark traces.
* To enable apples-to-apples comparison, we use a unified storage accounting methodology that includes metadata, and we define a high-level budget scaling policy for both predictors (TAGE vs SC allocation for TAGE-SC, and fixed feature set with scaled tables for MPP).
* For reproducibility, we run a fixed warmup and a fixed measurement window per trace using ChampSim’s `--warmup-instructions` and `--simulation-instructions`, and report statistics from the simulation phase only.
* As a reference point, we also report results for ChampSim’s built-in `hashed_perceptron`. As stretch goals, we consider adding a loop predictor to TAGE-SC and exploring an MPP combiner after both predictors become stable.

## 3. Introduction

* High-accuracy branch prediction is critical for modern processors because each misprediction wastes frontend bandwidth and execution resources, reducing IPC.
* In this course project, we study two CBP 2025 workshop predictors specified by the ChampSim project option: TAGE-SC and the Multiperspective Perceptron Predictor (MPP).
* These predictors represent two complementary design approaches, namely tagged geometric-history tables and perceptron-based learning over history-derived features.
* We implement both predictors in ChampSim and evaluate accuracy and performance across seven fixed storage budgets (24 KB to 1536 KB) using more than 100 benchmark traces under a unified storage accounting methodology (including metadata).

## 4. Targets and Scope

* Primary targets
  * CBP 2025 TAGE-SC (loop predictor is optional and treated as a stretch goal)
  * CBP 2025 MPP (a combiner with TAGE is optional and treated as a stretch goal)

* Since the CBP 2025 descriptions focus on differences, we refer to prior work* to concretize the implementation specification.
* For MPP as well, we use prior work* as the base and concretize the 2025 implementation specification.

* Stretch goals
  * Optionally add a loop predictor to TAGE and quantify the incremental benefit.
  * After both predictors become stable, optionally implement a combiner that combines TAGE with MPP.

* Budget scaling policy (high-level)
  * For TAGE-SC, we keep the number of components and the history-length scheme consistent with the CBP reference design and scale table entry counts under a fixed allocation ratio between TAGE and SC.
  * For MPP, we keep the feature set fixed and scale the table sizes (and, if needed, weight storage) to match each budget point.

## 5. Implementation Plan
### 5.0 Implementation specification before coding
* Before coding, we will read the key papers and study ChampSim’s branch predictor API, then write an implementation specification document (or implementation memo) to reduce integration risk.
* The spec document will explicitly define
  * predictor state and update rules (including what metadata is carried from predict_branch() to last_branch_result())
  * the storage accounting methodology (including metadata) and the budget allocation rule at each budget point
  * a minimal sanity-check plan (small-trace regression runs, etc.)

### 5.1 Stepwise implementation plan for TAGE-SC

* Step T0: Implement the TAGE core by referring to TAGE 2006 (tagged tables, provider and altpred, usefulness bits, allocation on mispredict, etc.).
* Step T1: Add the statistical corrector (SC) in a first working form by referring to TAGE 2011 (sum of multiple SC tables, inversion based on a threshold, use TAGE confidence as an input).
* Step T2: Move the SC structure and parameters closer to those described in CBP 2014 and 2016.
  * History length selection, number of tables, weight width, threshold update rule
  * Fix the budget allocation rule so the sweep is straightforward.
* Step T3: Add the CBP 2025 diffs (layer the diff mechanisms on top of the 2016-based design).
* Stretch: Add a loop predictor.

### 5.2 Stepwise implementation plan for MPP

* Step M0: Sanity-check baseline measurements (`hashed_perceptron`, and also `gshare` if needed).
* Step M1: Build an MPP-oriented framework based on a hashed perceptron (feature generation, sum computation, and a mechanism to retain the set of indices needed for updates).
* Step M2: First implement the prior-work MPP 2016 specification, then verify that training works and that the accuracy trend is reasonable.
* Step M3: Add the CBP 2025 diffs and any required filtering.
* Stretch: After TAGE-SC is stable, implement the CBP 2025 combiner.

### 5.3 Integration into ChampSim
* Maintain all predictor state required across calls (tables, histories, and any auxiliary structures).
* Implement the prediction and training flow through ChampSim’s hooks: make predictions in predict_branch() and train/update in last_branch_result(), consistent with ChampSim’s timing model.
* Correctly pass the metadata needed for training (e.g., table indices, provider/alt selection, confidence, and feature indices) from predict_branch() to last_branch_result().


## 6. Experimental Methodology

* Workloads
  * Use more than 100 traces.
  * Evaluate on the following SPEC benchmarks distributed with DPC 3.
    * SPEC CPU2006
      * 29 benchmark types
      * 94 traces
    * SPEC CPU2017
      * 20 benchmark types
      * 95 traces
  * Stretch goals
    * GAP suite (bc, bfs, cc, pr, sssp, tc × each graph)
      * 30 combinations
      * 123 traces
    * XSBench suite
      * 43 traces

* Execution window (reproducibility)
  * For each trace, we run a fixed warmup and a fixed measurement window using ChampSim’s `--warmup-instructions` and `--simulation-instructions`, and we report statistics from the simulation phase only.

* Budgets
  * For seven budget points (24, 48, 96, 192, 384, 768, 1536 KB), fix the allocation rule at each budget point and scale predictor storage according to the budget scaling policy above.

* Metrics
  * IPC
  * Branch MPKI

* Reference baseline 
  * As a reference point, we also report results for ChampSim’s built-in `hashed_perceptron`.

* Plots
  * Two line plots
    * IPC vs budget
    * MPKI vs budget

## 7. Team Plan

* Roles
  * Primary owner for TAGE-SC
  * Primary owner for MPP

* Execution
  * Mutually review key logic and critical paths.
  * Set weekly milestones and check progress on a weekly basis.

* Mutually review the implementation specification document and freeze it before major coding, to avoid interface and accounting inconsistencies.

## 8. Preliminary Work

* Repository setup
  * Obtain ChampSim and confirm it builds successfully.
* Built a parallel execution environment on a cluster.
* Baseline verification
  * Confirm that `hashed_perceptron` produces MPKI and IPC.
  * Ran SPEC 2006 and SPEC 2017 benchmarks.

## 9. References (main 8 plus datasets and tools)

### 9.1 TAGE family

1. André Seznec and P. Michaud, “A case for (partially) TAgged GEometric history length branch prediction,” JILP, 2006.
2. A. Seznec, “A New Case for the TAGE Branch Predictor,” MICRO, 2011. (refer to local PDF)
3. A. Seznec, “TAGE-SC-L Branch Predictors,” CBP Workshop, 2014.
4. A. Seznec, “TAGE-SC-L Branch Predictors Again,” CBP Workshop, 2016. ([SIGARCH][1])
5. A. Seznec, “TAGE-SC for CBP2025,” CBP Workshop, 2025. ([CBP Workshop paper][2])

### 9.2 MPP family

6. Daniel A. Jiménez, “Multiperspective Perceptron Predictor,” CBP Workshop, 2016.
7. D. A. Jiménez, “Multiperspective Perceptron Predictor for CBP2025,” CBP Workshop, 2025. ([dblp][3])

### 9.3 Simulator

8. N. Gober et al., “The Championship Simulator: Architectural Simulation for Education and Competition,” arXiv:2210.14324, 2022. ([dblp][4])
