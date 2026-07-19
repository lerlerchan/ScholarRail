# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Start here

**New session? Read `docs/HANDOFF.md` first** — cold-start guide: file map, verification commands, gotchas, design invariants, TODO. Implementation status lives in `plan/implementation-plan.md`.

## What this repo is

Documentation-only planning repo for **ScholarStack** — a postgraduate research pipeline (raw topic → Scopus-ready manuscript with verified citations). No code, no build, no tests here. The single source of truth is `docs/scholarstack-prd-v2.0.md`.

The actual components live in separate repos (LightRAG, scholarstack-pipeline, claude-scholar, notebooklm-py, paperspine) and are deployed on the K45VD machine under `~/scholarstack/`. Orchestration runs in n8n, not in this repo.

## Working on the PRD

- The PRD is versioned by consolidation: v2.0 supersedes v1.0–v1.6 + addendum. New decisions go into the existing v2.0 file (bump Version History table), not new sidecar docs.
- Every new feature must state which LLM tier it runs on (`deepseek-v4-flash` / `deepseek-v4-pro` / `claude-sonnet-4-6` / Ollama local) — cost-consciousness is a standing design constraint.
- Integrity checks are non-skippable by design; don't propose flows that bypass the Citation Verification Gate, integrity gates, or Ethics Hard Stop.
- Rejected tools (TokenTamer, LLM-Wiki-for-n8n, OpenDraft's 19-agent architecture) are documented in §3 with reasons — don't re-propose without new evidence.

## Key constraints to preserve

- **License:** project is AGPL-3.0. Release blocker: `academic-research-skills` (CC-BY-NC 4.0) must be fully replaced by the MIT fork `scholarstack-pipeline` before public release.
- **Adversarial separation:** CritiqueBot deliberately uses a different model family (Claude) than the generator (DeepSeek). Keep that separation in any pipeline change.
- **Verification is mechanical where possible:** Citation Verification Gate uses CrossRef + OpenAlex APIs with no LLM call; Ethics Hard Stop is a rule-based n8n filter. Don't convert these to LLM judgment.
- Open issues and under-evaluation items (Hermes front door vs thin Telegram router) are tracked in §13–§14 — update there, not in chat only.
