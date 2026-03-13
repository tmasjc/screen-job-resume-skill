# screen-job-resume

A Claude skill that extracts structured data from resumes, scores credibility of candidate claims, assesses role fit, and saves results to MongoDB — all in one pass.

## What it does

Every resume goes through four stages:

1. **Extract** — parse into structured JSON (`resume_schema_template.json`)
2. **Credibility screen** — score (1–10) and flag metric inflation, ownership vagueness, scope mismatches, and internal inconsistencies
3. **Role fit** — compare against a JD or inferred target role, adjusted for credibility
4. **Save to MongoDB** — upsert via MCP with email/phone dedup

Output is a compact chat-friendly report: candidate card, credibility score, fit recommendation (Strong yes / Yes / Maybe / No), and save status.

## Setup

Requires a [MongoDB MCP server](https://github.com/mongodb-js/mongodb-mcp-server) configured in your Claude host config (e.g. `claude_desktop_config.json`). No credentials are stored in this repo.

## Usage

Invoke as a Claude skill — see [SKILL.md](SKILL.md) for the full workflow, scoring rubric, and design rationale.
