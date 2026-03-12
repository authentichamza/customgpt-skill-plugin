# query-agent

Ask a plain-language question to the indexed project. Returns an AI answer with source citations.

## Critical Rules
- ALWAYS read `.customgpt-meta.json` to get the `agent_id`
- Each call starts a fresh conversation (no session reuse)
- Present the answer AND the sources clearly

---

## Step 1 — Get API Key

Check in order:
1. Env var `$CUSTOMGPT_API_KEY`
2. `.env` file — walk up from `$PWD` to `/` looking for a file containing `CUSTOMGPT_API_KEY=`
3. Read `~/.claude/customgpt-config.json` and extract the `apiKey` field

Use the first non-empty value. If none found, ask the user then save:
```bash
echo '{"apiKey":"KEY_HERE"}' > ~/.claude/customgpt-config.json
```

---

## Step 2 — Read Meta File

Walk up from `$PWD` to find `.customgpt-meta.json`. Extract `agent_id` and `indexed_folder`.

If not found:
> "No agent found in this directory tree. Run `/create-agent` first."

---

## Step 3 — Check Freshness (Informational)

Check if any files in `indexed_folder` are newer than `.customgpt-meta.json`. If so, inform the user:
> "Note: {N} files have changed since the last index. Run `/refresh-agent` for up-to-date results."

Then continue with the query regardless.

---

## Step 4 — Get the Question

If the user provided a question, use it. Otherwise ask:
> "What would you like to know about the indexed project?"

---

## Step 5 — Create a Conversation Session

```bash
curl -s -X POST "https://app.customgpt.ai/api/v1/projects/$AGENT_ID/conversations" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "name=Claude+Code+Query"
```

Extract `data.session_id`.

---

## Step 6 — Send the Question

```bash
curl -s -X POST "https://app.customgpt.ai/api/v1/projects/$AGENT_ID/conversations/$SESSION_ID/messages" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "prompt=$QUESTION" \
  -d "stream=false"
```

Extract `data.openai_response` (the answer) and `data.citations` (sources).

---

## Step 7 — Present Results

**Answer:**
{openai_response}

**Sources:**
- {citation title} — {citation url or filename}

Use the answer to help with the current task. If it references code, read those files for full context before making changes.
