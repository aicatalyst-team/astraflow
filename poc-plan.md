# PoC Plan: astraflow

## Project Classification
- **Type:** training
- **Key Technologies:** PyTorch 2.8, Megatron-Core, Hugging Face Transformers, Ray, Hydra/OmegaConf, FastAPI, Redis, PEFT, WandB
- **ODH Relevance:** AstraFlow is a large-scale asynchronous reinforcement learning system for training LLMs. It is directly relevant to OpenShift AI's mission of supporting ML training workloads at scale. While a full multi-node RL training run requires a GPU cluster, the PoC validates that the system can be containerized, installed, and its core orchestration framework initialized on OpenShift.

## PoC Objectives
What we want to prove:
1. The AstraFlow Python package can be successfully built and installed inside a container image on OpenShift
2. All core orchestration classes (`AstraFlow`, `AstraFlowService`, `RolloutBuffer`, `DataAcquisition`, `DataServing`, `EvalManager`) are importable and functional
3. Key ML dependencies (PyTorch, Transformers, Ray, Megatron-Core) are correctly installed and accessible
4. The container image is a viable base for running AstraFlow training jobs on an OpenShift AI cluster

## Infrastructure Requirements
- **Inference Server:** none
- **Vector Database:** none
- **Embedding Model:** none
- **GPU Required:** No (PoC validates package installation and import only; full training requires GPUs)
- **Persistent Storage:** none
- **Resource Profile:** medium (1Gi RAM, 500m CPU — PyTorch and Transformers imports are memory-intensive)
- **Sidecar Containers:** none

## Test Scenarios

### Scenario 1: import-check
- **Description:** Verify that the `astraflow` package imports successfully and exposes its version
- **Type:** cli
- **Input:** `python -c "import astraflow; print('version:', astraflow.__version__)"`
- **Expected:** Job exits 0, prints the astraflow version string
- **Timeout:** 60s

### Scenario 2: core-classes-accessible
- **Description:** Verify that the core orchestration classes (AstraFlow, AstraFlowService, RolloutBuffer, etc.) are importable from the top-level package
- **Type:** cli
- **Input:** `python -c "from astraflow import AstraFlow, AstraFlowService, RolloutBuffer, DataAcquisition, DataServing, EvalManager; print('All core classes imported successfully')"`
- **Expected:** Job exits 0, prints confirmation message
- **Timeout:** 60s

### Scenario 3: dataflow-module-check
- **Description:** Verify that the dataflow submodule and its registries (FILTER_REGISTRY, REPLAY_SELECTION_REGISTRY) are functional
- **Type:** cli
- **Input:** `python -c "from astraflow import FILTER_REGISTRY, REPLAY_SELECTION_REGISTRY, get_filter, get_replay_selection; print('Filter registry:', FILTER_REGISTRY); print('Replay selection registry:', REPLAY_SELECTION_REGISTRY)"`
- **Expected:** Job exits 0, prints the registries with their registered entries
- **Timeout:** 60s

### Scenario 4: dependencies-check
- **Description:** Verify that key dependencies (torch, transformers, ray, hydra) are installed and importable
- **Type:** cli
- **Input:** `python -c "import torch; import transformers; import ray; import hydra; print('torch:', torch.__version__); print('transformers:', transformers.__version__); print('ray:', ray.__version__)"`
- **Expected:** Job exits 0, prints version strings for all dependencies
- **Timeout:** 120s

## Dockerfile Considerations

This is a **training framework / library** — it does NOT run as a long-lived server. The Dockerfile should:

- Use a Python 3.10+ base image (e.g., `python:3.10-slim` or a PyTorch base image like `pytorch/pytorch:2.8.0-cuda12.1-cudnn9-runtime` if available, otherwise a standard Python image)
- Install system-level build dependencies needed for compiled packages (gcc, g++, cmake for pybind11/ninja/numba)
- Copy the repository and install via `pip install .` (the project uses `pyproject.toml` with setuptools)
- **Do NOT add EXPOSE** — there is no port to expose. This is a CLI/library package.
- **ENTRYPOINT** should be `["python"]` with **CMD** defaulting to `["-c", "import astraflow; print(astraflow.__version__)"]`
- Note: Some dependencies like `megatron-core==0.13.1`, `mbridge==0.13.0`, `torch_memory_saver`, `mathruler`, and `swanboard`/`swanlab` may require specific index URLs or may fail on CPU-only builds. The Dockerfile should handle installation failures for GPU-specific optional deps gracefully (e.g., install with `--no-deps` for problematic packages, or use a try/except approach).
- The `flash-attn` and `transformer-engine` optional deps should NOT be installed in the PoC (they require GPU compilation).
- Consider using `--extra-index-url https://download.pytorch.org/whl/cpu` for CPU-only PyTorch to reduce image size, unless GPU support is needed.

## Deployment Considerations

- **Deployment Model:** Job — the process runs a command and exits. Do NOT deploy as a Deployment — there is no long-running server, and a Deployment would CrashLoopBackOff.
- **Do NOT create a Service** — there is no port to expose.
- **Testing:** Each test scenario should be run as a separate Kubernetes Job. The Job runs the specified Python command, and success is verified by checking the Job's exit code (0 = success) and output via `kubectl logs`.
- **Resource requests:** 1Gi RAM minimum (PyTorch + Transformers imports are heavy), 500m CPU.
- **No external dependencies required** for the PoC — no database, no Redis, no model downloads. The PoC only validates that the package installs and its Python API is accessible.
- **Full training runs** would require GPU nodes, Ray cluster setup, model weight downloads, and dataset access — those are out of scope for this initial PoC. The PoC establishes that the containerized package is a working foundation.