---
description: Create a new CustomGPT.ai agent for a local folder and upload its files into the knowledge base.
triggers:
  - "create agent"
  - "create a new agent"
  - "new agent"
  - "setup agent"
  - "set up agent"
  - "initialize agent"
  - "create customgpt"
  - "new customgpt agent"
---

# create-agent

Create a CustomGPT.ai agent for a local folder and index its files into it.

## Critical Rules
- NEVER skip the API key check
- ALWAYS save `.customgpt-meta.json` before reporting success
- NEVER index `.git/`, `node_modules/`, `__pycache__/`, binary files, or `.env` files
- Use `find` to collect eligible files, then upload each one with curl — Claude handles the iteration

---

## Step 1 — Get API Key

Check in order:
1. Env var `$CUSTOMGPT_API_KEY`
2. `.env` file — walk up from `$PWD` to `/` looking for a file containing `CUSTOMGPT_API_KEY=`
3. Read `~/.claude/customgpt-config.json` and extract the `apiKey` field

Use the first non-empty value. If none found, ask:
> "Please provide your CustomGPT.ai API key. You can find it at https://app.customgpt.ai/profile#api-keys"

Once you have the key, save it:
```bash
echo '{"apiKey":"KEY_HERE"}' > ~/.claude/customgpt-config.json
```

---

## Step 2 — Get Folder to Index

Ask the user which folder to index, or use `$PWD` if they don't specify. Confirm it exists.

---

## Step 3 — Create the Agent

```bash
curl -s --request POST \
  --url "https://app.customgpt.ai/api/v1/projects" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "Content-Type: application/x-www-form-urlencoded" \
  --data "project_name=$(basename $FOLDER)&is_chat_active=1"
```

Read the response and extract `data.id` as `agent_id`. If missing, show the full response and stop.

---

## Step 4 — Collect Eligible Files

```bash
find "$FOLDER" -type f \
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

Read the list of files. Tell the user the count before uploading.

---

## Step 5 — Upload Each File

For each file from Step 4, compute `REL` as the path relative to `$FOLDER`, then run:

```bash
curl -s --request POST \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/sources" \
  --header "Authorization: Bearer ${API_KEY}" \
  --form "file=@${ABSOLUTE_PATH};filename=${REL}"
```

Read the HTTP status. HTTP 200 or 201 = success. Report each result as you go: `OK: {REL}` or `FAILED [{status}]: {REL}`.

---

## Step 6 — Save Meta File

```bash
cat > "${FOLDER}/.customgpt-meta.json" << EOF
{
  "agent_id": ${AGENT_ID},
  "agent_name": "$(basename $FOLDER)",
  "indexed_folder": "${FOLDER}",
  "created_at": "$(date -u +%Y-%m-%dT%H:%M:%SZ)"
}
EOF
```

---

## Step 7 — Report

> "Agent created. ID: {agent_id} | Folder: {folder} | Uploaded: {N} / {total}, Failed: {F}
> Use `/query-agent` to search it or `/refresh-agent` to re-sync after changes."
