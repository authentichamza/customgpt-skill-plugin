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

```bash
# 1. Already-exported env var
echo "${CUSTOMGPT_API_KEY:-}"

# 2. .env file in current or parent directories
dir="$PWD"
while [ "$dir" != "/" ]; do
  if [ -f "$dir/.env" ]; then
    grep -E '^export\s+CUSTOMGPT_API_KEY=|^CUSTOMGPT_API_KEY=' "$dir/.env" \
      | sed 's/^export\s*//' | sed 's/CUSTOMGPT_API_KEY=//' | tr -d '"'"'" | head -1
    break
  fi
  dir=$(dirname "$dir")
done

# 3. Saved config file
cat ~/.claude/customgpt-config.json 2>/dev/null
```

Use the first non-empty value as `$API_KEY`. If missing, ask the user for it.

---

## Step 2 — Read Meta File

```bash
dir="$PWD"
while [ "$dir" != "/" ]; do
  [ -f "$dir/.customgpt-meta.json" ] && cat "$dir/.customgpt-meta.json" && break
  dir=$(dirname "$dir")
done
```

Extract `agent_id`. If not found:
> "No agent found in this directory tree. Run `/create-agent` first."

---

## Step 3 — Fetch Project Info

```bash
curl -s --request GET \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "accept: application/json"
```

From the response extract:
- `data.project_name`
- `data.is_chat_active` — whether the chat bot is enabled
- `data.type` — e.g. `SITEMAP`, `FILE`
- `data.created_at`

---

## Step 4 — Fetch Stats

```bash
curl -s --request GET \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/stats" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "accept: application/json"
```

From the response extract:
- `data.pages_found`
- `data.pages_crawled`
- `data.pages_indexed`
- `data.total_words_indexed`
- `data.crawl_credits_used`
- `data.query_credits_used`
- `data.total_queries`
- `data.total_storage_credits_used`

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
