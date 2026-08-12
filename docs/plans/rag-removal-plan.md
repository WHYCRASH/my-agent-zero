# Removing the Embeddings / RAG / Memory System — Step-by-Step Plan

Status: draft awaiting approval
Branch: ubuntu (fork WHYCRASH/my-agent-zero)
Owner: dawtist-Zero
Supersedes: earlier brainstorm content of this file

## 1. Goal and non-goals

Goal:
- Remove the vector embedding, FAISS, persistent-memory, and document-RAG subsystems from Agent Zero cleanly and DOX-compliantly.
- Image/runtime no longer needs faiss-cpu, sentence-transformers, langchain vector stores, or the CPU torch install.
- WebUI/API no longer exposes embedding-model configuration; preset/override surfaces stop carrying embedding slots.

Non-goals (out of scope):
- Do not modify README.md (user constraint).
- Do not touch webui/js/transformers@3.0.2.js — vendored onnxruntime/transformers.js used by speech/vision features, not by FAISS RAG.
- Do not implement Xpra or nginx→Caddy (separate tracked TODOs).
- Do not remove the generic search helper (helpers/searxng.py, tools/search_engine.py) unless the consumer audit shows they are only used by the removed knowledge tool (decision D6).

## 2. Approval gates (required before implementation)

- G1 — Scope: remove both `_memory` and `_document_query` plugins entirely (recommended, Option B from the earlier brainstorm). Alternative is to keep `_document_query` on a non-vector keyword/BM25 backend (Option C, higher effort).
- G2 — agent.py: root AGENTS.md requires asking before modifying `agent.py`. Recommended: delete the `get_embedding_model()` extensible hook and the `_model_config` start extension that fills it. If not approved: leave the dead hook (no callers remain) and flag it.
- G3 — initialize.py: root AGENTS.md requires asking before modifying `initialize.py`. `initialize.py` references `knowledge_subdirs` and knowledge preload. Recommended: remove knowledge resolution/preload from initialize. If not approved: leave knowledge loading inert and flag residual.
- G4 — Deletion manifest: confirm the tracked deletions in §6.
- G5 — knowledge/: keep `knowledge/` markdown as inert source docs (recommended; cheap, keeps root child index and self-knowledge) or delete it.
- G6 — .github/workflows/docker-publish.yml: leave unchanged unless the canonical image target moves; docker/AGENTS.md already flags sync requirements.

## 3. Verified current implementation map (2026-08-12, branch ubuntu)

Core backend:
- models.py — `LiteLLMEmbeddingWrapper`, `LocalSentenceTransformerWrapper`, `_get_litellm_embedding`, `get_embedding_model()`; imports `from litellm import embedding`, `from langchain.embeddings.base import Embeddings`, `from sentence_transformers import SentenceTransformer`; `EMBEDDING` model-type member.
- helpers/providers.py — `ModelType = Literal["chat", "embedding"]`; `get_providers()` type param.
- helpers/settings.py — `embedding_providers` projection and provider aggregation.
- preload.py — `preload_embedding()` eagerly touches the embedding model when provider is huggingface.
- helpers/migration.py — `_migrate_memory()`: legacy `memory/embeddings` → `tmp/memory/embeddings`, other memory to `usr/memory`.
- conf/model_providers.yaml — `embedding:` provider section (~line 262).

Vector store:
- helpers/vector_db.py + vector_db.py.dox.md — `VectorDB` FAISS wrapper, `MyFaiss`, `CacheBackedEmbeddings`, simpleeval metadata filters.
- helpers/faiss_monkey_patch.py + .dox.md — python3.12/ARM shim for FAISS.

Consumers:
- plugins/_memory — persistent FAISS memory, knowledge import, dashboard, tools (memory_save/load/delete/forget/behaviour_adjustment), prompts, message-loop/monologue/system-prompt extensions, `embedding_model_changed` reload extension, webui dashboard/config/sidebar entries.
- plugins/_document_query — indexing/parsing/Q&A over documents; `VectorDB` store; `RecursiveCharacterTextSplitter`; langchain imports; parsers (pdf/text/html/image/unstructured/liteparse); tools/document_query.py; compat shim `tools/document_query.py` + .dox.md; prompts/skills/webui/hooks.
- tools/knowledge_tool._py — imports `Memory`, `DEFAULT_MEMORY_THRESHOLD`, `DocumentQueryHelper`; uses vector similarity search, memory search, searxng document QA; consumed via `fw.knowledge_tool.response.md`.
- initialize.py — `knowledge_subdirs` resolution (G3).

Config/API/UI:
- plugins/_model_config — helpers/model_config.py embedding config/build/providers/snapshot; api/model_config_get|set, model_override, model_presets, api_keys, model_search; extension `agent/Agent/get_embedding_model/start/_10_model_config.py`; `embedding_model_changed` event; mode_presets_fallback.yaml + provider_metadata.yaml `embedding:` sections; webui main.html, model-field.html, config.html, model-config-store.js, switcher-mixin.js.
- plugins/_a0_connector — api/v1/model_presets.py + model_switcher.py embedding signatures/notifications; connector AGENTS.md contract line mentions embedding-model reporting.
- webui/components/settings/settings-store.js — `embedding_providers` merged into provider lists.

Tests:
- tests/test_memory_cleanup.py, test_memory_quality.py, test_document_query_plugin.py, test_document_query_fallback.py — direct RAG/memory.
- tests/test_model_config_api_keys.py (embedding model cases), test_model_search.py, test_onboarding_static.py (`chat`/`embedding` loop), test_model_config_project_presets.py, test_model_config_ui.py, test_time_travel.py (`.a0proj/memory/index.faiss` artifact).

Dependencies/image:
- requirements.txt — faiss-cpu==1.11.0, sentence-transformers==3.0.1, langchain-core==0.3.49, langchain-community==0.3.19, langchain-unstructured==0.1.6 (verify simpleeval and unstructured consumers before trimming).
- DockerfileLocal lines 57-58 — CPU torch==2.4.0/torchvision==0.19.0 into /opt/venv-a0; docker/base + docker/run install requirements into venvs; /ins scripts reference requirements.
- docker/AGENTS.md — two-runtime contract currently says /opt/venv is Python 3.13 but the image has 3.12.3 in both venvs (pre-existing DOX drift; fix during the Phase 2/8 image DOX pass).

## 4. DOX workflow requirements

- Before each phase: read root AGENTS.md and the nearest AGENTS.md for every path touched (helpers, tools, plugins/_memory|_document_query|_model_config|_a0_connector, conf, prompts, docs, docker + base/run, tests, extensions, webui, knowledge).
- File-level DOX: helpers/, tools/, api/ direct `*.py` modules must have matching `*.py.dox.md`; when deleting vector_db.py, faiss_monkey_patch.py, or tools/document_query.py, delete the companion `.dox.md` in the same change; never leave stale file-level DOX.
- Update the nearest owning AGENTS.md in the same change as contract/structure changes; refresh child DOX index tables (plugins/AGENTS.md rows for `_memory` and `_document_query`).
- Closeout: re-check changed paths, run `git diff --check`, targeted tests, and a grep sweep.

## 5. Phases and steps

### Phase 0 — Approvals
- Run the plan by the user; obtain G1-G6.
- Record the exact file list (this doc) as the approved deletion manifest.

### Phase 1 — Audit and freeze
- Grep sweep: `embedding|Embeddings|get_embedding_model|VectorDB|vector_db|faiss|sentence_transformers|langchain|memory_(save|load|delete|forget)|document_query|knowledge_tool` across *.py, *_py, yaml, html, js, md.
- Confirm callers of `get_embedding_model*`, `VectorDB`, `Memory`, `DocumentQueryHelper`, the get_embedding_model extension point, and `embedding_model_changed`.
- Confirm simpleeval/langchain-unstructured have no consumers outside vector_db/document_query.
- Freeze the deletion manifest (§6). No behavior change in this phase.

### Phase 2 — Image and dependency weight
- DockerfileLocal: remove the `/opt/venv-a0 pip install torch==2.4.0 torchvision==0.19.0 ... cpu` step.
- requirements.txt: remove faiss-cpu, sentence-transformers, langchain-core, langchain-community, langchain-unstructured (after Phase 1 confirms consumers); remove simpleeval only if no other consumers remain.
- Adjust any docker /ins script that references torch/faiss/removed requirements.
- DOX touch: read docker/AGENTS.md, docker/base/AGENTS.md, docker/run/AGENTS.md before editing; update docker/AGENTS.md in the same change (remove torch/embedding claims; optionally fix the Python-version drift line). Run `git diff --check`.
- Verification: grep shows no faiss/torch install in Dockerfiles. Image rebuild on the laptop is optional and user-driven.

### Phase 3 — Core backend removal
- models.py: delete `LiteLLMEmbeddingWrapper`, `LocalSentenceTransformerWrapper`, `_get_litellm_embedding`, `get_embedding_model`; remove `from litellm import embedding`, `from langchain.embeddings.base import Embeddings`, `from sentence_transformers import SentenceTransformer`; remove the `EMBEDDING` model-type member and any `ModelType` union references.
- helpers/providers.py + .dox.md: make `ModelType` chat-only; remove embedding branches in `get_providers`.
- helpers/settings.py + .dox.md: drop `embedding_providers` field/projections/aggregation.
- helpers/migration.py + .dox.md: remove `_migrate_memory()` and its call; keep the rest of migration.
- preload.py: remove `preload_embedding()` and its task from the startup list.
- conf/model_providers.yaml: remove the `embedding:` section.
- DOX touch: read helpers/AGENTS.md and the three helper .dox.md files before editing; update them in the same change; read conf/AGENTS.md; run model/provider tests.
- Verification: `/opt/venv-a0/bin/python -c "import models, preload, helpers.settings, helpers.providers"`; grep shows no `get_embedding_model` outside agent.py (see G2).

### Phase 4 — Consumer removal (plugins/tools/helpers)
- plugins: `git rm -r plugins/_memory plugins/_document_query` (G1). Their prompts/tools/extension surfaces disappear with the dirs.
- tools: `git rm tools/document_query.py tools/document_query.py.dox.md` (compat shim + file DOX). `git rm tools/knowledge_tool._py`; remove `prompts/fw.knowledge_tool.response.md` if no other users.
- helpers: `git rm helpers/vector_db.py helpers/vector_db.py.dox.md helpers/faiss_monkey_patch.py helpers/faiss_monkey_patch.py.dox.md`.
- Update plugins/AGENTS.md child DOX index (remove `_memory` and `_document_query` rows) and any scope text in the same change.
- Update tools/AGENTS.md if tool-coverage expectations change; run the tools file-DOX coverage check.
- Verify helpers file-DOX coverage loop still passes (every helpers/*.py has .dox.md).
- D6: if searxng helper/search_engine are only used by knowledge_tool, add them to the deletion manifest (or keep as follow-up).

### Phase 5 — Model-config/connector/WebUI embedding surfaces
- plugins/_model_config/helpers/model_config.py: remove embedding mapping/entry, `get_embedding_model_config*`, `build_embedding_model`, `get_embedding_providers`, embedding snapshot, and `embedding_model_changed` notification helpers.
- plugins/_model_config/api: model_config_get (embedding_providers payload), model_config_set (embedding section + notify), model_override (embedding notify), model_presets (`_embedding_signature`/notify), api_keys (embedding provider list), model_search (embedding search scope).
- Delete plugins/_model_config/extensions/python/_functions/agent/Agent/get_embedding_model/start/_10_model_config.py.
- Remove the `embedding_model_changed` extension point: the `_memory` reload extension is gone in Phase 4; remove all `call_extensions_async(..., "embedding_model_changed", ...)` callers and any `_model_config` handler of the event.
- mode_presets_fallback.yaml + provider_metadata.yaml: remove `embedding:` sections.
- webui: plugins/_model_config/webui/main.html, model-field.html (embedding branch), config.html, model-config-store.js (embedding keys/providers/rows), switcher-mixin.js (embedding slot); webui/components/settings/settings-store.js (`embedding_providers` merge).
- plugins/_a0_connector: api/v1/model_presets.py + model_switcher.py — remove embedding signatures/notifications/override slot; update the connector AGENTS.md contract line to main/utility only.
- DOX touch: read plugins/_model_config/AGENTS.md, plugins/_a0_connector/AGENTS.md, webui/AGENTS.md before editing; update in the same change; refresh plugins/AGENTS.md if scope text changes.
- Verification: pytest model_config/onboarding subset; grep no `embedding` in plugins/_model_config, plugins/_a0_connector, or webui (excluding vendored transformers.js).

### Phase 6 — Prompts, docs, knowledge
- prompts/agent.system.main.tips.md: remove `document_query` guidance lines.
- prompts/agent.system.tool.parallel.md: remove the `document_query` parallel note.
- prompts/AGENTS.md: update the document/OCR routing clause; if no document tool remains, replace with plain wording in the same change.
- Remove `prompts/fw.memories_not_found.md`, `prompts/fw.memories_deleted.md`, `prompts/fw.knowledge_tool.response.md` if unreferenced (grep).
- knowledge/AGENTS.md (G5): if knowledge/ is kept, soften indexing/recall wording to source documentation; if removed, delete `knowledge/` and update root AGENTS.md child index.
- docs/: this plan is the migration note. Add a short "RAG removed" note in docs/plans or docs/developer; update docs/AGENTS.md only if structure changes.
- Do not modify README.md.

### Phase 7 — Tests
- Remove: tests/test_memory_cleanup.py, tests/test_memory_quality.py, tests/test_document_query_plugin.py, tests/test_document_query_fallback.py.
- Update: tests/test_model_config_api_keys.py (drop embedding get_embedding_model cases), tests/test_model_search.py (remove embedding section), tests/test_onboarding_static.py (drop `embedding` in model_type loop), tests/test_model_config_project_presets.py (embedding preset slots), tests/test_model_config_ui.py (embedding rows), tests/test_time_travel.py (replace `.a0proj/memory/index.faiss` fixture with a non-vector artifact).
- Add if useful: a startup import regression test asserting `models` imports without sentence-transformers/faiss.
- Verification: targeted `pytest` with the framework runtime.

### Phase 8 — DOX closeout and consistency sweep
- Re-read changed paths against the DOX chain; update nearest owners and child indexes (root, plugins, tools, helpers, conf, prompts, knowledge, docker, docs, extensions, webui).
- Grep sweep expecting zero (in tracked code): `embedding|Embeddings|get_embedding_model|sentence_transformers|faiss|VectorDB|vector_db|langchain|document_query|memory_(save|load|delete|forget|reload)|knowledge_tool|embedding_model_changed|embedding_providers`.
- Ensure no stale .dox.md for deleted modules; run coverage loops for helpers/, tools/, api/.
- Run `git diff --check`.

### Phase 9 — Verification
- Runtime import smoke (framework venv): import models, preload, helpers.settings, helpers.providers, run_ui; confirm no faiss/sentence-transformers import attempts.
- Target pytest: pytest tests/test_model_config_api_keys.py tests/test_onboarding_static.py tests/test_model_search.py tests/test_time_travel.py (or the full suite if cheap).
- Startup smoke: `python run_ui.py` (or in the podman container) — WebUI loads, model settings show main/utility only.
- Optional but recommended: podman rebuild `agent-zero:ubuntu-24.04` on the laptop; confirm image size drop and clean startup; log results.
- Report verified vs assumed in the final closeout.

### Phase 10 — Commit and push
- `git add -A`; commit message: `feat(rag): remove embeddings, FAISS, persistent memory, and document RAG` (or split into per-phase commits if preferred).
- Push to `origin ubuntu`.
- Update this plan's status to "implemented"; store one durable memory fact about the completed removal (not task history).

## 6. Deletion manifest (G4)

| Path | Action |
| --- | --- |
| helpers/vector_db.py, helpers/vector_db.py.dox.md | delete |
| helpers/faiss_monkey_patch.py, helpers/faiss_monkey_patch.py.dox.md | delete |
| plugins/_memory/ (whole tree, incl. AGENTS.md + README) | delete |
| plugins/_document_query/ (whole tree) | delete |
| tools/document_query.py, tools/document_query.py.dox.md | delete |
| tools/knowledge_tool._py | decision D6; likely delete |
| helpers/searxng.py + .dox.md, tools/search_engine.py + .dox.md | decision D6; only if no remaining consumers |
| prompts/fw.memories_not_found.md, fw.memories_deleted.md, fw.knowledge_tool.response.md | delete if unreferenced |
| tests/test_memory_cleanup.py, test_memory_quality.py, test_document_query_plugin.py, test_document_query_fallback.py | delete |
| knowledge/ | only if G5 = remove |
| requirements.txt lines (faiss/sentence-transformers/langchain, maybe simpleeval) | edit (remove lines) |
| DockerfileLocal torch/torchvision step | edit (remove step) |
| conf/model_providers.yaml `embedding:` section | edit (remove) |
| plugins/_model_config mode_presets_fallback.yaml + provider_metadata.yaml `embedding:` sections | edit (remove) |

## 7. Post-removal user cleanup (runtime, not tracked)
- Delete old vector artifacts if present: `usr/memory/`, `tmp/memory/embeddings/`, `.a0proj/memory/`, and legacy `memory/`.
- Remove embedding/plugin settings from `usr/plugins/_model_config/presets.yaml` and user plugin configs; chats referencing removed presets fall back to Default.
- Restart WebUI/container after removal.

## 8. Follow-ups (out of scope here)
- Xpra desktop stack and nginx→Caddy migration (tracked in docs/plans/ubuntu-24.04-migration.md).
- docker/AGENTS.md Python-version drift (Python 3.13 claim vs actual 3.12.3) — fix opportunistically during Phase 2/8.
- .github/workflows/docker-publish.yml sync if the canonical image target changes.
