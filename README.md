## Prerequisites

Install the required tools for building cctools:
```bash
unset PYTHONPATH
# Create conda environment
conda create -y -n dv5 -c conda-forge --strict-channel-priority python=3.10
conda activate dv5
# Install build requirements
conda install -y gcc_linux-64 gxx_linux-64 gdb m4 perl swig make zlib libopenssl-static openssl conda-pack packaging cloudpickle flake8 clang-format threadpoolctl
```

Build and install the latest version of cctools (which includes TaskVine):
```bash
git clone https://github.com/JinZhou5042/cctools.git
cd cctools
git fetch && git branch origin/sc25
git switch sc25
# Configure to install within the conda environment
./configure --with-base-dir $CONDA_PREFIX --prefix $CONDA_PREFIX
make -j8 && make install
# Verify installation
vine_worker --version   # You should see output indicating the cctools version
```

Install the required Python packages for this repository:
```bash
pip install dask==2024.7.1 dask-awkward==2024.7.0 awkward==2.6.6 coffea==2024.4.0 fastjet==3.4.2.1
```
These specific package versions have been tested and are known to work with this code. Other versions might work but are not guaranteed.

Create the poncho package for remote workers
``` bash
# first go to the shared filesystem where both local and remote workers can access
cd /scratch365/[USERNAME]/
poncho_package_create $CONDA_PREFIX sc25_env.tar.gz
# this might take a while
```

In the shared directory, create a factory.json as the specification for connecting workers:
``` bash
{
    "manager-name": "jzhou24-hgg7",
    "max-workers": 30,
    "min-workers": 30,
    "workers-per-cycle": 30,
    "condor-requirements": "((has_vast)",
    "cores": 32,
    "memory": 16*1024,
    "disk": 800*1024,
    "timeout": 36000
}
```

Then, start the factory with the following command:
``` bash
vine_factory -T condor -C factory.json --scratch-dir /scratch365/[USERNAME]/vi
ne_scratch --python-env sc25_env.tar.gz -d all -o factory.debug
```

Now the environment are prepared.


## Dataset Structure

Clone this project to your shared filesystem:
```
git clone https://github.com/JinZhou5042/dv5.git
cd dv5
```

Download the dataset to where this project is located:
```
cd 
```

The input data should be organized in the following directory structure. By default, the script looks for this structure under `/project01/ndcms/jzhou24/samples` (this path might need adjustment inside the `ecf_calculator.py` script):
```
samples/
├── hgg/           # Each subdirectory represents a sub-dataset
├── hgg_1/         # You can process all sub-datasets together or
├── hgg_2/         # choose to process only one specific
├── hgg_3/         # sub-dataset at a time
└── ...
```

Each sub-dataset directory must contain the input ROOT files for that dataset.

### Dataset Management

- If you have a large number of sub-datasets, the initial preprocessing step might take a significant amount of time.
- To speed up preprocessing:
  1. You can temporarily move unnecessary sub-dataset directories out of the main `samples/` directory.
  2. After running the preprocessing step (`--preprocess`), you can move them back if needed for future runs.
- To create new sub-datasets:
  1. Simply copy an existing sub-dataset directory (e.g., `cp -r samples/hgg samples/my_new_hgg`).
  2. Rename the copied directory to your desired new dataset name.
  3. The new sub-dataset will be automatically detected and included the next time you run the preprocessing step (`--preprocess`). Remember to re-run preprocessing if you add new datasets.

## Usage

The script uses DaskVine for distributed computation and typically involves several steps as outlined below.

```bash
python ecf_calculator.py [options]
```

### Core Workflow Steps

1.  **Preprocessing:** (Required once initially, and whenever datasets are added, removed, or changed)
    ```bash
    # Scans the samples/ directory, generates samples_ready.json, and then exits
    python ecf_calculator.py --preprocess
    ```
    This step prepares a JSON file (`samples_ready.json`) containing the file lists and metadata for all detected datasets, which is essential for the subsequent analysis steps.

2.  **Show Available Samples:** (Optional, to check detected datasets)
    ```bash
    # Lists the datasets found in samples_ready.json and their file counts
    python ecf_calculator.py --show-samples
    ```

3.  **Analysis Run (Generating and Executing Tasks):**
    ```bash
    # Basic run: processes the default 'hgg' dataset with default ECF settings (n<=3)
    # This generates the Dask task graph based on samples_ready.json and executes it using DaskVine.
    python ecf_calculator.py

    # Process a specific dataset (must exist in samples_ready.json):
    python ecf_calculator.py --sub-dataset hgg_1

    # Process all datasets listed in samples_ready.json:
    python ecf_calculator.py --all

    # Process all datasets with a higher ECF calculation bound (e.g., up to n=5):
    python ecf_calculator.py --ecf-upper-bound 5 --all
    ```

### Task Checkpointing (Optional but Recommended for Batch Runs or Re-runs)

This feature allows you to separate the Dask task graph *generation* (which can be time-consuming, especially with many files/datasets) from the actual *computation*. You generate the task graph once and save it to a file. Then, you can load and execute this saved graph multiple times, which is useful for testing different DaskVine tuning parameters without regenerating the graph each time.

1.  **Generate and Save Tasks:**
    ```bash
    # Run the analysis command as usual, but add --checkpoint-to <filename>
    # This generates the tasks based on the selected datasets and saves the graph
    # to 'my_tasks.pkl' *without* starting the computation.
    python ecf_calculator.py --all --ecf-upper-bound 5 --checkpoint-to my_tasks.pkl
    ```

2.  **Load and Execute Tasks:**
    ```bash
    # Use --load-from <filename> instead of dataset selection arguments (--all/--sub-dataset)
    # This loads the pre-generated task graph from 'my_tasks.pkl' and executes it.
    # You can combine this with various DaskVine configuration arguments.
    python ecf_calculator.py --load-from my_tasks.pkl [DaskVine options...]
    ```
    *Important Note: The `--checkpoint-to` and `--load-from` arguments are mutually exclusive; you cannot use both in the same command.*

### DaskVine Configuration Arguments

These command-line arguments control the interaction with the DaskVine distributed computing framework and allow tuning its behavior. They are typically applied when *executing* tasks (whether generated on-the-fly or loaded using `--load-from`).

**Manager Connection and Run Information:**

-   `--manager-name <name>`: Specify the name of the TaskVine manager to connect to. Defaults to a name incorporating the username (e.g., `{user}-hgg7`). Ensure this matches your running TaskVine manager's name.
-   `--template <template_name>`: Specifies a subdirectory name within the `run_info_path` where TaskVine run logs and reports will be stored. This is very useful for organizing outputs from different runs or experiments. The script first attempts to use a shared path (`/afs/crc.nd.edu/user/{U}/{USER}/taskvine-report-tool/logs`), but if that's not accessible, it falls back to creating `./vine-run-info`. The specified template directory will be created under the determined `run_info_path`.
-   `--enforce-template`: If the directory specified by `--template` already exists, delete it and its contents without asking for confirmation. Use this option with caution, as it can lead to data loss.

**General DaskVine Tuning:**

-   `--wait-for-workers <seconds>`: Specify the number of seconds the TaskVine manager should wait for workers to connect before potentially starting task execution.
-   `--disable-worker-join-after-first-run`: If set, prevent new workers from joining the computation after it has already started.
-   `--max-workers <count>`: Set the maximum number of workers the TaskVine manager will utilize for this computation.

**Fault Tolerance Tuning:**

-   `--temp-replica-count <count>`: Set the number of replicas TaskVine should maintain for temporary intermediate files created during computation (default: 1). Increasing this enhances resilience against worker failures but increases storage usage.
-   `--checkpoint-threshold <seconds>`: Define the minimum time (in seconds) an intermediate result must be kept in memory before TaskVine considers checkpointing it to more persistent storage (default: 30). This influences the aggressiveness of checkpointing for recovery.
-   `--enforce-worker-eviction-interval <seconds>`: Primarily for testing fault tolerance mechanisms. If set, TaskVine will periodically force workers to be evicted from the computation at the specified interval (in seconds).

**Performance and Scheduling Tuning:**

-   `--load-balancing`: Enable TaskVine's dynamic load balancing feature, which attempts to distribute tasks more evenly among workers.
-   `--load-balancing-interval <seconds>`: When load balancing is enabled, this sets how frequently (in seconds) TaskVine checks and potentially redistributes tasks (default: 3). Requires `--load-balancing`.
-   `--load-balancing-factor <factor>`: When load balancing is enabled, this sets the target ratio between the task queue size of the busiest and least busy workers (default: 1.1). A lower factor aims for more even balancing. Requires `--load-balancing`.
-   `--prune-depth <depth>`: Specify the Dask graph pruning depth (default: 0). May reduce graph complexity in some cases, potentially improving scheduling performance.
-   `--scheduling-mode <mode>`: Choose the task scheduling strategy used by DaskVine (default: `LIFO` - Last-In, First-Out). Other modes, such as `storage-footprint-aware`, might be available depending on the specific TaskVine version and configuration, potentially optimizing for different resource constraints.

### Examples

1.  **Minimal Workflow (Preprocess + Analyze default 'hgg' dataset):**
    ```bash
    # Step 1: Preprocess the data in the samples/ directory
    python ecf_calculator.py --preprocess

    # Step 2: Run the analysis using default settings
    python ecf_calculator.py
    ```

2.  **Analyze a Specific Dataset with Higher ECF Bound:**
    ```bash
    # Assumes preprocessing has already been completed
    python ecf_calculator.py --sub-dataset hgg_2 --ecf-upper-bound 6
    ```

3.  **Workflow using Task Checkpointing:**
    ```bash
    # Step 1: Preprocess (if samples/ changed or samples_ready.json doesn't exist)
    python ecf_calculator.py --preprocess

    # Step 2: Generate the task graph for all datasets (ECF bound 5) and save it
    python ecf_calculator.py --all --ecf-upper-bound 5 --checkpoint-to all_tasks_ecf5.pkl

    # Step 3: Load the saved task graph and execute it with specific DaskVine settings
    #         (e.g., higher replication, custom template dir, specific manager)
    python ecf_calculator.py --load-from all_tasks_ecf5.pkl --temp-replica-count 3 --template run_replication_3 --manager-name my-analysis-manager
    ```

4.  **Batch Experiments (Conceptual Overview):**
    The provided `run_batch_*.sh` scripts (e.g., `run_batch_checkpoint.sh`, `run_batch_replication.sh`) serve as practical examples. They typically use a pre-generated task file (via `--load-from`) and then iterate through different DaskVine tuning parameters (like `--checkpoint-threshold` or `--temp-replica-count`), often using the `--template` argument to organize the output logs for each specific parameter combination. Refer to these scripts for concrete usage patterns in systematic studies.

## Output

The primary analysis results, consisting of parquet files containing the calculated physics variables (ECFs, color ring, etc.), will be saved into the `output/{dataset}/` directory structure. A separate subdirectory is created for each processed dataset (e.g., `output/hgg/`, `output/hgg_1/`).

## Notes

-   Ensure you run `--preprocess` first whenever `samples_ready.json` is missing or the contents/structure of the `samples/` directory have been modified.
-   The preprocessing step (`--preprocess`) only generates the `samples_ready.json` file and then exits; the actual analysis requires running the script again without `--preprocess` (or with `--load-from`).
-   Task checkpointing (`--checkpoint-to` to save, `--load-from` to load) is highly recommended for efficiency when running multiple analyses with different parameters or for improving resilience against interruptions.
-   This script requires a functioning TaskVine setup: a TaskVine manager must be running and accessible, and TaskVine workers need to be started and connected to that manager.
-   The output directory (`output/`) and the TaskVine run information directory (`./vine-run-info` or a shared path like `/afs/crc.nd.edu/...`) will be created automatically if they do not already exist.
-   Always use the `--show-samples` command to verify the exact names of available sub-datasets before specifying one using the `--sub-dataset` argument.
-   Be mindful of potentially environment-specific paths hardcoded or defaulted within the script or documentation, such as `/project01/ndcms/jzhou24/samples` (for input data) and `/afs/crc.nd.edu/user/...` (for TaskVine run info). These might need adaptation when running the script in different computing environments.
-   The `--sporadic-failure` argument observed in some of the provided batch scripts (`run_batch_*.sh`) is not an argument accepted by `ecf_calculator.py` itself. It is likely an external argument intended for controlling the behavior of TaskVine components (like workers) during testing, potentially to simulate failures. 