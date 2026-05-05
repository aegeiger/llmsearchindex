# PoC Plan: llmsearchindex

## Project Classification
- **Type:** web-app (Streamlit-based RAG search UI)
- **Key Technologies:** Python, Streamlit, FAISS, Sentence Transformers (all-MiniLM-L6-v2), PyTorch, HuggingFace Hub, PyArrow
- **ODH Relevance:** Demonstrates local vector search over 203M web pages for LLM RAG augmentation — a core RHOAI use case. The embedding model and FAISS index could be served via RHOAI model serving in production.

## PoC Objectives
1. Deploy the Streamlit search UI on OpenShift with UBI-based container
2. Validate that the FAISS index loads and searches work end-to-end from within the cluster
3. Demonstrate the search capability with multiple query types

## Infrastructure Requirements
- **Deployment Model:** deployment
- **Listens on Port:** yes (8501 — Streamlit default)
- **Resource Profile:** large (needs ~6GB RAM for FAISS index + embedding model)
- **Inference Server:** none (uses built-in Sentence Transformers)
- **Vector Database:** in-memory (FAISS bundled, index downloaded from HuggingFace at runtime)
- **GPU Required:** no (CPU inference supported)
- **Persistent Storage:** none (index cached in HuggingFace Hub cache dir, ephemeral is acceptable for PoC)
- **Sidecar Containers:** none

## Test Scenarios

### Scenario 1: Streamlit UI Health Check
- **Description:** Verify the Streamlit app is up and serving HTTP
- **Type:** http
- **Input:** GET /
- **Expected:** HTTP 200 with HTML response containing "LLMSearchIndex"
- **Timeout:** 180 (index download + model load on cold start takes time)

### Scenario 2: Streamlit Health Endpoint
- **Description:** Verify Streamlit's built-in health endpoint responds
- **Type:** http
- **Input:** GET /_stcore/health
- **Expected:** HTTP 200 with "ok"
- **Timeout:** 180

### Scenario 3: CLI Search Test
- **Description:** Run the CLI search.py to validate the core search functionality works
- **Type:** cli (Job)
- **Input:** `python search.py "who invented sliced bread" --k 3`
- **Expected:** Exit 0, output contains search results with URLs and snippets
- **Timeout:** 300 (cold start + HuggingFace download)

## Dockerfile Considerations
- Base image: `registry.access.redhat.com/ubi9/python-312`
- Install system deps: gcc, g++ for faiss-cpu compilation
- Install Python deps from pyproject.toml (use pip install .)
- EXPOSE 8501 (Streamlit default)
- ENTRYPOINT: `streamlit run streamlit_app.py --server.port=8501 --server.address=0.0.0.0`
- USER 1001 for OpenShift compatibility
- HuggingFace cache dir needs to be writable: set HF_HOME to a writable location

## Deployment Considerations
- Deploy as Deployment + Service (Streamlit is a long-running web server)
- Use `large` resource profile: 4Gi request / 8Gi limit RAM, 2 CPU request / 4 CPU limit
- Separate Job manifest for the CLI search test (Scenario 3)
- No GPU tolerations needed
- Set HF_HOME env var to /tmp/hf_cache for writable cache in OpenShift
- imagePullPolicy: Always (image from quay.io)
- No readiness probe initially — Streamlit cold start is long due to index download
