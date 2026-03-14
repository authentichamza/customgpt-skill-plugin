---
name: customgpt-ai-rag:create-agent
description: Create a new CustomGPT.ai agent for a local folder and upload all eligible files into its knowledge base. Saves agent binding to .customgpt-meta.json.
argument-hint: "[folder path — defaults to current directory]"
allowed-tools: Bash, Read
triggers:
  - "create agent"
  - "create a new agent"
  - "create customgpt agent"
  - "new agent"
  - "new customgpt"
  - "setup agent"
  - "set up agent"
  - "initialize agent"
  - "init agent"
  - "index this repo"
  - "index this project"
  - "index everything"
  - "build a rag"
  - "build rag index"
  - "create a rag"
  - "set up rag"
---

# create-agent

Create a CustomGPT.ai agent for a local folder and upload all eligible files into its knowledge base.

## Rules

- NEVER proceed without an API key
- ALWAYS save `.customgpt-meta.json` before reporting success
- NEVER upload `.git/`, `node_modules/`, secrets (`.env`, `.env.*`), or binary files
- If `.customgpt-meta.json` already exists in the target folder, warn the user before overwriting — they may want `/refresh-agent` instead

---

## Step 1 — Resolve API Key

Follow the lookup procedure in `skills/_shared/api-key.md`. Store the result as `$API_KEY`.

---

## Step 2 — Determine the Folder

If the user specified a path, resolve it to an absolute path and verify it exists:

```bash
realpath "$USER_PATH"
```

If no path was given, use the Git repository root if available, otherwise `$PWD`:

```bash
git rev-parse --show-toplevel 2>/dev/null || echo "$PWD"
```

Store the result as `$FOLDER`. Tell the user: "Creating agent for: `{FOLDER}`"

Check whether `.customgpt-meta.json` already exists in `$FOLDER`. If it does, read it and ask:

> "An agent already exists for this folder: '{agent_name}' (ID: {agent_id}). Run `/refresh-agent` to re-sync it, or type 'overwrite' to create a new agent and replace the meta file."

If they don't confirm overwrite, stop.

---

## Step 3 — Create the Agent

```bash
curl -s --request POST \
  --url "https://app.customgpt.ai/api/v1/projects" \
  --header "Authorization: Bearer ${API_KEY}" \
  --form "project_name=$(basename $FOLDER)" \
  --form "file_data_retension=true"
```

Read the response. On success (HTTP 201), extract `data.id` as `$AGENT_ID` and `data.project_name`.

On failure, show the full response and stop. Common errors:
- HTTP 401 → invalid API key (see `skills/_shared/api-key.md`)
- HTTP 400 → check that `project_name` is not empty

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
  -not -path "*/target/*" \
  -not -path "*/.turbo/*" \
  -not -path "*/.parcel-cache/*" \
  -not -name ".env" \
  -not -name ".env.*" \
  \( \
    -name "*.js" -o -name "*.ts" -o -name "*.tsx" -o -name "*.jsx" \
    -o -name "*.mjs" -o -name "*.cjs" \
    -o -name "*.py" -o -name "*.go" -o -name "*.rb" -o -name "*.java" \
    -o -name "*.cs" -o -name "*.cpp" -o -name "*.c" -o -name "*.h" -o -name "*.hpp" \
    -o -name "*.rs" -o -name "*.swift" -o -name "*.kt" -o -name "*.php" \
    -o -name "*.scala" -o -name "*.lua" -o -name "*.r" -o -name "*.R" \
    -o -name "*.sh" -o -name "*.bash" -o -name "*.zsh" -o -name "*.fish" \
    -o -name "*.html" -o -name "*.htm" -o -name "*.css" -o -name "*.scss" \
    -o -name "*.sass" -o -name "*.less" -o -name "*.svelte" -o -name "*.vue" \
    -o -name "*.sql" -o -name "*.graphql" -o -name "*.proto" \
    -o -name "*.json" -o -name "*.jsonc" -o -name "*.yaml" -o -name "*.yml" \
    -o -name "*.toml" -o -name "*.xml" -o -name "*.ini" -o -name "*.cfg" \
    -o -name "*.conf" -o -name "*.env.example" \
    -o -name "*.md" -o -name "*.mdx" -o -name "*.rst" -o -name "*.txt" \
    -o -name "*.csv" -o -name "*.tsv" -o -name "*.pdf" -o -name "*.docx" \
    -o -name "*.doc" -o -name "*.odt" -o -name "*.pptx" -o -name "*.xlsx" \
  \)
```

Read the file list. Report the count: "Found {N} eligible files. Uploading..."

---

## Step 5 — Upload Each File

For each file, compute `REL` as its path relative to `$FOLDER` (strip the leading `$FOLDER/` prefix). Upload:

```bash
curl -s --request POST \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/sources" \
  --header "Authorization: Bearer ${API_KEY}" \
  --form "file=@${ABSOLUTE_PATH};filename=${REL}"
```

HTTP 200 or 201 = success. Report progress as you go: ✓ `{REL}` or ✗ `{REL}` (HTTP {status}).

Collect totals: `$UPLOADED` (success) and `$FAILED` (non-200/201).

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

> **Agent created successfully.**
>
> - Agent: `{agent_name}` (ID: `{agent_id}`)
> - Folder: `{folder}`
> - Uploaded: **{UPLOADED}** / {total} files{FAILED > 0 ? ", Failed: {FAILED}" : ""}
>
> CustomGPT.ai is now indexing your files. Run `/check-status` in a moment to see indexing progress, then `/query-agent` to start searching.

If any uploads failed, add:
> "Some files failed to upload. Run `/index-files` to retry specific files, or `/refresh-agent` to re-sync everything."
