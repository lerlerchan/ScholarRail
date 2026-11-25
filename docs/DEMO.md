# ScholarRail demo — from `session.bib` to a verified manuscript

Think of ScholarRail as guardrails on a paper-writing pipeline: the LLM can only cite what passed a mechanical check. This walkthrough shows the first gate, which runs without LightRAG.

## 0. Setup (one time)

```bash
cd ~/scholarstack
python3 -m venv .venv && .venv/bin/pip install -e scholarstack-pipeline bibtexparser requests
python3 scholarstack-pipeline/setup_keys.py      # hidden prompts -> ~/scholarstack/.env (600)
# LightRAG server on :9621, plus pandoc + tectonic on PATH, are needed for stages 2+
```

## 1. Start with a bibliography you collected

`session.bib` has two real papers and one invented one:

| key | DOI | truth |
|---|---|---|
| `vaswani2017` | 10.48550/arXiv.1706.03762 | real |
| `lecun2015` | 10.1038/nature14539 | real |
| `fake2024` | 10.1234/fake.2024.99999 | does not exist |

## 2. Run the Citation Verification Gate (no LLM involved)

```bash
.venv/bin/python citation_verifier.py session.bib
```

Real output (captured 2026-11-25): `vaswani2017` and `lecun2015` come back `verified`. `fake2024` is `rejected`, reason `not_found`: "resolves on none of CrossRef/DataCite/OpenAlex". Exit code is 1, so any n8n or shell chain stops. Every decision is appended to `logs/audit.jsonl`.

## 3. The rest of the chain

```bash
.venv/bin/python scholarstack.py <workdir>     # add --auto to skip the human gates
```

| Stage | What happens | Skippable? |
|---|---|---|
| 1.25 Verify | gate above, writes `verified.json` | no |
| 1.5 Ingest | each verified source goes to LightRAG via MarkItDown | no |
| 2.5 Coverage | every verified cite has a source, every source is verified | no |
| 3 Spine | PaperSpine contribution + blueprint via `/load-spine` | no |
| 4-10 Draft | section by section, grounded in retrieved context; `[@key]` checked against the verified pool, 2 re-drafts then hard fail | gates pause for you unless `--auto` |
| Review | CritiqueBot (Claude, a different model family) attacks logic and evidence | |
| Deliver | consistency check, then `paper.pdf` via pandoc + tectonic | |

Revisions always create `draft_vN+1`; nothing is overwritten.

## What you do vs what ScholarRail does

- **You:** pick the topic, curate `session.bib`, supply sources, confirm the contribution, approve each stage.
- **ScholarRail:** blocks fake references, keeps drafts grounded, logs every gate, builds the PDF.

## Not shown here

Stages 2+ need a running LightRAG server and paid API keys (DeepSeek, Anthropic), so they are described, not demonstrated. n8n wiring and the Telegram front door are not built yet.
