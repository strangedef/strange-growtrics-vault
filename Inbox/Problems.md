- Docs
- Code formatting
- Claude can read .env
- Plugins to save token

### NO Document processing 

Almost — "no processing" means no understanding pipeline. What happens vs doesn't:

Does NOT happen:
- No chunking, no embeddings, no vector index, no RAG
- No server-side PDF/DOCX parsing — binary stored verbatim, text=""
- No summarization/synthesis jobs (old SOURCE_INDEX / KNOWLEDGE_SYNTHESIS pipeline deleted)

DOES happen (thin, mechanical):
1. Upload validation — extension allowlist (pdf/docx/md/markdown/txt), 10MB cap, MIME derived from extension, never trusted from client.
2. Text decode — md/txt bytes ARE the doc, decoded into .text at upload. Only "parsing" done.
3. Read-time composition — run_execution's docs.py merges sources by origin priority (app_knowledge > uploaded > imported) into prompt brief + per-scan corpus. Deterministic, no LLM.
4. Lexical retrieval on device plane — project_knowledge.json artifact, regex token match, ~900-char snippets via search_project_knowledge.
5. PDF/DOCX "processing" deferred to LLM — original file attached as multimodal part at generation time; OpenRouter side parses it. So doc IS understood — just at use time by model, not at upload time by us.

Design bet: docs small enough (10MB cap) that raw-at-generation beats index-maintain-sync complexity. Trade: every generation pays full doc cost in tokens; no semantic search. Fine at current scale.