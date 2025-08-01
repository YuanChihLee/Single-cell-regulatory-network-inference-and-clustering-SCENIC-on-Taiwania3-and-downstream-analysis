# Single-cell-regulatory-network-inference-and-clustering-SCENIC-on-Taiwania3-and-downstream-analysis
This pipeline is used for processing data on Taiwania3, with subsequent analysis and visualization performed in an R environment.

# pySCENIC Multi-Run Script for Comprehensive Regulon Discovery

This repository contains a Bash script designed to run the `pySCENIC` pipeline multiple times on a High-Performance Computing (HPC) cluster managed by SLURM.

## 🎯 Goal

The network inference step (`GRNBoost2`) within the `pySCENIC` pipeline has a stochastic element. This means a single run may not capture all potential regulatory relationships.

The primary goal of this script is to **leverage this stochasticity to build a more comprehensive and robust set of regulons**. By executing `pySCENIC` multiple times with different random seeds, we can systematically explore various network inference outcomes and ultimately integrate them into a more complete and high-confidence gene regulatory network.

## ⚙️ How It Works

This Bash script automates the following workflow:

1.  **Resource Allocation (SLURM Configuration)**: The `#SBATCH` directives at the top of the script request computational resources (nodes, CPU cores, etc.) from the SLURM scheduling system.
2.  **Environment Setup**: The script automatically loads necessary system `modules` and activates the `conda` environment that contains `pySCENIC`.
3.  **Parameter & File Check**:
    * The user must first configure the paths to input files (e.g., the `.loom` file, TF list, etc.) within the script.
    * Before execution, the script validates that all required input files exist. If any are missing, it will abort and report an error.
4.  **Iterative Execution**:
    * The script executes a `for` loop 10 times (this number can be customized).
    * In each iteration, it performs the following actions:
        * Creates a unique output directory (e.g., `run_1`, `run_2`, ...).
        * Passes a **unique random seed** (`--seed "$i"`) to the `pyscenic grn` command. This is the most critical step in the entire process.
        * Continues with the `pyscenic ctx` and `pyscenic aucell` steps using the output from the `grn` command of that specific iteration.
5.  **Completion & Reporting**: After all loops have finished, the script calculates and displays the total execution time.

## 📋 Prerequisites

* An HPC environment using the SLURM scheduler.
* `conda` installed, with a configured environment containing `pySCENIC` and its dependencies.
* All required input files for `pySCENIC`:
    * An expression matrix (`.loom`)
    * A list of transcription factors (`.txt`)
    * Motif databases (`.feather` and `.tbl`)

## 🚀 Usage

1.  **Configure Parameters**:
    Open the Bash script file (`.sh`) and navigate to the `I N P U T -- F I L E S -- & -- P A R A M E T E R S` section. Modify the following variables to match your file paths:
    * `LOOM_FILE`
    * `TF_LIST`
    * `MOTIF_DB`
    * `ANNOTATION_FILE`
    * `MAIN_OUTPUT_DIR`

2.  **Submit the Job**:
    In your terminal, submit the script using the `sbatch` command:
    ```bash
    sbatch your_script_name.sh
    ```

## 📂 Output Structure

After the script completes successfully, you will find the following directory structure inside your `MAIN_OUTPUT_DIR`. The results from each run are neatly organized in separate directories.

```
SCENIC_10_runs_for_aggregation/
├── run_1/
│   ├── adj.tsv
│   ├── reg.csv
│   └── scenic_output.loom
├── run_2/
│   ├── adj.tsv
│   ├── reg.csv
│   └── scenic_output.loom
├── ...
└── run_10/
    ├── adj.tsv
    ├── reg.csv
    └── scenic_output.loom
```

## ⭐ Next Step: Aggregate Regulons

The purpose of this script is to **generate data**. To achieve the final goal, the critical next step is to **aggregate the results** from the 10 `reg.csv` files to identify the most stable regulons.

This requires a separate analysis script (e.g., in Python) that performs the following logic:
1.  Read all `run_X/reg.csv` files.
2.  Count the frequency of each unique TF-Target regulatory pair.
3.  Set a threshold (e.g., found in at least 3 out of 10 runs) to filter for stable regulons.
4.  Export this stable set into a final, high-confidence regulon list, which can then be used for downstream AUCell analysis or biological interpretation.
