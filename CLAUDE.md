# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

A Claude Code plugin (`customgpt-ai-rag`) that gives Claude Code persistent semantic search over any project using CustomGPT.ai's RAG engine. Skills are Markdown files — Claude reads instructions and executes `curl` calls. No build step, no runtime dependencies.

## Repository Layout

```
skills/<skill-name>/SKILL.md   — One skill per directory
skills/_shared/api-key.md      — Shared API key lookup procedure (referenced by all skills)
.claude-plugin/plugin.json     — Plugin manifest (name, version, keywords)
.claude-plugin/marketplace.json — Registry metadata
.customgpt-meta.json           — Per-project agent binding (agent_id, indexed_folder)
openapi.json                   — Full CustomGPT.ai API spec (authoritative reference)
```

## Skill File Format

Each `SKILL.md` starts with YAML frontmatter:

```yaml
---
name: customgpt-ai-rag:<skill-name>
description: One-line description shown in skill picker
argument-hint: "[example argument syntax]"
allowed-tools: Bash, Read
triggers:
  - "phrase that activates this skill"
  - "another trigger phrase"
---
```

The body is step-by-step instructions for Claude. `curl` code blocks handle API calls; prose handles logic, conditionals, and iteration. No shell loops — Claude iterates file-by-file.

## Available Skills

| Skill | Purpose |
|---|---|
| `help` | Onboarding, skill overview, routing guide |
| `create-agent` | Create a new agent and upload an entire folder |
| `index-files` | Add specific files or a directory to an existing agent |
| `refresh-agent` | Wipe and re-upload everything (full re-sync) |
| `reindex-file` | Delete and re-upload a single changed file |
| `query-agent` | Ask a plain-language question; get answer + citations |
| `check-status` | Agent metadata, chat availability, indexing stats |
| `list-pages` | All indexed documents with IDs and status |
| `check-page` | Look up whether a specific file/URL is indexed |
| `delete-page` | Remove a document from the knowledge base |
| `delete-agent` | Permanently delete the agent and meta file |

## Core Design Patterns

**API key resolution** (all skills reference `skills/_shared/api-key.md`):
1. `$CUSTOMGPT_API_KEY` env var
2. `.env` file — walk up from `$PWD` to `/`
3. `~/.claude/customgpt-config.json` → `apiKey` field

**Agent binding:** `.customgpt-meta.json` in the project root stores `agent_id` and `indexed_folder`. All skills walk up from `$PWD` to find it.

**File eligibility:** All upload skills share the same `find` command — code, web, data, docs, and images are included; `.git/`, `node_modules/`, `__pycache__/`, build dirs, cache dirs, and `.env*` are excluded.

**Destructive operations** (`delete-agent`, `delete-page`, `refresh-agent`) always confirm with the user before executing. `delete-agent` requires the user to type a specific phrase.

**Pagination:** Skills that list documents fetch 100 at a time and iterate until `data.pages.last_page` is reached. Claude handles iteration in prose — no shell loops.

**Freshness tracking:** After uploads, skills `touch` `.customgpt-meta.json`. `query-agent` warns if any files in `indexed_folder` are newer than the meta file.

**URL vs file documents:** The `/pages/{pageId}/reindex` API endpoint only works for URL-based documents. For uploaded files, the pattern is always delete-then-re-upload. `reindex-file` handles both cases.

## CustomGPT.ai API

Base URL: `https://app.customgpt.ai`

Key endpoints:

| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `/api/v1/projects` | Create agent (multipart/form-data) |
| `GET` | `/api/v1/projects/{id}` | Agent details |
| `DELETE` | `/api/v1/projects/{id}` | Delete agent |
| `GET` | `/api/v1/projects/{id}/stats` | Indexing statistics |
| `POST` | `/api/v1/projects/{id}/sources` | Upload a file (multipart/form-data) |
| `GET` | `/api/v1/projects/{id}/pages` | List documents (paginated, `?limit=100`) |
| `DELETE` | `/api/v1/projects/{id}/pages/{pageId}` | Remove a document |
| `POST` | `/api/v1/projects/{id}/pages/{pageId}/reindex` | Re-crawl URL document |
| `POST` | `/api/v1/projects/{id}/conversations` | Create query session |
| `POST` | `/api/v1/projects/{id}/conversations/{sessionId}/messages` | Send question |

Auth: `Authorization: Bearer {API_KEY}` on every request.

Image upload fields: `is_vision_enabled=true`, `ocr_mode=2` (AI Vision), `is_vision_compress_image=true|false`.

## Adding or Modifying Skills

1. Create `skills/<skill-name>/SKILL.md` with the frontmatter format above
2. Reference `skills/_shared/api-key.md` for the API key step — don't copy-paste it
3. Use the canonical `find` command from an existing upload skill for file collection — keep exclusions and extensions in sync across all skills
4. All destructive operations must confirm with the user before executing
5. Use `--form` (multipart) for endpoints that accept file uploads; `--data` / `--data-urlencode` (url-encoded) for text-only endpoints
6. All skills should reference other skills by name (e.g., "run `/check-status`") to guide users to the right next step
