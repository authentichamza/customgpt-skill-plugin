---
name: customgpt-ai-rag:help
description: Show available CustomGPT.ai RAG skills, prerequisites, and how to get started with semantic search over your project.
argument-hint: "[topic: setup | index | query | manage]"
allowed-tools: Read
triggers:
  - "customgpt help"
  - "rag help"
  - "how do I use customgpt"
  - "what customgpt skills are available"
  - "what can the customgpt plugin do"
  - "get started with rag"
  - "setup customgpt"
  - "customgpt getting started"
  - "help with customgpt"
  - "how does rag search work"
---

# CustomGPT.ai RAG Search — Help

Give Claude Code a persistent semantic search index over your entire project. Ask questions across thousands of files instantly — no re-reading, no context overflow.

## Prerequisites

| Requirement | Details |
|---|---|
| CustomGPT.ai account | Sign up at https://app.customgpt.ai |
| API key | https://app.customgpt.ai/profile#api-keys |
| `curl` | Pre-installed on macOS, Linux, Windows 10+ |

## Quick Start

```
1. /create-agent          → index this folder into a new CustomGPT.ai agent
2. /check-status          → wait until pages are indexed (index_status: ok)
3. /query-agent           → ask questions across your entire codebase
```

## Skill Reference

### Setup & Indexing

| Skill | What it does |
|---|---|
| `/create-agent` | Create a new agent and upload an entire folder |
| `/index-files [path]` | Add specific files or a directory to an existing agent |
| `/refresh-agent` | Wipe and re-upload everything (full re-sync after large changes) |
| `/reindex-file [file]` | Delete and re-upload a single changed file |

### Querying

| Skill | What it does |
|---|---|
| `/query-agent [question]` | Ask a plain-language question; get an answer with source citations |

### Monitoring & Management

| Skill | What it does |
|---|---|
| `/check-status` | Agent metadata, chat availability, and indexing statistics |
| `/list-pages` | All indexed documents with their IDs and crawl/index status |
| `/check-page [name]` | Look up whether a specific file or URL is indexed |
| `/delete-page [id]` | Remove a document from the knowledge base by page ID |
| `/delete-agent` | Permanently delete the agent and remove the local meta file |

## Which skill should I use?

**My files just changed** → `/reindex-file <filename>` for one file, or `/refresh-agent` for everything

**Index is stale across the whole project** → `/refresh-agent`

**I want to search without a full re-index** → `/query-agent` (it warns you if files have changed)

**I don't know if a file is indexed** → `/check-page <filename>`

**I want to see everything in the knowledge base** → `/list-pages`

**I want to start fresh on a different folder** → `/delete-agent` then `/create-agent` on the new folder

## How the index works

Each Claude Code project maps to one CustomGPT.ai agent. The agent ID and indexed folder are stored in `.customgpt-meta.json` at the project root. All skills read this file automatically.

After uploading files, CustomGPT.ai crawls and indexes them asynchronously. Use `/check-status` to monitor progress — queries work best once `pages_indexed` equals `pages_found`.

## Privacy & billing

Your files are uploaded to CustomGPT.ai's servers for indexing. The plugin uses your existing subscription's API quota — no extra charge. Your API key is stored locally in `~/.claude/customgpt-config.json` and never leaves your machine except when authenticating with the CustomGPT.ai API.
