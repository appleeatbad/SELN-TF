SELN-TF
SELN-TF: Self-Evolving LLM-Guided Node-Level Traffic Forecast Correction with Cross-Backbone Transfer. The framework is referred to as SELN-TF in the paper, while the released code, directory names, and command-line interfaces retain the original name SANE-TF. Both names refer to the same framework; the naming difference does not affect its use.

SELN-TF performs adaptive node-level corrections on traffic forecasts using frozen base forecasting models, a Node Residual Corrector (NRC), a Node Utility Estimator (NUE), and Qwen3-8B. Its correction policy is trained through supervised fine-tuning (SFT) and direct preference optimization (DPO), and can be transferred to other forecasting backbones.

The code supports four tasks: traffic flow, traffic speed, regional travel demand, and road occupancy. This repository contains the code for training, testing, and cross-backbone transfer. Datasets, external forecasting-model repositories, and Qwen weights must be downloaded separately.
1. System Requirements
Use a Linux machine with Conda, Python 3.10, an NVIDIA CUDA GPU, and the tools git, gcc, g++, and make. The provided setup uses PyTorch 2.6.0 with CUDA 12.4; the NVIDIA driver must support this CUDA runtime.

Open a terminal in the extracted or cloned project directory:
cd /your/path/SANE-TF
export SANE_TF_RESOURCES="$(cd .. && pwd)"
export SANE_TF_CONDA="$(command -v conda)"
export CONDA_EXE="$SANE_TF_CONDA"
export QWEN_MODEL_PATH="$PWD/models/Qwen3-8B"
SANE_TF_RESOURCES points to the parent directory of SANE-TF, where the datasets and external model repositories will be stored. The commands below use this directory layout.
2. Download External Forecasting Models
Download the repositories required for the tasks you intend to run. You do not need to download all models for a single task. Use the directory names shown below, as the task configurations refer to them.
Directory	Used for	Source
AGCRN-master	Traffic flow backbone	AGCRN
stgcn-main	Traffic flow backbone	STGCN
Bigscity-LibCity-master	GWNET, DCRNN, TGCN, STID, STG2Seq, etc.	LibCity
STAEformer	Road occupancy backbone	STAEformer
SDGFormer-master	Traffic flow transfer	SDGFormer
ESTGformer-main	Traffic speed transfer	ESTGFormer
STGormer-main	Travel demand transfer	STGormer
ST-SSL-main	Travel demand transfer	ST-SSL
iTransformer-main	Road occupancy transfer	iTransformer
TimeMixer-main	Road occupancy transfer	TimeMixer


For example, download the principal upstream repositories:
git clone https://github.com/LeiBAI/AGCRN.git "$SANE_TF_RESOURCES/AGCRN-master"
git clone https://github.com/hazdzz/STGCN.git "$SANE_TF_RESOURCES/stgcn-main"
git clone https://github.com/LibCity/Bigscity-LibCity.git "$SANE_TF_RESOURCES/Bigscity-LibCity-master"
git clone https://github.com/XDZhelheim/STAEformer.git "$SANE_TF_RESOURCES/STAEformer"
git -C "$SANE_TF_RESOURCES/STAEformer" checkout fc49d39b2f1a8e3cf37b6289d7240680e1690f3f
For transfer experiments, clone the corresponding target-model repositories from the table into the specified directories. The LPE-STGTN implementation is already included under lpe_stgtn/.

Before running STGCN, execute:
bash scripts/setup_stgcn.sh
3. Prepare the Datasets
3.1 PEMS08: Traffic Flow and Speed
Download the time-series data and road connectivity information from the ASTGNN PEMS08 dataset directory:
mkdir -p "$SANE_TF_RESOURCES/PEMS08"
curl -fL https://raw.githubusercontent.com/guoshnBJTU/ASTGNN/main/data/PEMS08/PEMS08.npz \
  -o "$SANE_TF_RESOURCES/PEMS08/pems08.npz"
curl -fL https://raw.githubusercontent.com/guoshnBJTU/ASTGNN/main/data/PEMS08/PEMS08.csv \
  -o "$SANE_TF_RESOURCES/PEMS08/distance.csv"
PEMS08 contains 170 traffic nodes sampled every 5 minutes. Channel 0 is used for traffic flow and channel 2 for traffic speed.

If you choose to run the original full-deployment script setup.sh, also prepare PEMS04 with the filenames PEMS04/pems04.npz and PEMS04/distance.csv.
3.2 NYCTAXI: Regional Travel Demand
This task requires two types of data.

1. Hourly regional demand data. Prepare the NYCTAXI201901-202003_DYNA dataset covering January 2019 through March 2020, with the following files:
NYCTAXI201901-202003_DYNA/
├── NYCTAXI201901-202003.dyna
├── NYCTAXI201901-202003.geo
├── NYCTAXI201901-202003.rel
└── config.json
The dataset uses the LibCity atomic-file format, and the prediction target is outflow (channel 1). Consult the LibCity dataset guide and NYCTAXI_DYNA preprocessing script to obtain or generate the data. When preprocessing, use the 2019-01 to 2020-03 period and make sure the region definitions, hourly intervals, and filenames match the project configuration.

2. Original NYC TLC trip records. LPE-STGTN uses the raw yellow-taxi trip records to construct an origin–destination (OD) graph. Download the 15 monthly Parquet files from January 2019 through March 2020, available via the NYC Taxi & Limousine Commission:
mkdir -p "$SANE_TF_RESOURCES/NYC_TLC_YELLOW_201901_202003"
for year in 2019 2020; do
  end=12
  [ "$year" = 2020 ] && end=3
  for month in $(seq 1 "$end"); do
    ym=$(printf '%04d-%02d' "$year" "$month")
    file="yellow_tripdata_${ym}.parquet"
    curl -fL "https://d37ci6vzurychx.cloudfront.net/trip-data/$file" \
      -o "$SANE_TF_RESOURCES/NYC_TLC_YELLOW_201901_202003/$file"
  done
done
3.3 Madrid: Road Occupancy
Madrid traffic records are available through the Madrid Open Data Portal. After installing the environments in Section 4 and preparing the upstream repositories, run:
bash scripts/run_occupancy_madrid.sh --setup
bash scripts/run_occupancy_madrid.sh --check
These commands prepare the MADRID_OCCUPANCY/ directory and its metadata. Keep the generated metadata alongside the data files. Road occupancy uses channel 0, sampled at 15-minute intervals.
4. Install Dependencies and Qwen3-8B
From the SANE-TF project root, create separate environments for native forecasting models and Qwen:
for kind in native qwen; do
  "$SANE_TF_CONDA" create -y --override-channels -c conda-forge \
    -p "$PWD/.envs/$kind" python=3.10 pip
  "$PWD/.envs/$kind/bin/python" -m pip install \
    pip==25.1.1 setuptools==75.8.0 wheel==0.45.1
  "$PWD/.envs/$kind/bin/python" -m pip install \
    torch==2.6.0 --index-url https://download.pytorch.org/whl/cu124
done

.envs/native/bin/python -m pip install numpy==1.26.4 Cython==0.29.37
.envs/native/bin/python -m pip install --no-build-isolation -r requirements-base.txt
.envs/qwen/bin/python -m pip install -r requirements-qwen.txt
.envs/native/bin/python -m pip install 'pyarrow>=15.0' torchdiffeq==0.2.3
.envs/qwen/bin/python -m pip install pandas==2.2.3 'pyarrow>=15.0'

.envs/native/bin/python -m pip check
.envs/qwen/bin/python -m pip check
Download the Qwen3-8B model weights:
.envs/qwen/bin/python - <<'PY'
import os
from huggingface_hub import snapshot_download

snapshot_download(
    repo_id="Qwen/Qwen3-8B",
    local_dir=os.environ["QWEN_MODEL_PATH"],
    allow_patterns=["*.json", "*.safetensors", "*.model", "*.txt", "*.jinja"],
    max_workers=4,
)
PY

export SANE_TASK_PYTHON="$PWD/.envs/qwen/bin/python"
.envs/qwen/bin/python -c 'import torch; assert torch.cuda.is_available(); print(torch.__version__, torch.version.cuda)'
If Qwen3-8B is already available locally, set QWEN_MODEL_PATH to the existing model directory and skip the download. The occupancy branch uses a separate .envs_occupancy/ environment, which is prepared by the --setup command in Section 3.3.
5. Run Training and Cross-Backbone Transfer
The repository includes 12 natural-language task examples. List the available tasks and command-line options:
bash scripts/sane_task_examples.sh --list
bash scripts/run_sane_nl.sh --help
Task ID	Experiment	Prerequisite
1	PEMS08 flow: AGCRN + STGCN	Flow data and environment
2	Flow transfer: GWNET	Task 1
3	PEMS08 speed: DCRNN + TGCN	Speed data and environment
4	Speed transfer: STID	Task 3
5	NYCTAXI demand: STG2Seq + LPE-STGTN	Both NYCTAXI data sources
6	Madrid occupancy: STAEformer + GWNET	Madrid --setup
7	Flow transfer: SDGFormer	Task 1
8	Speed transfer: ESTGFormer	Task 3
9	Demand transfer: STGormer	Task 5
10	Demand transfer: ST-SSL	Task 5
11	Occupancy transfer: iTransformer	Task 6
12	Occupancy transfer: TimeMixer	Task 6


For example, train the traffic flow models and transfer to GWNET:
# Check the dataset, environment, and source code
bash scripts/sane_task_examples.sh --run 1 --check

# Launch the full training pipeline in the background
bash scripts/sane_task_examples.sh --run 1 --background

# Check task status
bash scripts/sane_task_examples.sh --run 1 --status

# Once Task 1 has finished and produced a Bundle, run transfer
bash scripts/sane_task_examples.sh --run 2 --background
The other primary tasks are --run 3 (speed), --run 5 (demand), and --run 6 (occupancy). Run the corresponding transfer tasks after their primary training tasks finish.

Other useful commands:
bash scripts/sane_task_examples.sh --run 1 --foreground  # Run in the foreground
bash scripts/sane_task_examples.sh --run 1 --resume      # Resume an existing task
bash scripts/sane_task_examples.sh --run 1 --stop        # Stop a running task
You can also submit a custom natural-language request through the unified interface. The following example retains the Chinese input used by the original task scripts:
bash scripts/run_sane_nl.sh \
  --task '使用PEMS08，选择DCRNN和TGCN执行交通速度预测，完成NRC、NUE、Qwen SFT、三轮DPO及独立测试。' \
  --background
Experiments follow the chronological A/B/C/D/E/F split (30%/15%/15%/20%/10%/10%) for base-model training, NRC training, NUE training, SFT/DPO, validation, and independent testing, respectively. Successful primary runs produce a Bundle that can be used by the transfer tasks.
6. Find Logs and Results
The startup command prints the run-id and log location. Outputs are generally stored under:
runs/<run-id>/
├── logs/main.log       # Execution log
├── checkpoints/        # Trained model weights
├── bundle.json         # Correction module Bundle information
└── results/             # Test and evaluation results
Some tasks use slightly different output subdirectories. Check the task log for the actual paths.
