---
description: Permanently delete a CustomGPT.ai agent and remove its local meta file.
triggers:
  - "delete agent"
  - "remove agent"
  - "destroy agent"
  - "delete customgpt agent"
  - "remove customgpt"
  - "delete the agent"
---

# delete-agent

Permanently delete a CustomGPT.ai agent and remove its local meta file.

## Critical Rules
- ALWAYS confirm with the user before deleting — this is irreversible
- Remove `.customgpt-meta.json` only after successful deletion
- If the API call fails, do NOT remove the meta file

---

## Step 1 — Get API Key

Check in order:
1. Env var `$CUSTOMGPT_API_KEY`
2. `.env` file — walk up from `$PWD` to `/` looking for a file containing `CUSTOMGPT_API_KEY=`
3. Read `~/.claude/customgpt-config.json` and extract the `apiKey` field

Use the first non-empty value. If none found, ask the user.

---

## Step 2 — Read Meta File

Walk up from `$PWD` to find `.customgpt-meta.json`. Extract `agent_id`, `agent_name`, and `indexed_folder`.

If not found:
> "No agent found in this directory tree. Nothing to delete."

---

## Step 3 — Confirm with User

> "Are you sure you want to permanently delete agent '{agent_name}' (ID: {agent_id})? This cannot be undone. Type 'yes' to confirm."

If not confirmed:
> "Cancelled. Agent was not deleted."

---

## Step 4 — Delete the Agent

```bash
curl -s --request DELETE \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "accept: application/json"
```

Read the response. On failure, show the error and stop — do NOT remove the meta file.

---

## Step 5 — Remove Meta File

```bash
rm "${META_FILE_PATH}"
```

Where `META_FILE_PATH` is the full path to the `.customgpt-meta.json` found in Step 2.

---

## Step 6 — Report

> "Agent '{agent_name}' (ID: {agent_id}) deleted. Local meta file removed."
