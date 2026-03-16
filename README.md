# customgpt-ai-rag

A Claude Code plugin that gives Claude Code persistent semantic search over any project using [CustomGPT.ai](https://app.customgpt.ai)'s RAG engine. Index your codebase, documents, or any file collection once, then ask questions across everything instantly — no context overflow, no re-reading files.

**No build step. No Node.js. No Python. Requires only `curl`.**

---

## How It Works

Each project maps to one CustomGPT.ai agent. The agent ID and indexed folder are stored in `.customgpt-meta.json` at your project root. All skills read this file automatically — you never need to pass an agent ID manually.

Files are uploaded to CustomGPT.ai's servers, where they are chunked, embedded, and indexed. Queries run against that index and return AI-generated answers with source citations.

---

## Prerequisites

| Requirement | How to get it |
|---|---|
| CustomGPT.ai account | Sign up at [app.customgpt.ai](https://app.customgpt.ai) |
| API key | [app.customgpt.ai/profile#api-keys](https://app.customgpt.ai/profile#api-keys) |
| Claude Code | [claude.ai/code](https://claude.ai/code) |
| `curl` | Pre-installed on macOS, Linux, Windows 10+ |

---

## Installation

```
/plugin marketplace add https://github.com/adorosario/customgpt-skill-plugin
/plugin install customgpt-ai-rag
/reload-plugins
```

### Updating

```
/plugin marketplace update https://github.com/adorosario/customgpt-skill-plugin
/plugin install customgpt-ai-rag
/reload-plugins
```

---

## API Key Setup

The plugin resolves your API key automatically, checking these locations in order:

1. **Environment variable** — `$CUSTOMGPT_API_KEY`
2. **`.env` file** — walks up from `$PWD` looking for `CUSTOMGPT_API_KEY=`
3. **Config file** — `~/.claude/customgpt-config.json` → `apiKey` field

To save your key for all projects:

```bash
echo '{"apiKey":"YOUR_KEY_HERE"}' > ~/.claude/customgpt-config.json
```

If no key is found, the plugin will prompt you and save it automatically.

---

## Quick Start

```
1. /create-agent       → index this project into a new CustomGPT.ai agent
2. /check-status       → wait until indexing completes (index_status: ok)
3. /query-agent        → ask questions across your entire codebase
```

### Example

```
you: /create-agent
     → uploads all eligible files in the project to a new CustomGPT.ai agent
     → saves agent ID to .customgpt-meta.json

you: /check-status
     → shows indexing progress: pages_found, pages_indexed, index_status

you: /query-agent where is authentication handled?
     → returns an AI answer with citations pointing to the relevant files
```

---

## Skills Reference

### Setup & Indexing

| Skill | Argument | What it does |
|---|---|---|
| `/create-agent` | `[folder path]` | Create a new agent and upload an entire folder. Defaults to the Git root or `$PWD`. |
| `/index-files` | `[file, directory, or blank]` | Add specific files or a directory to an existing agent. Auto-creates an agent if none exists. |
| `/refresh-agent` | — | Wipe all indexed documents and re-upload everything. Use after large changes. |
| `/reindex-file` | `[filename or path]` | Delete and re-upload a single changed file. Handles both uploaded files and URL-based documents. |

### Querying

| Skill | Argument | What it does |
|---|---|---|
| `/query-agent` | `[your question]` | Ask a plain-language question. Returns an AI-generated answer with source citations. Warns if files have changed since the last index. |

### Monitoring & Management

| Skill | Argument | What it does |
|---|---|---|
| `/check-status` | — | Agent metadata, chat availability, and indexing statistics. |
| `/list-pages` | — | All indexed documents with their IDs and crawl/index status. |
| `/check-page` | `[filename or URL]` | Look up whether a specific file is indexed. |
| `/delete-page` | `[page ID]` | Remove a document from the knowledge base by page ID. |
| `/delete-agent` | — | Permanently delete the agent and remove `.customgpt-meta.json`. Requires confirmation. |

### Help

| Skill | Argument | What it does |
|---|---|---|
| `/help` | `[setup \| index \| query \| manage]` | Show available skills, prerequisites, and routing guidance. |

---

## Natural Language Triggers

You don't have to type slash commands. The plugin activates on natural language too:

```
"index this repo"         → /create-agent
"add this file"           → /index-files
"search my codebase"      → /query-agent
"what does my code say about X" → /query-agent
"show all indexed files"  → /list-pages
"re-sync everything"      → /refresh-agent
```

---

## Supported File Types

**Code:** `.js` `.ts` `.tsx` `.jsx` `.py` `.go` `.rb` `.java` `.cs` `.cpp` `.rs` `.swift` `.kt` `.php` `.sh` `.html` `.css` `.scss` `.vue` `.svelte` `.sql` `.graphql` `.proto` and more

**Config / Data:** `.json` `.yaml` `.toml` `.xml` `.ini` `.cfg` `.env.example`

**Docs:** `.md` `.rst` `.txt` `.pdf` `.docx` `.pptx` `.xlsx` `.csv`

**Images:** `.jpg` `.jpeg` `.png` `.webp` (with optional AI Vision processing)

**Excluded automatically:** `.git/` `node_modules/` `__pycache__/` `dist/` `build/` `.next/` `.venv/` `vendor/` `.env` `.env.*`

---

## Image Indexing (AI Vision)

When indexing image files, the plugin will ask whether to enable AI Vision processing. This uses CustomGPT.ai's vision model to extract richer content from images (diagrams, screenshots, scanned documents).

```
Image files detected (3 images). Enable AI Vision processing? (yes/no)
Compress images before vision processing? (yes/no)
```

---

## Keeping the Index Current

After files change, update the index with:

- **One file changed** → `/reindex-file path/to/file.py`
- **Several files changed** → `/index-files src/` to add a directory
- **Large refactor** → `/refresh-agent` to wipe and re-upload everything

The `/query-agent` skill automatically warns you if any files in the indexed folder are newer than the last index update:

```
Note: 4 file(s) have changed since the last index update.
Results may not reflect the latest changes. Run /refresh-agent to re-sync.
```

---

## Project Files

| File | Purpose |
|---|---|
| `.customgpt-meta.json` | Agent binding for this project (auto-created, **commit or gitignore as needed**) |
| `~/.claude/customgpt-config.json` | Your API key (never committed, stored locally) |

Example `.customgpt-meta.json`:

```json
{
  "agent_id": 12345,
  "agent_name": "my-project",
  "indexed_folder": "/home/user/projects/my-project",
  "created_at": "2025-01-01T00:00:00Z"
}
```

---

## Which Skill Should I Use?

| Situation | Skill |
|---|---|
| First time setting up | `/create-agent` |
| Adding new files to an existing agent | `/index-files [path]` |
| One file changed | `/reindex-file [filename]` |
| Many files changed or index is stale | `/refresh-agent` |
| Ask a question across the project | `/query-agent [question]` |
| Check if indexing is done | `/check-status` |
| See everything in the knowledge base | `/list-pages` |
| Check if a specific file is indexed | `/check-page [filename]` |
| Remove a document | `/delete-page [id]` |
| Start fresh | `/delete-agent` then `/create-agent` |

---

## Privacy & Billing

Your files are uploaded to CustomGPT.ai's servers for indexing. The plugin uses your existing CustomGPT.ai subscription's API quota — no additional charge from the plugin itself. Your API key is stored locally and only sent to the CustomGPT.ai API for authentication.

---

## License

MIT
