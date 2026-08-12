# Removing Embeddings / RAG from Agent Zero

## Current implementation map

### Core embedding stack
- `models.py` — `LiteLLMEmbeddingWrapper` and `LocalSentenceTransformerWrapper` (implements langchain `Embeddings`), `get_embedding_model()`, plus `from sentence_transformers import SentenceTransformer`.
- `helpers/vector_db.py` — `VectorDB` wrapper around `langchain_community.vectorstores.FAISS` with `MyFaiss` overrides, `CacheBackedEmbeddings`, cosine relevance scoring, `simpleeval` metadata filters, `format_docs_plain`, plus `import faiss` and the ARM/python3.12 monkey patch import.
- `helpers/faiss_monkey_patch.py` — shims `numpy.distutils.cpuinfo` so FAISS loads on python 3.12/ARM.
- `requirements.txt` — `faiss-cpu==1.11.0`, `sentence-transformers==3.0.1`, `langchain-core==0.3.49`, `langchain-community==0.3.19`; torch is image-installed in the Docker build (CPU `torch==2.4.0`/`torchvision==0.19.0`).

### Consumers
- `plugins/_memory/helpers/memory.py` — persistent FAISS memory index (duplicated `MyFaiss`), `LocalFileStore` embedding cache under `tmp/memory/embeddings`, per-subdir indexes under `usr/memory`, knowledge preload/import, consolidation.
- `plugins/_document_query/helpers/document_query.py` — `DocumentQueryStore` RAG over parsed documents: `RecursiveCharacterTextSplitter` chunking, `VectorDB.insert_documents`, metadata search, similarity-threshold search (default 0.5).
- `tool/document_query.py` compatibility shim and `plugins/_document_query/tools/document_query.py`.
- `plugins/_model_config/helpers/model_config.py` — `get_embedding_model_config()`, `get_embedding_model_config_object()`, embedding model settings; `api/model_presets.py`, `api/model_override.py`, connector `model_switcher.py` all manipulate embedding config.
- `preload.py` — `preload_embedding()` eagerly touches the embedding model when provider is `huggingface`.
- `helpers/migration.py` — legacy `memory/embeddings` → `tmp/memory/embeddings` migration.
- `conf/model_providers.yaml` — `embedding:` provider section (huggingface, google, lm_studio, llama_cpp, mistral, nvidia_nim, ollama, omlx, openai, ...).
- Tests touching embeddings: `test_model_config_api_keys.py`, `test_document_query_plugin.py`, `test_model_search.py`, `test_onboarding_static.py`, `test_model_config_project_presets.py`.

## Why removal is worth doing
- Dropping `faiss-cpu`, `sentence-transformers`, and the torch embedding install removes a large amount of image weight and the ARM python3.12 hack entirely.
- The embedding model is another API-key/config surface that must be configured, preloaded, and kept in sync across model switcher/presets/overrides.
- Persistent FAISS indexes and the embedding cache are stateful artifacts that can go stale when the embedding model changes (there is already a re-index path for that).

## Removal options

### Option A — Slim the image only (smallest change, leaves dead code)
- Remove `faiss-cpu`, `sentence-transformers`, and the torch CPU install from the build/requirements.
- Guard imports so `_memory`/`_document_query` fail soft when disabled.
- Leave framework code and UI surfaces in place.
- Pro: one-line-ish change. Con: dead code, dangling config/UI, and hidden failures.

### Option B — Clean end-state (recommended if memory/document-Q&A features are not needed)
1. Decide feature fate: drop `_memory` + `_document_query` entirely, or keep them with a non-vector fallback (Option C).
2. `requirements.txt`: remove `faiss-cpu`, `sentence-transformers`; trim `langchain-core`/`langchain-community` if no other consumers remain.
3. `DockerfileLocal`: remove the CPU torch/torchvision install step unless TTS/STT plugins still need torch (they currently do not import it directly).
4. `models.py`: delete the two embedding wrapper classes and `get_embedding_model()`; remove the `sentence_transformers` import.
5. Delete `helpers/vector_db.py` + `.dox.md` and `helpers/faiss_monkey_patch.py` + `.dox.md` (file-level DOX cleanup required by `helpers/AGENTS.md`).
6. `preload.py`: remove the `preload_embedding()` task.
7. `plugins/_memory`: remove FAISS index/vector persistence, or replace with simple keyword/SQLite store; update tools, prompts, plugin `AGENTS.md` and `README`.
8. `plugins/_document_query`: keep parsing if valuable but swap retrieval to text/BM25-style search, or remove with the plugin; update plugin DOX/README.
9. `plugins/_model_config` + `_a0_connector`: remove embedding config, presets, override surfaces, and mode-preset embedding entries; update their `AGENTS.md`.
10. `conf/model_providers.yaml`: remove the `embedding:` section.
11. `helpers/migration.py`: remove or no-op the `memory/embeddings` migration.
12. Update/remove embedding tests; keep DOX coverage check happy (`api/*.py.dox.md`, `tools/*.py.dox.md`, `helpers/*.py.dox.md`).
13. Verify: targeted pytest, startup smoke, and a clean Docker build.

### Option C — Keep features, drop vectors (medium effort, preserves UX)
- Replace vector memory retrieval with SQLite FTS or plain keyword matching in `_memory`.
- Replace document Q&A retrieval with BM25/trigram matching in `_document_query`.
- Same cleanup list as B for models/config/preload, but keep the plugin surfaces with the new backend.

## Recommended sequence (brainstorm)
1. Audit all `get_embedding_model*` callers and every embedding UI/API reference in the same commit (they break if you remove the backend first).
2. Remove image/requirements weight first so build time shrinks immediately.
3. Remove plugin consumers second, then shared helpers, then config/UI, then tests/DOX.
4. Ship a migration note for users with existing indexes (they become obsolete: delete `usr/memory` indexes + `tmp/memory/embeddings`, or convert if keeping features via Option C).

## Risks / things to keep in mind
- Any remaining `models.get_embedding_model()` caller will crash after backend removal — the whole surface must move together.
- `_memory` and `_document_query` are optional plugins (`always_enabled` not true for `_office`; check both plugin manifests), but their tools are referenced in default prompts and tests, so remove/repair those references in the same change.
- Sentence-transformers pulls torch; removing only the embedding stack without touching kokoro/whisper assumes those plugins do not import torch directly (verified: no `import torch` in `_kokoro_tts`/`_whisper_stt` code).
