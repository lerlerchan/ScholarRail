# ScholarStack Implementation Plan (approved 2026-07-19)

Derived from `docs/scholarstack-prd-v2.0.md`. Each phase: build → unit → integration → functionality → gate.

## Phase 1 — Foundation & Retrieval (Stages 0–1)
LightRAG server (file storage first; PostgreSQL when available — no sudo on K45VD), Google Scholar MCP, Semantic Scholar MCP, `/search-scholar` + `/export-bib`.
- Unit: bib export format, MCP response parsing, LightRAG insert payload.
- Integration: real MCP query → session.bib; LightRAG 3-doc round-trip.
- Functionality: real-topic shortlist; bge-m3 Manglish test (Open Issue #1).
- Also: grep configs for deprecated DeepSeek aliases (deadline 2026-07-24); Semgrep CI from day one; pytest fixtures with recorded API responses.

## Phase 2 — Citation Verification Gate (Stage 1.25)
CitationVerifier n8n node (CrossRef + OpenAlex, no LLM) + Audit Log JSONL.
- Unit: resolved / not_found / metadata_mismatch on fixtures.
- Integration: Phase-1 session.bib → verified pool + rejection log.
- Functionality: 3 planted fake citations → all rejected, correct reason codes. Permanent regression test.

## Phase 3 — Ingestion & Graph (Stages 1.5–2 + Gate 2.5)
notebooklm-py + MarkItDown, `nlm_to_lightrag.py` bridge, `/load-refs`, integrity-agent Ralph loop.
- Unit: bridge script; subprocess-injection tests (PRD §8).
- Integration: PDF → MarkItDown → LightRAG → attributed query results.
- Functionality: Gate 2.5 exit-condition tests (gap → FAIL+list; complete → PASS).

## Phase 4 — Argument & Drafting (Stages 3–4 + Gate 4.5)
PaperSpine `/load-spine`, scholarstack-pipeline stages 1–3 (deepseek-v4-pro), CritiqueBot Logic+Evidence.
- Unit: blueprint parsing, prompt assembly, CritiqueBot verdict schema.
- Integration: draft with zero citations outside verified pool.
- Functionality: full Stage 0→4 run; CritiqueBot bounded-loop termination.
- Must build against MIT fork `scholarstack-pipeline`, never CC-BY-NC original.

## Phase 5 — Full Manuscript & Post-Pipeline (Stages 5–10)
Pipeline 4–10, remaining CritiqueBot modes, Consistency Checker, tectonic PDF, Revision Mode, TL;DR.
- Unit: version naming (never overwrite), consistency diff, LaTeX escaping.
- Integration: end-to-end `/scholarstack` → PDF; revision → v2 + auto checks.
- Functionality: one real paper; verify cost claim; Semgrep green.

## Phase 6 — Front Door: thin Telegram router (Hermes DEFERRED)
Router bot → n8n webhooks with JSON schemas. That contract doubles as future Hermes tool interface — adopting Hermes later = swap router, zero pipeline rework.
- Hermes deferred because PRD §13.1 conditions unmet (skill-loop audit conflict, multi-tenancy, sandboxing) and v1 use case is pipeline-trigger → thin router wins.
- Re-evaluate after 1 semester if open-ended questions dominate router traffic.

## Order constraints
- Phase 2 blocks Phase 4 (writers only see verified pool).
- License blocker (§11) parallel to all: fork replacement + author email.

## Risks
- HIGH: notebooklm-py unofficial API breakage → MarkItDown fallback first-class.
- HIGH: license blocker → send email now.
- MEDIUM: bge-m3 Manglish; Scholar MCP rate limits (Semantic-only fallback).
- LOW: DeepSeek alias deprecation 2026-07-24.

## Environment facts (probed 2026-07-19)
- Host: lerler-K45VD, Python 3.12.3, no sudo.
- PostgreSQL not installed → LightRAG file-based storage initially.
- ~/scholarstack created fresh this session.

## Phase 1 status (2026-07-19)
DONE:
- `~/scholarstack` skeleton, `.venv` with lightrag-hku[api] + markitdown[all] + pytest.
- LightRAG server running on :9621 — deepseek-v4-flash LLM, Ollama `bge-m3` embeddings (note: Ollama model name is `bge-m3`, NOT `BAAI/bge-m3`), file storage. `.env` at `~/scholarstack/.env` (chmod 600).
- Integration tests green: `~/scholarstack/tests/test_lightrag_roundtrip.py` (health + insert→query round-trip, ~69s, requires `file_source` field and unique source name per run).
- Manglish retrieval probe passed (single sample — Open Issue #1 preliminary OK).
- MCPs registered user-scope: `semantic-scholar` (uvx), `google-scholar` (local venv `.venv-scholar-mcp`).
- No legacy DeepSeek aliases found in claude-scholar clone.
- claude-scholar + Google-Scholar-MCP-Server cloned to ~/scholarstack.

BLOCKED / USER ACTION:
- Google Scholar scraping blocked from this IP (`MaxTriesExceededException`) — Open Issue #2 confirmed; Semantic Scholar is primary path.
- Semantic Scholar public API 429s without key → **apply for free API key** (semanticscholar.org/product/api#api-key-form).
- GitHub forks missing: `scholarstack-pipeline` (release blocker), `notebooklm-py` — create before Phase 4. MarkItDown covers ingestion meanwhile.
- Upstreams provided 2026-07-19: PaperSpine = github.com/WUBING2023/PaperSpine (paper: arxiv.org/abs/2604.05018); academic-research-skills = github.com/imbad0202/academic-research-skills (CC-BY-NC — reference only, never wire into pipeline; MIT fork `scholarstack-pipeline` must replace it pre-release); OpenDraft = github.com/federicodeponte/opendraft (rejected 19-agent architecture, comparison reference only); LightRAG = github.com/hkuds/lightrag (installed via pip `lightrag-hku[api]`).
- License email to academic-research-skills author still unsent (§11).

NEXT: `/export-bib` flow + bib-format unit tests (needs Semantic Scholar key), Phase 3 ingestion bridge.

## Phase 2 status (2026-07-19) — CORE DONE
- `~/scholarstack/citation_verifier.py` — mechanical gate, no LLM. Resolver chain: CrossRef → DataCite → OpenAlex (title-search fallback when no DOI). Exit code 0/1 for n8n gate. Audit Log JSONL append (`~/scholarstack/logs/audit.jsonl`).
- **Design finding:** arXiv DOIs (10.48550/*) are DataCite-registered — CrossRef AND OpenAlex DOI lookup both miss them. PRD §5 "CrossRef + OpenAlex" insufficient; DataCite added to chain. PRD should be amended.
- Tests: 7 unit (monkeypatched, 0.07s, zero network) + 1 live functionality test — 3 planted fakes (fabricated DOI, fabricated title, stolen DOI with wrong title) all rejected with correct reason codes; real paper verified. 8/8 green. Permanent regression suite per plan.
- Tolerances: title fuzzy-match ≥0.80 (difflib), year ±1 (online-first vs print).
- Remaining for Phase 2: n8n node wrapping (webhook → this script) — deferred to n8n wiring pass; K45VD-side logic complete.

## Phase 3 status (2026-07-19) — CORE DONE
- `~/scholarstack/ingest.py` — any file → MarkItDown (library call, no subprocess/shell → no injection surface per §8) → LightRAG `/documents/text` with `file_source` = basename. Exit-code CLI.
- `~/scholarstack/integrity_check.py` — Gate 2.5, mechanical: verified-pool keys vs LightRAG processed doc stems. Bidirectional: `missing_sources` (cited-not-ingested → Ralph loop gap list) + `unverified_sources` (ingested-never-verified → hallucination guard). Exit 0/1, audit-logged. Convention: source file for key K named `K.*`.
- PRD amended to v2.0.1 (DataCite in resolver chain, §5 + version history).
- Ops finding: Ollama bge-m3 cold-load exceeds LightRAG's 60s embed worker timeout → `EMBEDDING_TIMEOUT=180` in `.env`. LightRAG dedups identical content across filenames (409-like failure) — test content must be unique per run.
- All tests green: 9 unit (offline) + 4 integration (live) across 3 suites.
- Deferred: notebooklm-py path (fork repo missing; MarkItDown is the working ingestion path).
