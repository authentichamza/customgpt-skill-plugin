---
description: Check the indexing status of a CustomGPT.ai agent — shows project info, chat status, and page crawl/index stats.
triggers:
  - "check status"
  - "agent status"
  - "index status"
  - "check agent"
  - "how many pages"
  - "pages indexed"
  - "pages crawled"
  - "indexing progress"
  - "check index"
  - "status of agent"
  - "status of the agent"
---

# check-status

Show the current status of a CustomGPT.ai agent: project metadata, chat availability, and page crawl/index statistics.

## Critical Rules
- ALWAYS read `.customgpt-meta.json` to get `agent_id`
- Make two API calls: one for project info, one for stats
- Present results in a readable summary — do NOT dump raw JSON

---

## Step 1 — Get API Key

Check in order:
1. Env var `$CUSTOMGPT_API_KEY`
2. `.env` file — walk up from `$PWD` to `/` looking for a file containing `CUSTOMGPT_API_KEY=`
3. Read `~/.claude/customgpt-config.json` and extract the `apiKey` field

Use the first non-empty value as `$API_KEY`. If none found, ask the user for it.

---

## Step 2 — Read Meta File

Walk up from `$PWD` to find `.customgpt-meta.json`. Extract `agent_id`.

If not found:
> "No agent found in this directory tree. Run `/create-agent` first."

---

## Step 3 — Fetch Project Info

```bash
curl -s --request GET \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "accept: application/json"
```

Extract: `data.project_name`, `data.is_chat_active`, `data.type`, `data.created_at`

---

## Step 4 — Fetch Stats

```bash
curl -s --request GET \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/stats" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "accept: application/json"
```

Extract: `data.pages_found`, `data.pages_crawled`, `data.pages_indexed`, `data.total_words_indexed`, `data.crawl_credits_used`, `data.query_credits_used`, `data.total_queries`, `data.total_storage_credits_used`

---

## Step 5 — Report

Present a clean summary:

> **Agent: {project_name}** (ID: {agent_id})
>
> | Field | Value |
> |---|---|
> | Chat active | {is_chat_active} |
> | Type | {type} |
> | Created | {created_at} |
>
> **Indexing Stats:**
>
> | Metric | Count |
> |---|---|
> | Pages found | {pages_found} |
> | Pages crawled | {pages_crawled} |
> | Pages indexed | {pages_indexed} |
> | Words indexed | {total_words_indexed} |
> | Crawl credits used | {crawl_credits_used} |
> | Query credits used | {query_credits_used} |
> | Total queries | {total_queries} |
> | Storage credits used | {total_storage_credits_used} |

If `pages_crawled < pages_found`, add:
> "Note: {pages_found - pages_crawled} page(s) not yet crawled. Indexing may still be in progress — run again in a moment or use `/refresh-agent` to force a re-sync."

If `is_chat_active` is false, add:
> "Note: Chat is disabled for this agent."
