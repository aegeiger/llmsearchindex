# Executive Summary: LLMSearchIndex PoC

**Project:** [LLMSearchIndex](https://github.com/zakerytclarke/llmsearchindex)
**Date:** 2026-05-05
**Result:** PASS (3/3 tests passed)

## What

LLMSearchIndex is a Python library providing fully local, internet-scale web search across 203 million web pages (Wikipedia + FineWeb). It uses FAISS binary indexing with Sentence Transformers embeddings for sub-10-second search latency without external API calls.

## What We Did

Containerized the Streamlit web UI and CLI search tool on a UBI9 base image with CPU-only PyTorch. Deployed to OpenShift as a Deployment + Service, with a separate Job for CLI validation. Pushed image to quay.io/aicatalyst/llmsearchindex.

## Results

| Test | Result |
|------|--------|
| Web UI responds on port 8501 | PASS |
| Streamlit health endpoint | PASS |
| CLI search returns relevant results | PASS |

Search query "who invented sliced bread" returned 3 relevant results in 7.8 seconds.

## Key Metrics

- **Cold start:** ~92s (index download from HuggingFace)
- **Search latency:** ~8s per query
- **Memory:** 4-8 GB
- **GPU:** Not required

## Recommendation

Good candidate for RHOAI demos (score: 72/100). The visual Streamlit UI searching 203M pages locally is compelling. For production, add persistent storage for index caching and a HuggingFace token for authenticated downloads.

## Links

- Fork: https://github.com/aegeiger/llmsearchindex
- Image: https://quay.io/repository/aicatalyst/llmsearchindex
