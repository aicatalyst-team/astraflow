# PoC Report: AstraFlow — Asynchronous RL Training System for LLMs

## 1. Executive Summary

AstraFlow, a large-scale dataflow-oriented asynchronous reinforcement learning system for training (multi-)agentic LLMs, was evaluated for containerization and deployment readiness on OpenShift AI. The PoC objective was to validate that AstraFlow and its extensive ML dependency stack (PyTorch, Megatron-Core, Transformers, Ray) could be packaged into a container image, deployed as Kubernetes Jobs, and have its core modules verified through import checks. The container image was **successfully built and pushed**, and all four test Jobs were **successfully deployed** to the cluster. However, the PoC execution phase **failed due to an RBAC permission error** — the service account lacked permission to list events in the target namespace, preventing the test harness from collecting Job results. The underlying Jobs may have succeeded, but results could not be verified programmatically.

---

## 2. Project Analysis

- **Repository URL:** `https://github.com/Infini-AI-Lab/astraflow`
- **Project Name:** AstraFlow
- **Repository Summary:** AstraFlow is a large-scale dataflow-oriented asynchronous reinforcement learning system for training (multi-)agentic LLMs. Built primarily in Python, it leverages PyTorch, Megatron-Core, Transformers, Ray, and other ML/distributed computing frameworks. It supports fully async multi-policy collaborative RL, elastic heterogeneous cross-region rollouts, and composable data algorithms. The repository includes the core AstraFlow system, environment components (ASearcher, AgentBench), and various training recipes.

### Components Detected

| Component | Language | Build System | ML Workload | Port |
|-----------|----------|-------------|-------------|------|
| astraflow | Python | pip | Yes | None |

- **Project Classification:** Training workload
- **Technologies and Frameworks:**
  - PyTorch 2.8
  - Megatron-Core
  - Hugging Face Transformers
  - Ray (distributed compute)
  - Hydra / OmegaConf (configuration)
  - FastAPI (service endpoints)
  - Redis (message brokering)
  - PEFT (parameter-efficient fine-tuning)
  - WandB (experiment tracking)
- **Existing CI/CD:** GitHub Actions

---

## 3. PoC Objectives

### What We Set Out to Prove

1. The AstraFlow Python package can be successfully built and installed inside a container image on OpenShift.
2. All core orchestration classes (`AstraFlow`, `AstraFlowService`, `RolloutBuffer`, `DataAcquisition`, `DataServing`, `EvalManager`) are importable and functional.
3. Key ML dependencies (PyTorch, Transformers, Ray, Megatron-Core) are correctly installed and accessible.
4. The container image is a viable base for running AstraFlow training jobs on an OpenShift AI cluster.

### Why This Project Is Relevant to Open Data Hub / OpenShift AI

AstraFlow directly addresses the large-scale ML training use case that OpenShift AI targets. Its asynchronous RL training architecture, multi-policy collaborative training, and support for heterogeneous compute make it an excellent candidate for leveraging ODH's distributed training infrastructure, Data Science Pipelines, and resource management capabilities.

### Infrastructure Requirements Identified

| Requirement | Value |
|-------------|-------|
| Inference Server | None |
| Vector Database | None |
| Embedding Model | None |
| GPU Required | No (PoC only; production requires GPUs) |
| Persistent Storage | None |
| Resource Profile | Medium (1Gi RAM, 500m CPU) |
| Sidecar Containers | None |
| Deployment Model | Kubernetes Job |
| Test Strategy | CLI-based import checks |

---

## 4. Pipeline Execution

### Intake

The intake phase analyzed the AstraFlow repository at `/workspace/astraflow` (source: `https://github.com/Infini-AI-Lab/astraflow`). A single Python component was detected using `pip` as its build system. The project was classified as a **training** workload with no service port (it is not a long-running server for PoC purposes). Existing CI/CD via GitHub Actions was noted.

### PoC Plan

The plan defined four CLI-based test scenarios to validate package installation and import correctness:

1. **import-check** — Verify `astraflow` imports and exposes its version
2. **core-classes-accessible** — Verify core orchestration classes are importable
3. **dataflow-module-check** — Verify dataflow submodule registries are functional
4. **dependencies-check** — Verify PyTorch, Transformers, Ray, and Hydra imports

Infrastructure was set to `medium` profile (500m CPU, 1Gi RAM). No GPU, PVC, or sidecar containers were required.

### Fork

The project was forked to a GitLab repository for artifact management. Build artifacts and deployment manifests were committed to the `autopoc-artifacts` branch.

### Containerize

A single Dockerfile was generated for the `astraflow` component:

- `Dockerfile.astraflow` — Python-based image installing AstraFlow and all its dependencies via `pip`

### Build

| Image | Tag | Status | Retries |
|-------|-----|--------|---------|
| `quay.io/aicatalyst/astraflow-astraflow` | `latest` | ✅ Built & Pushed | 0 |

The image was built successfully on the first attempt with no retries required. This confirms that the full AstraFlow dependency tree (including PyTorch 2.8, Megatron-Core, Ray, Transformers, PEFT, etc.) can be resolved and installed in a container environment.

### Deploy

All four Kubernetes Jobs were deployed successfully to the `astraflow` namespace with **0 retries**:

| Resource | Type | Status |
|----------|------|--------|
| `astraflow` | Namespace | ✅ Created |
| `astraflow-import-check` | Job | ✅ Deployed |
| `astraflow-core-classes-accessible` | Job | ✅ Deployed |
| `astraflow-dataflow-module-check` | Job | ✅ Deployed |
| `astraflow-dependencies-check` | Job | ✅ Deployed |

No routes were created (expected — this is a Job-based PoC with no HTTP endpoints).

### PoC Execute

The PoC execution agent attempted to monitor the deployed Jobs and collect results but **failed due to an RBAC permission error**. The service account `system:serviceaccount:autopoc-test:autopoc-runner` lacked permission to list events in the `astraflow` namespace.

---

## 5. Test Results

| Scenario | Status | Duration | Details |
|----------|--------|----------|---------|
| import-check | ❌ ERROR | N/A | Could not verify — RBAC error prevented result collection |
| core-classes-accessible | ❌ ERROR | N/A | Could not verify — RBAC error prevented result collection |
| dataflow-module-check | ❌ ERROR | N/A | Could not verify — RBAC error prevented result collection |
| dependencies-check | ❌ ERROR | N/A | Could not verify — RBAC error prevented result collection |

**Aggregate:** 0/4 verified (all 4 inconclusive due to infrastructure error)

### Root Cause of Failure

The PoC execution phase failed before any individual test scenario could be evaluated. The failure was **not** in the application code or container image — it was an RBAC/permissions issue in the test harness infrastructure.

**Error:**
```
Error from server (Forbidden): events is forbidden: User "system:serviceaccount:autopoc-test:autopoc-runner" 
cannot list resource "events" in API group "" in the namespace "astraflow"
```

### What Went Wrong

The `autopoc-runner` service account in the `autopoc-test` namespace does not have a `Role` or `ClusterRole` granting `list` permission on `events` resources in the `astraflow` namespace. The test execution agent uses `kubectl get event` to monitor Job completion and collect results.

### Suggestions for Fixing

1. **Create a Role/RoleBinding** in the `astraflow` namespace granting the `autopoc-runner` service account permission to `get`, `list`, and `watch` the following resources:
   - `events`
   - `jobs`
   - `pods`
   - `pods/log`

   ```yaml
   apiVersion: rbac.authorization.k8s.io/v1
   kind: Role
   metadata:
     name: autopoc-runner-role
     namespace: astraflow
   rules:
   - apiGroups: [""]
     resources: ["events", "pods", "pods/log"]
     verbs: ["get", "list", "watch"]
   - apiGroups: ["batch"]
     resources: ["jobs"]
     verbs: ["get", "list", "watch"]
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: RoleBinding
   metadata:
     name: autopoc-runner-binding
     namespace: astraflow
   subjects:
   - kind: ServiceAccount
     name: autopoc-runner
     namespace: autopoc-test
   roleRef:
     kind: Role
     name: autopoc-runner-role
     apiGroup: rbac.authorization.k8s.io
   ```

2. **Alternatively**, use a `ClusterRole` with a `ClusterRoleBinding` to grant cross-namespace access for the autopoc pipeline.

3. **Re-run the PoC execution phase** after fixing RBAC permissions.

---

## 6. Infrastructure Deployed

| Property | Value |
|----------|-------|
| **Kubernetes Namespace** | `astraflow` |
| **Container Image** | `quay.io/aicatalyst/astraflow-astraflow:latest` |
| **Image Registry** | Quay.io (`aicatalyst` organization) |

### Kubernetes Resources Created

| Resource Type | Name | Status |
|--------------|------|--------|
| Namespace | `astraflow` | Created |
| Job | `astraflow-import-check` | Deployed |
| Job | `astraflow-core-classes-accessible` | Deployed |
| Job | `astraflow-dataflow-module-check` | Deployed |
| Job | `astraflow-dependencies-check` | Deployed |

### Resource Allocations

| Resource | Request | Limit |
|----------|---------|-------|
| CPU | 500m | 500m |
| Memory | 1Gi | 1Gi |

- **Service URLs / Routes:** None (Job-based workload)
- **Sidecar Containers:** None
- **PVCs:** None
- **GPU:** None (PoC phase only)

---

## 7. Recommendations

### Production Readiness

**Not production-ready.** While the container image builds successfully, the PoC test results are inconclusive due to an infrastructure error. Before declaring production readiness:

1. Fix the RBAC issue and re-run all four test scenarios.
2. Validate actual training execution with a small model on GPU-equipped nodes.
3. Add health checks and resource monitoring for long-running training jobs.
4. Implement proper secret management for WandB API keys and Redis credentials.

### Performance

- **PyTorch 2.8 and Megatron-Core** are memory-intensive even at import time. The 1Gi RAM allocation for the PoC is marginal; production training workloads will require **16-128 Gi RAM** per pod depending on model size.
- AstraFlow's async architecture with Ray requires careful tuning of Ray cluster resources (head node vs. worker nodes).
- Production deployments should use NVIDIA GPU operator with at minimum A100 or H100 GPUs for RL training.

### Security

- **Container image provenance:** The base image and all pip packages should be scanned for CVEs before production use.
- **Redis:** If used as a message broker in production, Redis must be configured with authentication and TLS.
- **FastAPI endpoints:** AstraFlow's service components expose HTTP APIs that need authentication and network policies.
- **Model weights:** Access to Hugging Face model repositories may require API tokens stored as Kubernetes Secrets.
- **WandB:** Experiment tracking credentials should be injected via Secrets, not environment variables in manifests.

### Scalability

- AstraFlow is designed for multi-node, multi-GPU training with Ray as the orchestration layer. Scaling requires:
  - A Ray cluster (head + workers) deployed on OpenShift
  - NCCL/RDMA networking for inter-GPU communication
  - Shared storage (e.g., Ceph/NFS PVCs) for checkpoints and datasets
- The elastic heterogeneous rollout feature suggests it can handle mixed GPU types, which is valuable for heterogeneous OpenShift clusters.
- Consider using the KubeRay operator for managing Ray clusters on OpenShift.

### Next Steps

1. **Immediate:** Fix RBAC permissions for the `autopoc-runner` service account and re-execute the PoC test phase.
2. **Short-term:** Run a minimal training example (e.g., a small policy gradient update on a toy model) on a GPU-equipped node to validate end-to-end functionality.
3. **Medium-term:** Deploy a Ray cluster alongside AstraFlow using KubeRay and validate distributed rollout functionality.
4. **Long-term:** Integrate with ODH Data Science Pipelines for orchestrating training runs and Model Registry for checkpoint management.

---

## 8. Open Data Hub / OpenShift AI Considerations

### Relevant ODH Components

| ODH Component | Relevance | Notes |
|---------------|-----------|-------|
| **Data Science Pipelines** | High | Orchestrate multi-stage RL training workflows (data prep → rollout → training → evaluation) |
| **Workbenches** | Medium | JupyterLab workbenches for interactive development and debugging of AstraFlow configs |
| **Model Registry** | High | Track trained model checkpoints, policy versions, and RL training metadata |
| **TrustyAI** | Medium | Monitor model performance drift and reward function behavior over training iterations |
| **Model Serving (KServe)** | Low (for PoC) | Not relevant for training phase; useful later for serving trained policies |
| **ModelMesh** | Low | Not directly applicable to training workloads |

### Migration Path: Vanilla K8s → ODH-Managed Deployment

1. **Phase 1 — Current State:** Vanilla Kubernetes Jobs with manually built container images
2. **Phase 2 — KubeRay Integration:** Deploy Ray clusters using KubeRay operator on OpenShift; configure AstraFlow to use the Ray cluster endpoint
3. **Phase 3 — DSP Integration:** Wrap AstraFlow training stages as Data Science Pipeline tasks (Argo Workflow steps or Tekton tasks)
4. **Phase 4 — Full ODH:** Use ODH Workbenches for config development, DSP for orchestration, Model Registry for checkpoint tracking, TrustyAI for reward monitoring

### ODH-Specific Feature Recommendations

- **Data Science Pipelines:** Define an Argo/Tekton pipeline with steps: `prepare-dataset` → `start-ray-cluster` → `run-astraflow-training` → `evaluate-policy` → `register-model`. AstraFlow's Hydra-based configuration makes parameterization straightforward.
- **Model Registry:** Register each RL policy checkpoint with metadata including reward curves, training hyperparameters, and environment configuration.
- **Workbenches:** Provide JupyterLab environments pre-loaded with AstraFlow and its dependencies for interactive development of training recipes and reward functions.
- **TrustyAI:** Monitor reward signal statistics and policy behavior metrics across training iterations to detect training instability or reward hacking.

---

## 9. Appendix

### Artifact Links

| Artifact | Location |
|----------|----------|
| PoC Plan | `poc-plan.md` (on `autopoc-artifacts` branch) |
| Test Script | `poc_test.py` (on `autopoc-artifacts` branch) |
| Dockerfile | `Dockerfile.astraflow` (on `autopoc-artifacts` branch) |
| K8s Manifests | Deployment manifests (on `autopoc-artifacts` branch) |
| Test Output | `poc-test-output/` (on `autopoc-artifacts` branch) |
| Container Image | `quay.io/aicatalyst/astraflow-astraflow:latest` |
| Source Repository | `https://github.com/Infini-AI-Lab/astraflow` |

### Build Errors Encountered

None. The container image built successfully on the first attempt.

### Deploy Errors Encountered

None. All four Jobs and the namespace were created successfully with zero retries.

### Execution Errors Encountered

| Error | Phase | Details |
|-------|-------|---------|
| RBAC Forbidden | PoC Execution | `events is forbidden: User "system:serviceaccount:autopoc-test:autopoc-runner" cannot list resource "events" in API group "" in the namespace "astraflow"` |

### Retry Summary

| Phase | Retries |
|-------|---------|
| Build | 0 |
| Deploy | 0 |
| Execute | 0 (failed immediately on RBAC error) |

### Manual Verification Commands

To manually verify the deployed Jobs after fixing RBAC:

```bash
# Check Job status
kubectl get jobs -n astraflow

# View logs for each Job
kubectl logs -n astraflow job/astraflow-import-check
kubectl logs -n astraflow job/astraflow-core-classes-accessible
kubectl logs -n astraflow job/astraflow-dataflow-module-check
kubectl logs -n astraflow job/astraflow-dependencies-check

# Check pod status
kubectl get pods -n astraflow
```
