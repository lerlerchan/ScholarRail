# ScholarStack — Product Requirements Document

**Version:** v2.0-Draft (consolidates v1.0 – v1.6 addendum)
**Product:** ScholarStack — Postgraduate Research Pipeline
**Owner:** Ts. Dr. Chan Ler-Kuan (LerLer Chan) — Aliran Nuvesta Sdn Bhd / SUC FEIT
**Date:** 2026-07-19
**Status:** Draft
**License:** AGPL-3.0 (project) — see §11 for component license notes
**Canonical file:** `~/projects/scholarstack/PRD.md` on K45VD

---

## Version History

| Version | Date | Summary |
|---|---|---|
| v1.0 | 2026-06-10 | Initial 5-tool pipeline PRD (claude-scholar, notebooklm-py, LightRAG, PaperSpine, academic-research-skills) |
| v1.1–v1.3 | Jun 2026 | DeepSeek V4 as primary LLM; MarkItDown added; tool rejections (TokenTamer, LLM-Wiki-for-n8n); AGPL-3.0 + phased release decisions |
| v1.4 | 2026-06-18 | Literature Discovery Layer: Google Scholar MCP + Semantic Scholar MCP; Semgrep CI; OWASP threat model; pytest strategy |
| v1.5 | 2026-06-18 | Version rename of v1.4 content; 7-component architecture confirmed |
| v1.6 | 2026-06-20 | Quality hardening (Light-skills analysis): CritiqueBot adversarial reviewer, Cross-Document Consistency Checker, Research Ethics Hard Stop, Verification Audit Log |
| v1.6 addendum | 2026-07-15 | OpenDraft analysis: Citation Verification Gate, Expose Mode, TL;DR Tool, Revision Mode + versioning; 19-agent architecture rejected |
| **v2.0** | **2026-07-19** | **Full consolidation of all above into one document; addendum features promoted to core; Hermes Agent front-door added as Under Evaluation (§13)** |
| v2.0.1 | 2026-07-19 | Citation Verification Gate: DataCite added to resolver chain (empirical finding during Phase 2 implementation — arXiv DOIs missed by CrossRef+OpenAlex) |
| v2.0.2 | 2026-07-20 | Product renamed **ScholarStack → ScholarRail** (name-collision review; ScholarRail is globally unique on GitHub, free on PyPI). Repos: github.com/lerlerchan/ScholarRail + scholarrail-pipeline. "ScholarStack" below is the historical name — read as ScholarRail. |

---

## 1. Problem Statement

Postgraduate students in Malaysia (and SEA broadly) struggle with three compounding problems:

1. **Citation hallucination** — AI tools invent references; students submit papers with fabricated sources.
2. **Fragmented tooling** — literature search, note-taking, argument structuring, and manuscript writing live in separate apps with no shared memory.
3. **Writing integrity gaps** — no systematic check between draft stages; supervisors catch problems only at final submission.

ScholarStack addresses all three by wiring existing open-source tools into a single Claude Code harness, orchestrated via n8n, with mechanical integrity verification loops at every stage gate.

---

## 2. Product Vision

An integrated open-source research pipeline that takes a postgraduate student from **raw topic** to **Scopus-ready manuscript** with grounded citations, integrity verification, and persistent project memory throughout.

> Human decision-making stays at the center. The pipeline accelerates the grunt work.

**Primary users:** Postgraduate students (Master's / PhD) at SUC FEIT and institutions served by Aliran Nuvesta
**Secondary users:** Research supervisors; lecturers running research methodology courses
**Platform:** K45VD local deployment (Ubuntu MATE 24.04); Claude Code harness; n8n orchestration; Ollama local fallback

**Positioning note (from OpenDraft comparison):** ScholarStack is a *research support layer* — support + verify, not autonomously write full theses. It is a pipeline/harness, not a skill pack; installable skill packs (e.g. Academic Research Skills) are candidate *components*, not competitors.

---

## 3. Component Inventory

| # | Component | Role | License |
|---|---|---|---|
| 1 | **Google Scholar MCP** (JackKuo666) | Broad live paper search — keyword & advanced, author lookup, citation counts | MIT |
| 2 | **Semantic Scholar MCP** (akapet00) | Deep retrieval — citation networks, BibTeX export, 13 API tools, session-tracked, cached | MIT |
| 3 | **claude-scholar** | Harness orchestrator, session memory, Obsidian KB, Zotero MCP | MIT |
| 4 | **notebooklm-py** | Source preprocessor — PDFs, URLs, YouTube ingestion | Unofficial API |
| 5 | **MarkItDown** (Microsoft) | Universal file→Markdown conversion; no-Google-account fallback | MIT |
| 6 | **LightRAG** (HKUDS) | Knowledge graph retrieval layer — grounded citations, PostgreSQL storage, bge-m3 embeddings | MIT |
| 7 | **PaperSpine** | Argument layer — motivation confirmation, section blueprints, rewrite matrix | MIT |
| 8 | **scholarstack-pipeline** | 10-stage manuscript writing pipeline — integrity checks, peer review simulation, LaTeX output | MIT (fork — see §11) |

**Evaluated and rejected:**
- **TokenTamer** — code-context compression; irrelevant to prose-heavy academic pipeline.
- **LLM-Wiki-for-n8n** — duplicates LightRAG; reintroduces Google account dependency.
- **OpenDraft's 19-agent architecture** — built for from-scratch thesis generation; ScholarStack's narrower support+verify scope is covered by LightRAG + CritiqueBot + Citation Verification Gate without 19 agent hops of latency and n8n complexity.

---

## 4. LLM Strategy

| Role | Model | Notes |
|---|---|---|
| Primary drafting | `deepseek-v4-pro` | Stages 4–10 manuscript work |
| Entity extraction / light tasks | `deepseek-v4-flash` | LightRAG graph build |
| Adversarial critique | `claude-sonnet-4-6` (Anthropic API) | Deliberately a *different* model family from the generator |
| Scoping / TL;DR / offline fallback | Ollama (local, K45VD) | Expose Mode and cheap utilities run local |

- User-supplied API keys. Estimated **$0.48–$1.20 per full paper run** on DeepSeek vs $15–40 on Claude.
- Legacy DeepSeek model aliases deprecated 2026-07-24 — use only `deepseek-v4-flash` / `deepseek-v4-pro` strings.
- Cost-consciousness is a standing design constraint: every new feature must state which model tier it runs on.

---

## 5. Pipeline Flow (v2.0 — merged)

```
Stage 0 — Topic Scoping
  /scholarstack → student inputs research topic, keywords, scope

Stage 0.5 — Expose Mode (optional, recommended)          [promoted from v1.6 addendum]
  /scope → fast scoping pass on Ollama (~3x cheaper than full run)
  → source count + year spread, author clustering, suggested outline,
    bibliography with DOIs, go/no-go note
  → shares retrieval nodes with /draft; skips PaperSpine + writing agents

Stage 1 — Literature Discovery
  /search-scholar → Google Scholar MCP (broad sweep)
                  → Semantic Scholar MCP (deep retrieval + citation network)
  → student curates shortlist → /export-bib → session.bib

Stage 1.25 — Citation Verification Gate                  [promoted from v1.6 addendum]
  CitationVerifier n8n node — pre-write, not post-hoc:
    - DOI resolves? (CrossRef → DataCite → OpenAlex chain, no LLM call)
      [v2.0.1: DataCite added — arXiv DOIs (10.48550/*) are DataCite-registered
       and resolve on neither CrossRef nor OpenAlex DOI lookup]
    - Title/author/year match claimed metadata?
    - Reject → excluded from citation pool, logged to Audit Log
      (reason: not_found / metadata_mismatch)
  → writing agents only ever see pre-verified sources

Stage 1.5 — Source Ingestion
  notebooklm-py → ingest PDFs/URLs/YouTube from curated list
  MarkItDown → convert non-PDF files
  → summaries route directly to LightRAG (no Obsidian staging)

Stage 2 — Knowledge Graph Build
  /load-refs → session.bib + summaries → LightRAG insert
  LightRAG → entity extraction (deepseek-v4-flash) → PostgreSQL

[INTEGRITY GATE 2.5 — Ralph loop]
  integrity-agent → citation coverage, hallucinated-ref check
  FAIL → loop to Stage 1 with gap list | PASS → proceed

Stage 3 — Argument Structuring
  PaperSpine → motivation confirmation, problem statement, blueprint
  /load-spine → blueprint into pipeline context

Stage 4 — Literature Review Draft
  scholarstack-pipeline Stages 1–3 → LightRAG-grounded draft (deepseek-v4-pro)

[INTEGRITY GATE 4.5 — Ralph loop]
  integrity-agent → citation format, coherence, plagiarism signals
  + CritiqueBot Logic + Evidence Attack (per section)
  FAIL → repair + re-draft | PASS → proceed

Stages 5–10 — Manuscript Development
  scholarstack-pipeline Stages 4–10 → methodology, results, discussion,
  abstract, peer-review simulation, LaTeX output (deepseek-v4-pro)
  + CritiqueBot Claim–Evidence Alignment (abstract)
  + CritiqueBot Scope + Rejection Risk (post-venue-selection)

Post-pipeline
  Cross-Document Consistency Checker (runs once)
  tectonic → PDF compilation
  Zotero MCP → reference sync; Obsidian KB → session archived
```

**Standing rules:** single `/scholarstack` entry command chains all stages with human confirmation gates; **integrity checks cannot be skipped**.

---

## 6. Quality & Integrity Subsystems (from v1.6)

### 6.1 CritiqueBot — Adversarial Reviewer Agent
- Model: `claude-sonnet-4-6`, deliberately separate from the DeepSeek generator.
- Four attack modes:
  1. **Novelty + Feasibility Attack** — fires post-ideation; outputs `proceed: true/false` n8n gate.
  2. **Logic + Evidence Attack** — fires per draft section; flags unsupported assertions, citation-integrity issues, AI-marker phrases.
  3. **Claim–Evidence Alignment** — fires on abstract.
  4. **Scope + Rejection Risk** — fires post-venue-selection.
- Bounded loop per trigger matrix, with human review node as fallback.
- **v2.0 role shift:** with the Citation Verification Gate handling fake-citation detection pre-write, CritiqueBot narrows to reasoning/consistency errors — cheaper and more reliable.

### 6.2 Cross-Document Consistency Checker
- Runs once post-pipeline across all generated documents (proposal, chapters, abstract).
- Also runs automatically before/after each revision (see §7.2).

### 6.3 Research Ethics Hard Stop
- Rule-based n8n Filter node — blocks fabricated citations and unsupported SOTA claims. Not LLM-judged; mechanical.

### 6.4 Verification Audit Log
- JSONL at `~/scholarstack/logs/` on K45VD (home partition — avoids `/dev/sda6` root disk constraint).
- Records: integrity gate results, CritiqueBot verdicts, Citation Verification Gate rejections, revision quality scores.

---

## 7. Standalone Utilities (promoted from v1.6 addendum)

### 7.1 TL;DR Tool
- Command/webhook: drop a PDF/URL → 5-bullet structured summary (thesis, key finding, method, implication, limitation).
- Uses MarkItDown for PDF→text; runs on Ollama/DeepSeek already provisioned. Not part of the main drafting flow.

### 7.2 Revision Mode with Versioning
- Natural-language revision instructions → `draft_v2.md`, `draft_v3.md`, …
- Never overwrites — always a new version; student can diff/roll back.
- Consistency Checker + quality score run automatically before/after each revision; results stored in Audit Log.

---

## 8. Security & Testing (from v1.4)

- **Semgrep CE in CI** — rulesets: `p/secrets`, `p/python`, `p/owasp-top-ten`.
- **OWASP API Security threat model**, scoped to CLI risks:
  - API key handling
  - Subprocess injection in `nlm_to_lightrag.py`
  - SSRF in LightRAG/DeepSeek HTTP calls
  - Insecure file handling for student uploads
- **Testing:** per-phase pytest — script-level tests + Ralph-loop exit-condition tests.
- **Third-party skill intake:** any new skill pack vetted through SkillSpector (static scan) before install on K45VD.

---

## 9. Integration Wiring (build items)

1. **notebooklm-py → LightRAG bridge** — export summaries as Markdown, feed to LightRAG insert API (~30-line Python script).
2. **LightRAG as literature-reviewer backend** — override claude-scholar's `literature-reviewer` agent to query `localhost:9621/query` first.
3. **PaperSpine → scholarstack-pipeline handoff** — `/load-spine` loads `section_blueprints.md` + `confirmed_motivation.md` pre-Stage-2.
4. **CitationVerifier n8n node** — retrieval branch, before sources reach LightRAG context (CrossRef + OpenAlex, no LLM).
5. **`/scope` n8n trigger** — parallel flow to `/draft`, shared retrieval nodes.

---

## 10. Infrastructure

| Item | Detail |
|---|---|
| Compute | K45VD — Ubuntu MATE 24.04, user `lerler`, Tailscale `100.87.252.112` |
| Orchestration | n8n at `n8n.srv1082633.hstgr.cloud` (Hostinger VPS) |
| LightRAG server | `localhost:9621`; PostgreSQL storage; `BAAI/bge-m3` embeddings |
| LaTeX | tectonic |
| Credentials in n8n | `DEEPSEEK_API_KEY`, `ANTHROPIC_API_KEY` |
| Logs | `~/scholarstack/logs/` (home partition) |

---

## 11. Licensing & Release Blockers

- Project license: **AGPL-3.0**, target `github.com/lerlerchan/scholarstack`.
- **RELEASE BLOCKER:** `academic-research-skills` is CC-BY-NC 4.0 — incompatible with AGPL-3.0 and commercial workshop use. The clean MIT fork **`scholarstack-pipeline`** must fully replace it before any public release. Original author permission/attribution email still outstanding.
- Light-skills (MIT) — structural patterns freely borrowable, no compatibility issue.

---

## 12. Rollout Phases

| Phase | Scope | Timing |
|---|---|---|
| Phase 1 — OSS | GitHub release under AGPL-3.0, K45VD-style local deployment, user-supplied API keys | After release blocker cleared |
| Phase 2 — Cloud | Laravel on lertech.cloud, paid tiers | Deferred 3–6 months after Phase 1 |

---

## 13. Under Evaluation (not committed)

### 13.1 Hermes Agent front door (discussed 2026-07-19)
Hermes Agent (Nous Research) as a conversational front end so non-technical scholars interact via Telegram instead of n8n directly. Proposed division of labour if adopted:

- **Hermes** = conversational front door, intent routing, per-student memory.
- **n8n** = the deterministic pipeline (unchanged); exposed to Hermes as constrained tools/skills via webhooks.

Conditions before adoption:
1. Disable Hermes's self-improving skill loop for student-facing tenants (conflicts with Audit Log / Ethics Hard Stop auditability).
2. Solve multi-tenancy — Hermes is a single-user agent; per-student Docker isolation needed; budget K45VD resource cost.
3. Hard sandboxing — minimal toolset, workspace-scoped filesystem, cron creation off.

Cheaper alternative under comparison: thin Telegram router bot (ZhiFuu QiBot / StaffBot Pro pattern) in front of n8n webhooks — one orchestrator, full determinism, no Hermes memory/skills.

Decision hinges on user story: pipeline trigger ("run my lit review") → thin router wins; open research conversation ("help me think through methodology") → Hermes earns its complexity.

### 13.2 Candidate skill packs (from ranked-list comparison, 2026-07-12)
- **Auto-Research-In-Sleep** — unattended overnight runs via n8n scheduling. Vet via SkillSpector first.
- **Auto Empirical Research Skills** — empirical social science methods (relevant to GBQ/AHP questionnaire work). Vet via SkillSpector first.
- All other ranked packs: skip — redundant or out of scope.

---

## 14. Open Issues

| # | Issue | Status |
|---|---|---|
| 1 | `bge-m3` embedding adequacy for Manglish-heavy theses | Open — needs empirical test |
| 2 | Google Scholar MCP rate limiting in workshop use | Open — monitor; fallback to Semantic Scholar only |
| 3 | Semantic Scholar coverage of Malaysian/SEA conference papers | Open — spot-check |
| 4 | notebooklm-py stability (unofficial API) | Open — monitor for breakage |
| 5 | academic-research-skills author permission for MIT fork | Open — email to send (release blocker) |
| 6 | Hermes vs thin-router front door decision | Open — see §13.1 |

---

## 15. Installation Sequence (K45VD)

```bash
# 1. Clone repos
cd ~/scholarstack
git clone https://github.com/HKUDS/LightRAG lightrag
git clone https://github.com/lerlerchan/scholarstack-pipeline   # MIT fork — release blocker until ready
git clone https://github.com/lerlerchan/claude-scholar
git clone https://github.com/lerlerchan/notebooklm-py
git clone https://github.com/lerlerchan/paperspine

# 2. Python env
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
pip install markitdown[all] --break-system-packages

# 3. LightRAG
pip install "lightrag-hku[api]" --break-system-packages
# .env: LLM_MODEL=deepseek-v4-flash, LLM_BINDING=openai,
#       OPENAI_API_KEY=<DEEPSEEK_API_KEY>, OPENAI_BASE_URL=https://api.deepseek.com,
#       EMBEDDING_MODEL=BAAI/bge-m3
lightrag-server &   # localhost:9621

# 4. MCPs
python -m venv .venv-scholar-mcp && source .venv-scholar-mcp/bin/activate
pip install google-scholar-mcp-server && deactivate
claude mcp add semantic-scholar -s user -- uvx \
  --from git+https://github.com/akapet00/semantic-scholar-mcp semantic-scholar-mcp

# 5. Keys
echo "DEEPSEEK_API_KEY=sk-..." >> ~/.env
echo "DEEPSEEK_DEFAULT_MODEL=deepseek-v4-flash" >> ~/.env
echo "DEEPSEEK_REASONING_MODEL=deepseek-v4-pro" >> ~/.env
# ANTHROPIC_API_KEY as n8n credential (CritiqueBot)
```
