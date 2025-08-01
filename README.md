# Single-cell-regulatory-network-inference-and-clustering-SCENIC-on-Taiwania3-and-downstream-analysis
This pipeline is used for processing data on Taiwania3, with subsequent analysis and visualization performed in an R environment.

#!/bin/bash
#
# This script runs the pySCENIC pipeline 10 times with different random seeds.
# The primary goal is to capture a more comprehensive set of regulons by leveraging
# the stochastic nature of the GRN inference step.
# The results from all runs can be aggregated later to build a robust regulon set.
#

#==============================================================================
# S L U R M -- C O N F I G U R A T I O N
#==============================================================================
#SBATCH -A MST112171          # Account name/project number
#SBATCH -J SCENIC_aggregate   # Job name reflecting the aggregation goal
#SBATCH -p ct56               # Partition name
#SBATCH -N 1                  # Request a single node
#SBATCH --ntasks-per-node=28  # Number of tasks (processes) to run on the node
#SBATCH -c 1                  # Number of CPU cores per task
#SBATCH -o %x_%j.out          # Standard output file (%x: job name, %j: job ID)
#SBATCH -e %x_%j.err          # Standard error output file

#==============================================================================
# E N V I R O N M E N T -- S E T U P
#==============================================================================
echo "========================================="
echo "Setting up the environment..."

# Load necessary modules from the cluster environment
module purge
module load twcc
module load old-module
module load libs/singularity/3.10.2

# Initialize and activate the Conda environment as requested
conda init bash
source ~/.bashrc # Or the appropriate shell config file
conda activate scenic_env

echo "Environment setup complete."
echo "========================================="


#==============================================================================
# I N P U T -- F I L E S -- & -- P A R A M E T E R S
#==============================================================================
# --- Main Inputs (shared across all runs) ---
LOOM_FILE="merged_filtered.loom"
TF_LIST="allTFs_mm.txt"
MOTIF_DB="mm10__refseq-r80__10kb_up_and_down_tss.mc9nr.genes_vs_motifs.rankings.feather"
ANNOTATION_FILE="motifs-v10nr_clust-nr.mgi-m0.001-o0.0.tbl"

# --- Resources ---
NUM_WORKERS=28

# --- Output Directory ---
MAIN_OUTPUT_DIR="SCENIC_10_runs_for_aggregation" # Parent directory for all run results

#==============================================================================
# P R E - R U N -- C H E C K
#==============================================================================
echo "Checking for the existence of input files..."
for f in "$LOOM_FILE" "$TF_LIST" "$MOTIF_DB" "$ANNOTATION_FILE"; do
  if [ ! -f "$f" ]; then
    echo "[ERROR] Input file not found: $f. Aborting."
    exit 1
  fi
done
echo "All input files found."
echo "========================================="


#==============================================================================
# E X E C U T E -- P I P E L I N E -- 10 -- T I M E S
#==============================================================================
cd /work/j120885731/SCENIC || exit 1

start_time=$(date +%s)
echo "[INFO] Starting 10 pySCENIC runs to capture a comprehensive regulon set..."
mkdir -p "$MAIN_OUTPUT_DIR"

for i in {1..10}; do
  RUN_DIR="${MAIN_OUTPUT_DIR}/run_${i}"
  mkdir -p "$RUN_DIR"
  echo "-------------------------------------------------"
  echo "🚀 Starting Run ${i}/10 for regulon discovery"
  echo "   Output Directory: ${RUN_DIR}"
  echo "-------------------------------------------------"

  # Step 1: GRN with a unique seed to explore different network possibilities
  echo "[INFO] Run ${i}, Step 1: Running GRNBoost2 with seed ${i}..."
  pyscenic grn "$LOOM_FILE" "$TF_LIST" -o "${RUN_DIR}/adj.tsv" --num_workers "$NUM_WORKERS" --seed "$i"

  # Step 2: Regulon prediction
  echo "[INFO] Run ${i}, Step 2: Running cisTarget (ctx)..."
  pyscenic ctx "${RUN_DIR}/adj.tsv" "$MOTIF_DB" --annotations_fname "$ANNOTATION_FILE" --expression_mtx_fname "$LOOM_FILE" --output "${RUN_DIR}/reg.csv" --mode "dask_multiprocessing" --mask_dropouts --num_workers "$NUM_WORKERS"

  # Step 3: AUCell scoring
  echo "[INFO] Run ${i}, Step 3: Running AUCell..."
  pyscenic aucell "$LOOM_FILE" "${RUN_DIR}/reg.csv" --output "${RUN_DIR}/scenic_output.loom" --num_workers "$NUM_WORKERS"

  echo "[SUCCESS] Finished Run ${i}/10."
done

#==============================================================================
# J O B -- C O M P L E T I O N
#==============================================================================
end_time=$(date +%s)
elapsed=$(( (end_time - start_time) / 60 ))
echo "================================================="
echo "🎉 All 10 SCENIC runs have finished successfully."
echo "⏰ Total time elapsed: ${elapsed} minutes."
echo "📂 Results are in '${MAIN_OUTPUT_DIR}'. Next step is to aggregate the regulons."
echo "================================================="
