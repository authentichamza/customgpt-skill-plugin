---
name: customgpt-ai-rag:check-status
description: Show the current status of the CustomGPT.ai agent — project metadata, chat availability, and document crawl/index statistics. Use to verify the index is ready before querying.
argument-hint: "[optional: agent ID — defaults to current project's agent]"
allowed-tools: Bash, Read
triggers:
  - "check status"
  - "agent status"
  - "index status"
  - "check agent"
  - "check agent status"
  - "how many pages"
  - "how many files indexed"
  - "pages indexed"
  - "pages crawled"
  - "indexing progress"
  - "check index"
  - "is the index ready"
  - "is indexing done"
  - "status of agent"
  - "status of the agent"
  - "how is the index"
---

# check-status

Show the current status of a CustomGPT.ai agent: project metadata, chat availability, and document crawl/index statistics. Run this after `/create-agent` or `/refresh-agent` to know when the index is ready to query.

## Rules

- ALWAYS read `.customgpt-meta.json` to get `agent_id` — never hardcode it
- Make two API calls: one for project info, one for stats
- Present a clean summary — do NOT dump raw JSON

---

## Step 1 — Resolve API Key

Follow the lookup procedure in `skills/_shared/api-key.md`. Store the result as `$API_KEY`.

---

## Step 2 — Read Meta File

Walk up from `$PWD` toward `/`, looking for `.customgpt-meta.json`. Extract `agent_id` and `indexed_folder`.

If not found:
> "No agent found in this directory tree. Run `/create-agent` to set one up first."

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

## Step 4 — Fetch Indexing Stats

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

> **Agent: `{project_name}`** (ID: `{agent_id}`)
> Indexed folder: `{indexed_folder}`
>
> | Setting | Value |
> |---|---|
> | Chat active | {is_chat_active} |
> | Type | {type} |
> | Created | {created_at} |
>
> **Indexing Progress**
>
> | Metric | Count |
> |---|---|
> | Documents found | {pages_found} |
> | Documents crawled | {pages_crawled} |
> | Documents indexed | {pages_indexed} |
> | Words indexed | {total_words_indexed} |
> | Total queries run | {total_queries} |
> | Crawl credits used | {crawl_credits_used} |
> | Query credits used | {query_credits_used} |
> | Storage credits used | {total_storage_credits_used} |

**Status interpretation — add the relevant note:**

- If `pages_indexed == pages_found` and `pages_found > 0`:
  > "Index is fully ready. Use `/query-agent` to start searching."

- If `pages_indexed < pages_found` and `pages_crawled < pages_found`:
  > "Indexing in progress ({pages_indexed}/{pages_found} complete). Check again in a moment, or run `/list-pages` to see which documents are still queued."

- If `pages_crawled == pages_found` and `pages_indexed < pages_found`:
  > "All documents crawled; {pages_found - pages_indexed} still being indexed. Check again in a moment."

- If `pages_found == 0`:
  > "No documents found. Run `/index-files` to upload files to this agent."

- If `is_chat_active` is false:
  > "Warning: Chat is disabled for this agent. Queries will not work until chat is enabled."
