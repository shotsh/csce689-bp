# Proposal: Implement CBP 2025 TAGE-SC and MPP in ChampSim, and sweep capacity under fixed hardware budgets

## 1. Title

* CSCE 689 Special Topics in Branch Prediction Project Proposal

## 2. Abstract

* In this project, we implement two state-of-the-art branch predictors from the CBP 2025 workshop, TAGE-SC and the Multiperspective Perceptron Predictor (MPP), in the latest ChampSim.
* We evaluate prediction accuracy and performance across seven fixed storage budgets (24 KB to 1536 KB, doubling each step) using more than 100 benchmark traces.
* To enable apples-to-apples comparison, we use a unified storage accounting methodology that includes metadata, and we define a high-level budget scaling policy for both predictors (TAGE vs SC allocation for TAGE-SC, and fixed feature set with scaled tables for MPP).
* For reproducibility, we run a fixed warmup and a fixed measurement window per trace using ChampSim’s `--warmup-instructions` and `--simulation-instructions`, and report statistics from the simulation phase only.
* As a reference point, we also report results for ChampSim’s built-in `hashed_perceptron`. If time permits, we will perform focused ablations on the CBP 2025 designs to explain which components drive accuracy.

## 3. Introduction

* High-accuracy branch prediction is critical for modern processors because each misprediction wastes frontend bandwidth and execution resources, reducing IPC.
* In this course project, we study two CBP 2025 workshop predictors, TAGE-SC and the Multiperspective Perceptron Predictor (MPP), by porting/adapting the CBP 2025 championship reference predictors to ChampSim.
* These predictors represent two complementary design approaches, namely tagged geometric-history tables and perceptron-based learning over history-derived features.
* We implement both predictors in ChampSim and evaluate accuracy and performance across seven fixed storage budgets (24 KB to 1536 KB) using more than 100 benchmark traces under a unified storage accounting methodology (including metadata).

## 4. Targets and Scope

* Primary targets
  * CBP 2025 TAGE-SC (as in the championship reference code)
  * CBP 2025 MPP integrated with TAGE-SC-L (as in the championship reference code)

* We will start from the CBP 2025 championship reference code and adapt it to run in ChampSim with minimal algorithmic changes.
* Prior work will be used to understand design intent and to document configuration and storage accounting.

* If time permits (ablations)
  * Evaluate simplified configurations by disabling selected components or features one at a time to understand their contribution to accuracy and MPKI.
  * (Optional) Run MPP without its intended integration only as an ablation to quantify the impact of the coupling.

* Budget scaling policy (high-level)
  * For TAGE-SC, we keep the number of components and the history-length scheme consistent with the CBP reference design and scale table entry counts under a fixed allocation ratio between TAGE and SC.
  * For MPP, we keep the feature set fixed and scale the table sizes (and, if needed, weight storage) to match each budget point.

## 5. Implementation Plan

We will apply the following steps to both TAGE-SC and MPP.

### Step 1: Extract the CBP-side code as a standalone predictor library

* (a) Identify the CBP headers and utilities that the predictor core depends on.
* (b) Make the predictor buildable and compilable in isolation, independent of ChampSim.

### Step 2: Build a ChampSim adapter

* Bring over the CBP 2025 predictor classes/functions largely as-is.
* Repackage the PC / branch type / target information coming from ChampSim into the format expected by the CBP predictor.
* Save the required metadata (e.g., sets of indices) and pass it to the update path.
* Invoke the predictor from ChampSim’s `predict_branch()` and `last_branch_result()`.
* Call `update()` according to ChampSim’s update timing (i.e., when the actual outcome arrives in `last_branch_result()`, update using the saved metadata).

### Step 3: Run with non-speculative updates to match ChampSim

* We follow ChampSim’s update model. If the reference code assumes history progression at prediction time, we will keep the required history bookkeeping internal to the predictor without modifying ChampSim.
* Do not implement squash or rollback in ChampSim.

### Concrete tasks

* Locate the predictor entry points in the CBP 2025 code and identify all dependencies.
* Create a minimal project (or a static library) that can compile the predictor code by itself.
* Based on an existing predictor implementation such as `gshare`, build a wrapper/adapter layer in ChampSim that calls into the CBP predictor.
  * Make it callable from ChampSim’s `predict_branch()` / `last_branch_result()`.
  * Save and carry the metadata needed for updates (e.g., referenced index sets) from predict to update.
* Call the CBP predictor and run at least one trace end-to-end.
* Get to a point where Branch MPKI does not “break” (i.e., does not degrade dramatically).
  * Confirm that it does not become significantly worse than `gshare` / `hashed_perceptron`.

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
  * If time permits (stretch workloads)
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
  * Mutually review and freeze the adapter interface and storage accounting rules before scaling to the full budget sweep.

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
