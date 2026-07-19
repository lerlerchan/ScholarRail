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

## Phase 4 status (2026-07-19) — CORE DONE
- PaperSpine V4 (WUBING2023) cloned + `paper-spine` skill installed to `~/.claude/skills/` — it is now ONE orchestration skill with playbooks + gate scripts, hosts incl. Claude Code and Hermes CLI. Argument layer outputs live in `paper_rewriting_output/`: `confirmed_contribution.md` (V4 hard gate), `confirmed_motivation.md`, `rewrite_matrix.md` (= PRD's "section blueprints").
- `~/scholarstack/load_spine.py` — /load-spine handoff: PaperSpine outputs → context JSON; enforces contribution-before-drafting gate.
- `~/scholarstack/draft_section.py` — Stage 4 drafting: LightRAG `only_need_context` retrieval → deepseek-v4-pro draft → **mechanical** post-check that every `[@key]` citation ∈ verified pool; exit 1 on violation (Ralph loop decides re-draft). Convention: `[@key]` citation markers.
- CritiqueBot implemented as Claude Code agent `~/.claude/agents/critiquebot.md` (4 attack modes, JSON verdict block, bounded single-shot). Different model family from generator per §6.1. **API/n8n variant deferred: no ANTHROPIC_API_KEY provisioned on K45VD/n8n yet.**
- Tests: 3 unit + 1 live integration (ingest source under citation-key filename → grounded draft → zero citations outside pool; DeepSeek v4-pro, ~4min). All green.
- Running totals: 12 unit + 5 integration tests green.

## Intake decisions (2026-07-19)
- `mochow13/google-scholar-mcp`: **REJECTED** — SkillSpector static scan (LLM tier off, no key): risk 100/100 CRITICAL. Known-vulnerable @modelcontextprotocol/sdk 1.11, unpinned deps, TM1 parameter-abuse hits. Also moot: Google blocks scraping from this IP regardless of wrapper. Clone kept at `~/scholarstack/candidate-google-scholar-mcp` for reference.
- Scholar Gateway (docs.scholargateway.ai): Wiley-only corpus (8M articles, 2000+ journals), MCP `semantic_search` at connector.scholargateway.ai/mcp, OAuth 2.1 institutional/trial. **Complement, not replacement** for Semantic Scholar (coverage). Claude-side connector exists in harness — needs interactive user auth. Candidate for Stage 1 as secondary retrieval source.
- notebooklm-py upstream identified: github.com/teng-lin/notebooklm-py — cloned to `~/scholarstack/notebooklm-py` (not yet integrated; MarkItDown remains primary ingestion).

## Phase 5 status (2026-07-19) — UTILITIES DONE, PIPELINE BLOCKED
- tectonic 0.16.9 installed to `~/.local/bin` (static musl binary, no sudo). PDF compile verified by test.
- `~/scholarstack/llm.py` — shared DeepSeek chat helper.
- `~/scholarstack/revise.py` — Revision Mode §7.2: draft → draft_vN.md, never overwrites, preserves [@key] citations by prompt contract, audited.
- `~/scholarstack/tldr.py` — TL;DR §7.1: file/URL → MarkItDown → 5 bullets on v4-flash.
- `~/scholarstack/consistency_check.py` — §6.2: docs → contradiction JSON on v4-flash, pass/fail exit code, audited. Live test caught planted n=42 vs n=60 + alpha contradiction.
- Tests now 15 unit + 7 integration, all green.
- ~~BLOCKED: manuscript pipeline stages 5–10~~ → resolved by clean-room build, see below.

## scholarstack-pipeline clean-room build (2026-07-19) — DONE
- **Designed from PRD §5 only** — the CC-BY-NC reference repo's contents were never read this session; clean-room defensible. Author-permission email now nice-to-have, not blocker.
- Local repo `~/scholarstack/scholarstack-pipeline` (git, MIT, commit f195597): `stages.py` (7 draft-stage prompt templates + peer-review sim), `pipeline.py` (~150-line runner: LightRAG-grounded per stage, mechanical [@key]-in-pool gate with bounded re-draft MAX=2 then hard fail, human confirmation gates unless --auto, audit-logged, pandoc→tectonic assembly).
- pandoc 3.10 installed to ~/.local/bin (static, no sudo).
- Tests: 3 unit (incl. full offline run → real PDF) + 1 live single-stage — all green. Totals now 18 unit + 8 integration.
- TODO before public release: extract shared modules from ~/scholarstack into the package (currently sys.path import), pyproject, push to github.com/lerlerchan/scholarstack-pipeline (user creates repo).
- Scopus APIs (dev.elsevier.com): free academic non-commercial tier with API key — strong Stage 1 + verification candidate for SUC FEIT; **Phase 2 paid cloud would need commercial license**. User signup required.

## Publication + chain entry (2026-07-19) — DONE
- `github.com/lerlerchan/scholarstack-pipeline` created via API + pushed (MIT, pyproject, Semgrep CI: p/secrets p/python p/owasp-top-ten).
- `~/scholarstack/scholarstack.py` — single `/scholarstack` entry per PRD §5 standing rule: verify → ingest → Gate 2.5 → spine → pipeline → consistency → PDF. Each gate short-circuits with exit 1; integrity non-skippable. 3 unit tests green (gate short-circuits + happy path).
- Totals: 21 unit + 8 integration green.
- Still user-side: Semantic Scholar key, Scopus key, Scholar Gateway auth, ANTHROPIC_API_KEY for n8n CritiqueBot, n8n workflow wiring, Telegram router (blocked on n8n webhooks existing).
