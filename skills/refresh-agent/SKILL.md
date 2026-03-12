---
description: Delete all pages from the CustomGPT.ai agent and re-sync the entire indexed folder from scratch.
triggers:
  - "reindex"
  - "re-index"
  - "reindex skills"
  - "refresh agent"
  - "refresh the agent"
  - "re-sync"
  - "resync"
  - "full reindex"
  - "full re-index"
---

# refresh-agent

Delete all documents from the agent and re-upload everything from the indexed folder. Use after making changes to the codebase.

## Critical Rules
- ALWAYS read `.customgpt-meta.json` to get `agent_id` and `indexed_folder`
- Delete ALL pages before re-uploading — do not skip deletion
- Paginate the fetch loop — there may be more than 100 pages; read `data.pages.last_page` to know when to stop
- Touch the meta file after upload to reset the freshness timestamp

---

## Step 1 — Get API Key

Check in order:
1. Env var `$CUSTOMGPT_API_KEY`
2. `.env` file — walk up from `$PWD` to `/` looking for a file containing `CUSTOMGPT_API_KEY=`
3. Read `~/.claude/customgpt-config.json` and extract the `apiKey` field

Use the first non-empty value. If none found, ask the user.

---

## Step 2 — Read Meta File

Walk up from `$PWD` to find `.customgpt-meta.json`. Extract `agent_id` and `indexed_folder`.

If not found:
> "No agent found in this directory tree. Run `/create-agent` first."

Confirm with the user before proceeding:
> "I will delete all pages from agent {agent_id} and re-index {indexed_folder}. Continue? (yes/no)"

---

## Step 3 — Check for Changed Files (Informational)

```bash
find "$indexed_folder" -newer "${META_FILE_PATH}" -type f
```

Report the count of changed files to the user — informational only. The refresh always re-syncs everything.

---

## Step 4 — Fetch All Page IDs

Fetch page 1 to get the full list of document IDs:

```bash
curl -s --request GET \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/pages?page=1&limit=100&order=asc" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "accept: application/json"
```

Read `data.pages.data[]` and collect all `id` values. Read `data.pages.last_page` — if greater than 1, repeat for each remaining page:

```bash
curl -s --request GET \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/pages?page=${PAGE}&limit=100&order=asc" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "accept: application/json"
```

Collect all IDs across all pages before starting deletion.

---

## Step 5 — Delete Each Page

For each page ID collected in Step 4:

```bash
curl -s --request DELETE \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/pages/${PAGE_ID}" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "accept: application/json"
```

HTTP 200 = deleted. Count successes and failures. Report progress (e.g. "Deleted 12 / 45...").

---

## Step 6 — Collect Eligible Files

```bash
find "$indexed_folder" -type f \
  -not -path "*/.git/*" \
  -not -path "*/node_modules/*" \
  -not -path "*/__pycache__/*" \
  -not -path "*/.next/*" \
  -not -path "*/dist/*" \
  -not -path "*/build/*" \
  -not -path "*/.cache/*" \
  -not -path "*/vendor/*" \
  -not -path "*/coverage/*" \
  -not -path "*/.venv/*" \
  -not -path "*/venv/*" \
  \( \
    -name "*.js" -o -name "*.ts" -o -name "*.tsx" -o -name "*.jsx" \
    -o -name "*.py" -o -name "*.go" -o -name "*.rb" -o -name "*.java" \
    -o -name "*.cs" -o -name "*.cpp" -o -name "*.c" -o -name "*.h" \
    -o -name "*.rs" -o -name "*.swift" -o -name "*.kt" -o -name "*.php" \
    -o -name "*.sh" -o -name "*.bash" -o -name "*.zsh" \
    -o -name "*.html" -o -name "*.css" -o -name "*.scss" \
    -o -name "*.sql" -o -name "*.graphql" \
    -o -name "*.json" -o -name "*.yaml" -o -name "*.yml" -o -name "*.toml" \
    -o -name "*.xml" -o -name "*.ini" -o -name "*.cfg" -o -name "*.conf" \
    -o -name "*.md" -o -name "*.mdx" -o -name "*.rst" -o -name "*.txt" \
    -o -name "*.csv" -o -name "*.pdf" -o -name "*.docx" \
  \)
```

Tell the user the total file count before uploading.

---

## Step 7 — Upload Each File

For each file from Step 6, compute `REL` as its path relative to `indexed_folder`, then:

```bash
curl -s --request POST \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/sources" \
  --header "Authorization: Bearer ${API_KEY}" \
  --form "file=@${ABSOLUTE_PATH};filename=${REL}"
```

HTTP 200 or 201 = success. Report each result: `OK: {REL}` or `FAILED [{status}]: {REL}`.

---

## Step 8 — Reset Freshness Timestamp

```bash
touch "${META_FILE_PATH}"
```

---

## Step 9 — Report

> "Agent {agent_id} refreshed.
> - Deleted: {D} pages
> - Uploaded: {U} / {total} files, Failed: {F}
>
> Use `/query-agent` to search the updated index."
