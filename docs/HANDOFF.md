# ScholarRail — Implementation Handoff

> **Rename note (2026-07-20):** product renamed ScholarStack → **ScholarRail**. GitHub repos renamed (old URLs redirect). Local paths (`~/scholarstack/`, `scholarstack.py`, `~/github/ScholarStack/`) keep the old name — renaming live paths breaks nothing-worth-fixing; treat the names as synonyms.

**Written:** 2026-07-19, end of the session that built Phases 1–5.
**Audience:** any future Claude instance (Opus, Sonnet, or other) or human developer picking this up cold. Assumes zero prior session context. Read this + `plan/implementation-plan.md` + the PRD before touching anything.

---

## 1. What this project is (30 seconds)

ScholarStack takes a postgraduate student from research topic → Scopus-ready manuscript, with **mechanically verified citations** at every gate. It is a pipeline of open-source tools glued by small Python scripts, not a monolith. The core promise: **writing LLMs only ever see pre-verified sources, and every gate decision is audit-logged.** Spec: `docs/scholarstack-prd-v2.0.md` (v2.0.1). Owner: LerLer Chan (lerler).

## 2. Where everything lives

Two repos + one deployment directory:

| Location | What | Git remote |
|---|---|---|
| `~/github/ScholarStack` | Docs, PRD, plan, this handoff. **No code.** | github.com/lerlerchan/ScholarRail |
| `~/scholarstack/scholarstack-pipeline` | Manuscript pipeline (stages 4–10), MIT | github.com/lerlerchan/scholarrail-pipeline |
| `~/scholarstack/` | Deployment: all glue scripts, venv, clones, logs. **Not a git repo** (contains `.env` with API key — never commit it) | none |

Key files in `~/scholarstack/` (all have tests in `~/scholarstack/tests/`):

| File | Role (PRD ref) |
|---|---|
| `citation_verifier.py` | Stage 1.25 gate. BibTeX → CrossRef→DataCite→OpenAlex resolver chain. No LLM. Exit 0/1 |
| `ingest.py` | Stage 1.5. Any file → MarkItDown (library call, never shell) → LightRAG, `file_source`=basename |
| `integrity_check.py` | Gate 2.5. Verified-pool keys vs ingested doc stems, bidirectional. Exit 0/1 |
| `load_spine.py` | §9.3. PaperSpine `paper_rewriting_output/` → context JSON. Fails if no `confirmed_contribution.md` |
| `draft_section.py` | Stage 4 single-section drafting + `check_citations()` (used by pipeline too) |
| `revise.py` | §7.2 Revision Mode. Always writes `_vN+1`, never overwrites |
| `tldr.py` | §7.1. File/URL → 5 bullets on v4-flash |
| `consistency_check.py` | §6.2. Cross-doc contradiction check on v4-flash. Exit 0/1 |
| `llm.py` | Shared DeepSeek chat helper (reads key from `~/scholarstack/.env`) |
| `scholarstack.py` | **Single entry** (PRD §5 standing rule): verify→ingest→gate→spine→pipeline→consistency→PDF |

Other assets: CritiqueBot agent at `~/.claude/agents/critiquebot.md`; `paper-spine` skill at `~/.claude/skills/paper-spine/`; MCPs `semantic-scholar` + `google-scholar` registered user-scope in `~/.claude.json`; `tectonic` + `pandoc` static binaries in `~/.local/bin`.

## 3. How to verify the system works (run this first)

```bash
# server up? (start if not: cd ~/scholarstack && setsid nohup .venv/bin/lightrag-server \
#   > logs/lightrag-server.log 2>&1 < /dev/null & disown)
curl -s localhost:9621/health

# offline unit tier — free, <30s, no network. 21 tests must pass:
~/scholarstack/.venv/bin/pytest ~/scholarstack/tests ~/scholarstack/scholarstack-pipeline/tests -m "not integration"

# live tier — costs cents (DeepSeek) + ~5 min. 8 tests:
export PATH="$HOME/.local/bin:$PATH"
~/scholarstack/.venv/bin/pytest ~/scholarstack/tests ~/scholarstack/scholarstack-pipeline/tests -m integration
```

If unit tests fail, something rotted — fix before building anything new.

## 4. Hard-won gotchas (do not rediscover these)

1. **arXiv DOIs (10.48550/\*) are DataCite-registered.** CrossRef 404s them; OpenAlex DOI-lookup misses them too. `fetch_datacite()` in the resolver chain is load-bearing. This was an empirical PRD amendment (v2.0.1).
2. **Ollama model name is `bge-m3`, not `BAAI/bge-m3`.** The HF-style name 404s against Ollama and halts LightRAG's whole pipeline.
3. **Ollama cold-load exceeds LightRAG's default 60s embed timeout.** `EMBEDDING_TIMEOUT=180` is set in `~/scholarstack/.env`. If embed timeouts reappear, warm the model first (`curl localhost:11434/api/embed -d '{"model":"bge-m3","input":"x"}'`).
4. **LightRAG dedups**: same `file_source` name → 409; identical *content* under a new name → failed doc. Tests embed timestamps in both name and content.
5. **LightRAG `/documents/text` requires `file_source`** field; processing is async — poll `/documents` for state `processed` (30s is not enough; budget 3 min).
6. **Google Scholar scraping is blocked from this IP** (`MaxTriesExceededException`). Not a bug in our code. Semantic Scholar is the primary search path once its key exists.
7. **No sudo on K45VD.** Static binaries to `~/.local/bin` (tectonic, pandoc pattern). PostgreSQL not installed — LightRAG runs file-based storage; that's fine at current scale.
8. **`pkill -f lightrag-server` from a compound Bash call kills your own shell** (exit 144). Restart the server in a separate `setsid nohup ... & disown` command.
9. **DeepSeek key lives in `~/scholarstack/.env`** as `LLM_BINDING_API_KEY` (copied from `~/github/Agent_K_Telegram/.env`). `llm.py` and `draft_section.py` fall back to it. Never print or commit it.

## 5. Non-negotiable design invariants

Preserve these in any change — they are the product:

- **Verification is mechanical where specified.** Citation gate = HTTP lookups, no LLM. Gate 2.5 = set comparison. Ethics stop (future n8n) = rule filter. Do not "improve" them into LLM judgment.
- **Writers only see the verified pool.** Citation format is `[@key]`; after every generation, `check_citations()` runs; violations → bounded re-draft (max 2) → hard fail. Never widen.
- **Adversarial separation:** CritiqueBot is Claude-family, generator is DeepSeek. Keep families split.
- **Every gate decision appends to `~/scholarstack/logs/audit.jsonl`.** New gates must audit too.
- **Revisions never overwrite** — always `_vN+1`.
- **Clean-room:** `~/scholarstack/reference-academic-research-skills-CCBYNC` (CC-BY-NC) has **never been read** by the implementing session. `scholarstack-pipeline` was designed from PRD §5 stage names only. Keep it that way: do not open that directory's files, or the MIT claim weakens.
- **Model tiers (PRD §4):** v4-pro = manuscript prose; v4-flash = extraction/summaries/checks; Ollama = free local; Claude = critique only. State the tier for any new feature.

## 6. What is NOT done (ordered by value)

1. **API keys (user actions, everything else is ready):** Semantic Scholar (free form), Scopus (free academic — note commercial license needed if the paid cloud phase uses it), `ANTHROPIC_API_KEY` for the n8n CritiqueBot node, Scholar Gateway OAuth (Wiley corpus, complement only).
2. **First real end-to-end run** — the chain is tested piecewise + offline-e2e, but no real student paper has flowed through `scholarstack.py`. Do this before adding features.
3. **n8n wiring** (n8n.srv1082633.hstgr.cloud): wrap `citation_verifier.py`, `/scope`, `/draft` as webhook-triggered flows. All K45VD-side scripts already speak exit codes for this.
4. **Phase 6 thin Telegram router** → n8n webhooks. Blocked on 3. **Hermes stays deferred** — see plan for rationale; don't relitigate without a semester of router traffic data.
5. **Package extraction**: `pipeline.py` imports shared modules via `sys.path` from `~/scholarstack`. Before any public release announcement, move shared code into the `scholarstack-pipeline` package proper so CI can run the full suite.
6. **notebooklm-py integration** (clone at `~/scholarstack/notebooklm-py`) — optional; MarkItDown covers ingestion.
7. **Expose Mode (`/scope`)** — PRD Stage 0.5, cheap Ollama scoping pass. Not built.
8. **Courtesy email** to academic-research-skills author — no longer a release blocker (clean-room), still polite.

## 7. Working conventions this project uses

- Plan + status live in `plan/implementation-plan.md` — append status sections there, commit with Conventional Commits, push to origin (established pattern; user says "commit"/"push" freely).
- Tests: two tiers via pytest marker `integration`. Unit tier must stay offline (<1s) — monkeypatch every fetcher/LLM call. Every new gate gets a "planted fake" style functionality test.
- PRD changes: edit `docs/scholarstack-prd-v2.0.md` in place, bump the Version History table (v2.0.x for empirical corrections).
- Third-party skill/MCP intake: `skillspector scan --no-llm <dir>` first (binary at `~/.local/bin/skillspector`); CRITICAL verdict = reject and record in plan.
- Persistent cross-session memory: `~/.claude/projects/-home-lerler-github-ScholarStack/memory/` (index in `MEMORY.md` there).
- The user (lerler) works in short imperative messages, often just URLs + a verb. URLs = "evaluate/integrate this". "yes"/"continue" = proceed with the stated next steps. Prefers autonomous execution, terse reporting, lazy-but-correct engineering.
