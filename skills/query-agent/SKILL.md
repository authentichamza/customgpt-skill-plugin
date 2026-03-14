---
name: customgpt-ai-rag:query-agent
description: Ask a plain-language question to the indexed CustomGPT.ai agent and get an AI-generated answer with source citations. Warns if the index may be stale.
argument-hint: "[your question about the indexed project]"
allowed-tools: Bash, Read
triggers:
  - "query agent"
  - "ask agent"
  - "query the agent"
  - "ask the agent"
  - "search agent"
  - "query customgpt"
  - "ask customgpt"
  - "search the index"
  - "search the knowledge base"
  - "search indexed files"
  - "what does the index say"
  - "what does the agent know"
  - "find in the index"
  - "look up in the index"
  - "search my codebase"
  - "search my project"
  - "search my docs"
  - "what do my files say about"
  - "find across my files"
---

# query-agent

Ask a plain-language question to the indexed project. Returns an AI-generated answer grounded in your files, with source citations.

## Rules

- ALWAYS read `.customgpt-meta.json` to get `agent_id` — never hardcode it
- Each call creates a fresh conversation session (stateless)
- ALWAYS present both the answer AND the sources — sources let the user verify accuracy
- If the agent has no indexed pages yet, instruct the user to run `/check-status` first

---

## Step 1 — Resolve API Key

Follow the lookup procedure in `skills/_shared/api-key.md`. Store the result as `$API_KEY`.

---

## Step 2 — Read Meta File

Walk up from `$PWD` toward `/`, looking for `.customgpt-meta.json`. Extract `agent_id` and `indexed_folder`.

If not found:
> "No agent found in this directory tree. Run `/create-agent` to set one up first."

---

## Step 3 — Freshness Warning (Optional)

Check whether any files in `indexed_folder` were modified after the meta file's timestamp:

```bash
find "$indexed_folder" -newer "${META_FILE_PATH}" -type f -not -path "*/.git/*" -not -path "*/node_modules/*"
```

If any files are newer, inform the user before proceeding:
> "Note: {N} file(s) have changed since the last index update. Results may not reflect the latest changes. Run `/refresh-agent` to re-sync, or continue with the current index."

Then continue regardless — do not stop.

---

## Step 4 — Get the Question

If the user provided a question with the command, use it directly.

Otherwise ask:
> "What would you like to search for in the indexed project?"

---

## Step 5 — Create a Conversation Session

```bash
curl -s --request POST \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/conversations" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "Content-Type: application/x-www-form-urlencoded" \
  --data "name=Claude+Code+Query"
```

Read the response and extract `data.session_id` as `$SESSION_ID`.

If the response indicates the agent has no content or is not ready, tell the user:
> "The agent has no indexed content yet. Run `/check-status` to see indexing progress."

---

## Step 6 — Send the Question

```bash
curl -s --request POST \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/conversations/${SESSION_ID}/messages" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "prompt=${QUESTION}" \
  --data "stream=false"
```

Read the response and extract:
- `data.openai_response` — the AI-generated answer
- `data.citations` — array of source references (each has `title`, `url` or `filename`, and optionally `page_id`)

If `openai_response` is empty or null, the agent may still be indexing. Tell the user:
> "The agent returned an empty response. The index may still be processing — run `/check-status` to verify, then try again."

---

## Step 7 — Present Results

Format the output clearly:

---

**Answer**

{openai_response}

**Sources**

| # | Source | Details |
|---|--------|---------|
| 1 | {title or filename} | {url or "uploaded file"} |
| 2 | ... | ... |

---

If citations is empty or null, add:
> "No specific sources cited. The agent may be using general knowledge rather than indexed content — check `/check-status` to confirm the index is ready."

**After displaying results**, consider whether any cited files are relevant to the current task. If so, use the Read tool to open them for full context before making code changes.
