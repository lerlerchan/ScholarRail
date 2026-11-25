# Privacy

Last updated: 2026-11-25. Applies to the ScholarRail software and to https://scholarrail.staff-bot.com.

## The website

- It is a static page. It sets **no cookies**, runs **no analytics**, loads **no third-party scripts, fonts or trackers**, and has **no forms**.
- Like any web server, the host keeps standard access logs (IP address, time, requested path, browser user-agent). They are used only to keep the site running and secure. They are not analysed for marketing, sold or shared.
- Links to GitHub leave the site; GitHub's own privacy policy applies there.

## The software

ScholarRail runs **on your own machine**. The project operates no servers that receive your manuscripts, sources, keys or audit log, and the code contains no telemetry.

Your data leaves your machine only when you run a stage that calls a service you configured:

| Service | What is sent | Why |
|---|---|---|
| DeepSeek (`api.deepseek.com`) | Retrieved source excerpts and prompts | Drafting and entity extraction |
| Anthropic | Draft sections | CritiqueBot review, if you enable it |
| CrossRef, DataCite, OpenAlex, doi.org | DOI, title, author, year of each cited work | Citation verification (no manuscript text) |
| Semantic Scholar, Scopus | Search queries | Literature discovery, if you enable it |

Each provider handles that data under its own terms; check them before uploading confidential, unpublished or ethics-restricted material. Your local LightRAG server and its database stay on your machine.

## What is stored locally

- API keys in `.env` (file mode 600). Never commit this file.
- The verification audit log (`audit.jsonl`), drafts, and the knowledge-graph store. Delete them any time; nothing is mirrored elsewhere.

## Planned hosted plans

Hosted Cloud plans shown on the site are a **draft preview and do not exist yet**. This policy will be updated, with what is collected and why, before any account or payment feature launches.

## Contact

Open an issue at https://github.com/lerlerchan/ScholarRail/issues for privacy questions or deletion requests.
