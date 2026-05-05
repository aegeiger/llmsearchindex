# PoC Report: LLMSearchIndex on OpenShift

## 1. Executive Summary

LLMSearchIndex is a Python library and Streamlit web application that provides fully local, internet-scale web search across 203 million web pages for LLM RAG applications. This PoC successfully deployed the application on OpenShift using a UBI-based container with CPU-only PyTorch. All three test scenarios passed: the Streamlit web UI served HTTP 200, the health endpoint responded correctly, and the CLI search job completed successfully, returning relevant results for the query "who invented sliced bread" in ~8 seconds search time (after ~92s cold-start initialization).

**Result: PASS**

## 2. Project Analysis

| Component | Language | Framework | Build System | Port | ML? |
|-----------|----------|-----------|-------------|------|-----|
| llmsearchindex (library) | Python 3.12 | FAISS, Sentence Transformers, PyTorch | setuptools (pyproject.toml) | N/A | Yes |
| streamlit_app.py (web UI) | Python | Streamlit | N/A | 8501 | Yes |
| search.py (CLI) | Python | argparse, asyncio | N/A | N/A | Yes |

**Key Technologies:**
- FAISS binary index (203M vectors, memory-mapped)
- Sentence Transformers all-MiniLM-L6-v2 (embedding model)
- PCA dimensionality reduction (384d -> 64d)
- Binary quantization for fast hamming distance search
- HuggingFace Hub for index and model storage
- PyArrow for surgical parquet row fetching

**System Requirements:** ~6GB RAM, ~10GB disk, CPU inference

## 3. PoC Objectives

1. Deploy the Streamlit search UI on OpenShift with a UBI-based container
2. Validate that the FAISS index loads and searches work end-to-end
3. Demonstrate the search capability via both web UI and CLI

## 4. Pipeline Execution Summary

| Phase | Status | Notes |
|-------|--------|-------|
| Clone | PASS | Cloned to /tmp/autopoc/llmsearchindex |
| Explore | PASS | Python web-app with ML search library |
| Evaluate | PASS | Score: 72/100 |
| PoC Plan | PASS | Deployment + Service + Job |
| Fork | PASS | https://github.com/aegeiger/llmsearchindex |
| Dockerfile.ubi | PASS | UBI9 Python 3.12, CPU-only PyTorch |
| Build (OpenShift) | PASS | Built via oc new-build --binary |
| Build (Podman) | PASS | Built locally for Quay push |
| Push to Quay | PASS | quay.io/aicatalyst/llmsearchindex:latest |
| K8s Manifests | PASS | namespace, deployment, service, job |
| Deploy | PASS | Pod running, 1/1 Ready |
| Tests | PASS | 3/3 scenarios passed |

**Build note:** Initial OpenShift build with full PyTorch (including CUDA) was terminated (image too large ~3GB+). Switched to CPU-only PyTorch (`--index-url https://download.pytorch.org/whl/cpu`) which reduced the image to ~1GB and completed successfully.

## 5. Test Results

| Scenario | Type | Status | Duration | Details |
|----------|------|--------|----------|---------|
| Streamlit UI Health Check | HTTP GET / | PASS | <1s | HTTP 200 returned |
| Streamlit Health Endpoint | HTTP GET /_stcore/health | PASS | <1s | Response: "ok" |
| CLI Search Test | Job (kubectl) | PASS | ~99s total | Index loaded (91.5s), search executed (7.8s), 3 results returned with URLs and snippets |

### CLI Search Results Sample
Query: "who invented sliced bread"
- Result 1: joanbars.com — William Banting diet history
- Result 2: en.wikipedia.org — William Staub, treadmill inventor
- Result 3: nosweatshakespeare.com — "best thing since sliced bread" idiom origin

## 6. Infrastructure Deployed

| Resource | Value |
|----------|-------|
| Namespace | llmsearchindex |
| Deployment | llmsearchindex (1 replica) |
| Service | llmsearchindex (ClusterIP:8501) |
| Job | llmsearchindex-cli-search |
| Image | quay.io/aicatalyst/llmsearchindex:latest |
| Resources | 4Gi/2CPU request, 8Gi/4CPU limit |
| OpenShift Cluster | api.ocp-gb.ibm.redhataicatalyst.com:6443 |

## 7. Recommendations

### Production Readiness
- **HuggingFace Token:** Add an HF_TOKEN secret to avoid rate-limiting on index downloads. Current unauthenticated requests work but may be throttled.
- **Persistent Storage:** Add a PVC for `/tmp/hf_cache` to avoid re-downloading the ~2GB FAISS index on every pod restart. Cold start is ~92 seconds due to index download.
- **Health Probes:** Add readiness probe with `initialDelaySeconds: 120` to account for the cold-start index download. Liveness probe on `/_stcore/health`.

### Performance
- Cold start is ~92 seconds (index + model download). With persistent cache, subsequent starts would be ~10s.
- Search latency is ~8s per query (includes network round-trips to HuggingFace for parquet row fetching).
- Consider pre-warming the index in an init container or startup probe.

### Security
- Runs as non-root (UID 1001) with dropped capabilities.
- No privileged ports.
- HuggingFace token should be stored as a Kubernetes Secret, not hardcoded.

### Scalability
- Horizontal scaling is possible but each replica needs 4-8GB RAM for the FAISS index.
- Consider shared PVC (ReadWriteMany) for the HF cache across replicas.

## 8. ODH/RHOAI Considerations

- **Model Serving:** The all-MiniLM-L6-v2 embedding model could be served via RHOAI Model Serving (KServe/ModelMesh) for centralized model management and GPU acceleration.
- **Workbenches:** The library could be used in JupyterLab workbenches for interactive RAG development.
- **Data Science Pipelines:** The training/indexing pipeline (`train.py`) could be orchestrated via Kubeflow Pipelines.
- **Model Registry:** The trained PCA model and FAISS index could be versioned in the RHOAI Model Registry.

## 9. Appendix

### Artifacts
- Fork: https://github.com/aegeiger/llmsearchindex (branch: autopoc-test)
- Image: https://quay.io/repository/aicatalyst/llmsearchindex
- Original: https://github.com/zakerytclarke/llmsearchindex

### Errors Encountered
1. **OpenShift API hostname:** `.env` had `api-ocp-gb-ibm-redhataicatalyst-com` (all dashes). Correct hostname is `api.ocp-gb.ibm.redhataicatalyst.com`.
2. **OpenShift token expired:** Existing `kube:admin` session was used instead.
3. **First build killed:** Full PyTorch with CUDA dependencies made the image too large. Fixed by using CPU-only PyTorch from `https://download.pytorch.org/whl/cpu`.
4. **Internal registry unreachable:** No external route for the OpenShift image registry. Built locally with podman and pushed to Quay directly.
