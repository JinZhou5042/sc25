# ECF Calculator for High Energy Physics Analysis

This project provides a framework for calculating Energy Correlation Functions (ECFs) and other jet substructure variables using distributed computing with Dask and TaskVine. It is designed for analyzing large datasets common in High Energy Physics (HEP).

## Installation

Follow these steps to set up the necessary environment and dependencies.

### 1. Create Conda Environment

First, set up a dedicated conda environment with the required Python version and build tools:

```bash
unset PYTHONPATH
# Create conda environment named 'dv5' with Python 3.10
conda create -y -n dv5 -c conda-forge --strict-channel-priority python=3.10
conda activate dv5
# Install build requirements for cctools
conda install -y gcc_linux-64 gxx_linux-64 gdb m4 perl swig make zlib libopenssl-static openssl conda-pack packaging cloudpickle flake8 clang-format threadpoolctl
```

### 2. Build and Install cctools (with TaskVine)

Next, build and install the specific version of cctools used by this project:

```bash
# Clone the repository
git clone https://github.com/JinZhou5042/cctools.git
cd cctools
# Fetch the specific branch required for this project
git fetch origin sc25
git switch sc25
# Configure cctools to install within the active conda environment
./configure --with-base-dir $CONDA_PREFIX --prefix $CONDA_PREFIX
# Build and install
make -j8 && make install
# Verify installation
vine_worker --version # Should display the installed cctools version
cd .. # Return to the main project directory
```

### 3. Install Python Packages

Install the required Python packages using pip:

```bash
pip install dask==2024.7.1 dask-awkward==2024.7.0 awkward==2.6.6 coffea==2024.4.0 fastjet==3.4.2.1
```
*Note: These specific versions are recommended for compatibility.*

## Setup

### 1. Clone This Project

Clone this repository to your shared filesystem:
```bash
git clone https://github.com/JinZhou5042/sc25.git
cd sc25
```

### 2. Package Conda Environment for TaskVine Workers

Create a poncho package containing the conda environment. This allows TaskVine workers to replicate the environment remotely. Ensure you are in a directory accessible by both the manager and workers (e.g., a shared filesystem).

```bash
# Navigate to a shared filesystem location if not already there
# This command packages the 'dv5' conda environment created earlier
poncho_package_create $CONDA_PREFIX sc25_env.tar.gz
```
*This process might take some time.*

### 3. Configure Resources (factory.json)

The `factory.json` file defines the resource requirements (cores, memory, disk) for TaskVine workers.

-   **Review and adjust `factory.json`** to match the resources available in your computing environment.
-   **Important:** If you modify the `cores` value in `factory.json`, you must also update the `lib_resources` dictionary in the `dask.compute` call within `ecf_calculator.py` to match. Specifically, `lib_resources={'cores': N}` where `N` is the number of cores per worker.

### 4. Start TaskVine Workers

Open a new terminal session on a machine with access to the shared filesystem where you placed the poncho package. Run the following script to start the TaskVine factory, which manages worker creation:

```bash
bash run_factory.sh
```
You should see messages indicating workers are connecting to the manager. Keep this factory process running during your analysis.

## Dataset Preparation

### 1. Directory Structure

Input data (ROOT files) must be organized into subdirectories under a main data directory. By default, the script expects this structure within a `samples/` directory relative to the script's location:

```
samples/
├── dataset1/      # e.g., hgg
│   ├── file1.root
│   └── file2.root
├── dataset2/      # e.g., hgg_1
│   └── file3.root
└── ...
```
-   **Default Path:** The script currently defaults to `/project01/ndcms/jzhou24/samples`. You might need to modify the `samples_path` variable inside `ecf_calculator.py` if your data resides elsewhere.

### 2. Download Input Data (Example)

If you need to download datasets, ensure you are in the main project directory (`sc25`). The data should be downloaded into the `samples` subdirectory.

```bash
# Assuming you are in the 'sc25' directory
cd samples
# Example: Download data using curl, wget, or copy commands
# Replace 'https://example.com/data.tar.gz' with the actual source
# curl -O https://example.com/data.tar.gz && tar xzf data.tar.gz
# wget https://example.com/data.tar.gz && tar xzf data.tar.gz
# cp /path/to/your/data/*.root .
cd .. # Return to the 'sc25' project directory
```

### 3. Dataset Management

-   **Preprocessing Time:** If you have many datasets, the initial preprocessing step (`--preprocess`) can be time-consuming as it scans all files.
-   **Selective Preprocessing:** To speed this up, you can temporarily move unused dataset directories out of the main data directory before running `--preprocess`. Move them back afterward if needed.
-   **Adding New Datasets:**
    1.  Create a new subdirectory within your main data directory (e.g., `samples/new_dataset`).
    2.  Place the corresponding ROOT files inside it.
    3.  Re-run the `--preprocess` step for the script to recognize the new dataset.

## Usage

The main script `ecf_calculator.py` orchestrates the analysis using DaskVine.

```bash
python ecf_calculator.py [options]
```

### Core Workflow Steps

1.  **Preprocessing (Required First Run & After Dataset Changes):**
    Scans the data directory, analyzes file metadata, and generates `samples_ready.json`. This file is crucial for subsequent steps.
    ```bash
    # Run preprocessing only. Exits after creating/updating samples_ready.json.
    python ecf_calculator.py --preprocess
    ```

2.  **Show Available Samples (Optional):**
    Lists the datasets and file counts found in `samples_ready.json`. Useful for verifying detected datasets.
    ```bash
    python ecf_calculator.py --show-samples
    ```

3.  **Analysis Run (Task Generation & Execution):**
    Generates the Dask task graph based on `samples_ready.json` (or loads a saved graph) and executes it using the connected TaskVine workers.
    ```bash
    # Basic run: Process default 'hgg' dataset, ECF n<=3
    python ecf_calculator.py

    # Process a specific dataset (must be listed by --show-samples)
    python ecf_calculator.py --sub-dataset hgg_1

    # Process all datasets found in samples_ready.json
    python ecf_calculator.py --all

    # Process all datasets with a higher ECF calculation bound (e.g., n<=5)
    python ecf_calculator.py --ecf-upper-bound 5 --all
    ```

### Task Checkpointing (Recommended for Batch Runs / Re-runs)

Separate task graph *generation* from *execution*. Generate the graph once, save it, and then load it multiple times for execution, potentially with different DaskVine tuning parameters.

1.  **Generate and Save Task Graph:**
    Add `--checkpoint-to <filename.pkl>` to an analysis command. This saves the Dask graph to the specified file *without* executing it.
    ```bash
    # Generate tasks for all datasets (ECF n<=5) and save to a file
    python ecf_calculator.py --all --ecf-upper-bound 5 --checkpoint-to my_tasks_ecf5.pkl
    ```

2.  **Load and Execute Task Graph:**
    Use `--load-from <filename.pkl>` *instead* of dataset selection arguments (`--all`, `--sub-dataset`). Loads the pre-generated graph and executes it.
    ```bash
    # Load the graph and execute it with specific DaskVine options
    python ecf_calculator.py --load-from my_tasks_ecf5.pkl --temp-replica-count 2
    ```
    *Note: `--checkpoint-to` and `--load-from` are mutually exclusive.*

### DaskVine Configuration Arguments

These arguments fine-tune the interaction with TaskVine during task *execution*.

**Manager Connection & Run Logging:**

-   `--manager-name <name>`: Connect to a specific TaskVine manager. Defaults to `{user}-hgg7`. Ensure this matches the name used by your running manager.
-   `--template <template_name>`: Create a subdirectory named `<template_name>` within the TaskVine run information path (`run_info_path`) to store logs and reports for this specific run. Helps organize outputs. The script attempts to use a shared path (`/afs/crc.nd.edu/...`) first, falling back to `./vine-run-info`.
-   `--enforce-template`: If the specified `--template` directory already exists, delete it without confirmation. **Use with caution.**

**General Tuning:**

-   `--wait-for-workers <seconds>`: Time (seconds) for the manager to wait for workers before starting computation.
-   `--disable-worker-join-after-first-run`: Prevent new workers from joining once computation begins.
-   `--max-workers <count>`: Limit the maximum number of workers used (default: 30000).

**Fault Tolerance:**

-   `--temp-replica-count <count>`: Number of replicas for intermediate files (default: 1). Higher values increase resilience but use more storage.
-   `--checkpoint-threshold <seconds>`: Minimum time (seconds) an intermediate result is kept in memory before considering checkpointing to storage (default: 30). Affects recovery strategy.
-   `--enforce-worker-eviction-interval <seconds>`: **For testing only.** Forces worker eviction at regular intervals to simulate failures.

**Performance & Scheduling:**

-   `--load-balancing`: Enable TaskVine's dynamic load balancing.
-   `--load-balancing-interval <seconds>`: Frequency (seconds) for load balancing checks (default: 3). Requires `--load-balancing`.
-   `--load-balancing-factor <factor>`: Target ratio between busiest/least busy worker queues (default: 1.1). Lower means more even balancing. Requires `--load-balancing`.
-   `--prune-depth <depth>`: Dask graph pruning depth (default: 0). May optimize scheduling.
-   `--scheduling-mode <mode>`: Task scheduling strategy (default: `LIFO`). Other modes like `storage-footprint-aware` might be available.

### Examples

1.  **Minimal Workflow (Preprocess + Analyze default 'hgg'):**
    ```bash
    # 1. Preprocess data in samples/
    python ecf_calculator.py --preprocess

    # 2. Run analysis with defaults
    python ecf_calculator.py
    ```

2.  **Analyze Specific Dataset with Higher ECF Bound:**
    ```bash
    # Assumes preprocessing is done
    python ecf_calculator.py --sub-dataset hgg_2 --ecf-upper-bound 6
    ```

3.  **Workflow using Checkpointing:**
    ```bash
    # 1. Preprocess (if needed)
    python ecf_calculator.py --preprocess

    # 2. Generate and save task graph for all datasets, ECF n<=5
    python ecf_calculator.py --all --ecf-upper-bound 5 --checkpoint-to all_tasks_ecf5.pkl

    # 3. Load graph and run with specific settings
    python ecf_calculator.py --load-from all_tasks_ecf5.pkl --temp-replica-count 3 --template run_rep3 --manager-name my-manager
    ```

4.  **Batch Experiments:**
    The `run_batch_*.sh` scripts (e.g., `run_batch_replication.sh`) demonstrate how to run experiments systematically. They typically use `--load-from` with a pre-generated task file and loop through different DaskVine tuning parameters (like `--temp-replica-count`), using `--template` to separate the logs for each run configuration.

## Output

Analysis results (parquet files containing calculated variables like ECFs, color ring, etc.) are saved in the `output/` directory, organized by dataset:

```
output/
├── dataset1/      # e.g., output/hgg/
│   ├── part-0.parquet
│   └── ...
├── dataset2/      # e.g., output/hgg_1/
│   └── ...
└── ...
```

## Important Notes

-   **Preprocessing:** Always run `--preprocess` if `samples_ready.json` is missing or if the contents/structure of the data directory (`samples/` or custom path) changes. Preprocessing only creates the JSON file; a separate run is needed for analysis.
-   **TaskVine Setup:** A running TaskVine manager and connected workers are required for analysis execution. Ensure the manager name matches (`--manager-name`).
-   **Paths:** Be mindful of environment-specific paths:
    -   Input data path (default: `/project01/ndcms/jzhou24/samples`, configurable in `ecf_calculator.py`).
    -   TaskVine run info path (default: `/afs/crc.nd.edu/user/...` or `./vine-run-info`).
    -   Poncho package location (must be on a shared filesystem).
    Adapt these paths as needed for your environment.
-   **Output/Log Directories:** The `output/` directory and the TaskVine run info directory (e.g., `./vine-run-info`) are created automatically if they don't exist.
-   **`--show-samples`:** Use this to confirm the exact names of available datasets before using `--sub-dataset`.
-   **`--sporadic-failure` Argument:** This argument, seen in some `run_batch_*.sh` scripts, is **not** part of `ecf_calculator.py`. It's likely intended for external TaskVine worker control during fault-tolerance testing.
-   **Resource Matching:** Remember to keep `factory.json` (`cores`) and `ecf_calculator.py` (`lib_resources={'cores': ...}`) synchronized if you adjust worker core counts. 