# Noa API documentation

We’re currently providing only MCP Server API (alpha version). This API endpoint can be plugged into AI code assistants, or invoked via HTTP directly.

For using Noa via API, you need to provide a GitHub token.

# Provided tools

## `start_repo_audit`

Launch a security audit for a GitHub repository ref. Requires owner, repo, ref, and a GitHub token with repo access.

### Arguments

| Parameter | type   | required | description |
| --------- | ------ | -------- | ----------- |
| `owner`   | string | required | GitHub repository owner (user or organization). |
| `repo`    | string | required | GitHub repository name. |
| `ref`     | string | required | Git reference (branch, tag, or commit SHA). |
| `githubToken` | string | required | GitHub personal access token with `repo` scope. |

### Output schema

| Return value | type   | description |
| ------------- | ------ | ----------- |
| `auditJobId`  | string | Audit job UUID. |
| `owner`       | string | GitHub repository owner. |
| `repo`        | string | GitHub repository name. |
| `ref`         | string | Git reference audited. |


## `get_repo_audit`

Retrieve the latest status and results for a previously started repository audit job.

### Arguments

| Parameter | type   | required | description |
| --------- | ------ | -------- | ----------- |
| `auditJobId` | string | required | Audit job UUID returned from `start_repo_audit`. |

### Output schema

| Return value | type   | description |
| ------------- | ------ | ----------- |
| `auditJobId`  | string | Audit job UUID. |
| `status`      | string | Current audit status (`pending`, `running`, `completed`, `failed`, or `interrupted`). |
| `analysis`    | string | Final audit summary (present when status is `completed`). |
| `errorMessage` | string | Failure details (present when status is `failed` or `interrupted`). |

## `list_repo_audits`

List audit jobs for a GitHub repository. 

| Parameter | type   | required | description |
| --------- | ------ | -------- | ----------- |
| `owner`   | string | required | GitHub repository owner. |
| `repo`    | string | required | GitHub repository name. |
| `githubToken` | string | required | GitHub personal access token with `repo` scope. |

### Output schema

| Return value | type   | description |
| ------------- | ------ | ----------- |
| `owner`       | string | GitHub repository owner. |
| `repo`        | string | GitHub repository name. |
| `jobs`        | array  | List of audit job summaries. Each entry includes `jobId`, `status`, `ref`, `createdAt`, and optionally `completedAt`, `analysisResultId`, and `errorMessage`. |

# GitHub token

The first way is to issue a GitHub token directly via GitHub. These tokens can be scoped to particular repos. Noa needs repo_read permissions. Here are up-to-date links for issuing a GitHub token (also known as personal access tokens):
- Documentation: https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens
- Direct link for issuing a new token: https://github.com/settings/personal-access-tokens/new
- Direct link for listing issued tokens and revoking issued tokens if needed: https://github.com/settings/tokens

The second way of sourcing a GitHub token is asking the code assistant to pass its own GitHub token to the Noa tools.

# Configuring Noa for Devin

- Go to MCP Marketplace page: https://app.devin.ai/settings/mcp-marketplace and click on “Add Your Own” (or directly go to https://app.devin.ai/settings/mcp-marketplace/setup/custom )

- MCP server options:
  - `Server name`: `noa-mcp`
  - `Transport Type`: HTTP
  - `Server URL`:
  - If you’d like to use your custom GitHub token, provide it via `Secrets -> Add a new secret`:
    - `Secret Name`: `GITHUB_TOKEN`
    - `Secret Value`: `gh_...`
    - Click on `Save secret`
    - Expand `Custom Headers (optional)`
    - `Header name`: `x-github-token`
    - `Header value`: `$GITHUB_TOKEN`
  - Click on `Enable`

# Using Noa from a Devin session using Devin's GitHub token
- Start a new Devin session
- Adjust the repo and branch, and try the prompt `Use noa-mcp MCP server, invoke tool "start_repo_audit" using the Devin's GitHub token for repo "noa" and branch "main". Do not analyze the repo contents for invoking the prompt.`
- Re-run `get_repo_audit` with the returned `auditJobId` to track progress or collect the final summary.

# Using Noa via HTTP

All MCP HTTP calls go to `https://api.noa.logicalintelligence.com/api/mcp`. Each call is stateless—send one JSON-RPC payload per request.

### 1. Start an audit

```bash
GITHUB_TOKEN="ghp_yourgithubtoken"

curl -sS \
  -X POST "https://api.noa.logicalintelligence.com/api/mcp" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  --data '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
      "name": "start_repo_audit",
      "arguments": {
        "owner": "logical-intelligence",
        "repo": "noa-docs",
        "ref": "main",
        "githubToken": "'"${GITHUB_TOKEN}"'"
      }
    }
  }'

# Example output:
# event: message
# data: {"result":{"structuredContent":{"auditJobId":"...","owner":"logical-intelligence","repo":"noa-docs","ref":"main"},"content":[{"type":"text","text":"Started repository audit for logical-intelligence/noa-docs@main.\nJob ID: ...\nTrack progress with `get_repo_audit` using this job ID."}]},"jsonrpc":"2.0","id":1}

# Copy the structuredContent.auditJobId from the response body for later requests.
```

The response body includes the structured job payload (`auditJobId`, `owner`, `repo`, `ref`). Copy the `structuredContent.auditJobId` value for subsequent requests.

### 2. Poll audit status

Use the `AUDIT_JOB_ID` copied from the previous step (replace the placeholder below).

```bash
AUDIT_JOB_ID="replace-with-audit-job-id"

curl -sS \
  -X POST "https://api.noa.logicalintelligence.com/api/mcp" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  --data '{
    "jsonrpc": "2.0",
    "id": 2,
    "method": "tools/call",
    "params": {
      "name": "get_repo_audit",
      "arguments": {
        "auditJobId": "'"${AUDIT_JOB_ID}"'"
      }
    }
  }'

# Example output:
# event: message
# data: {"result":{"structuredContent":{"auditJobId":"...","status":"running"},"content":[{"type":"text","text":"Audit job ...\nStatus: RUNNING\n\nThe audit is still processing. Re-run this tool to refresh the status."}]},"jsonrpc":"2.0","id":2}

#event: message
#data: {"result":{"structuredContent":{"auditJobId":"...","status":"completed","analysis":"..."},"content":"..."},"jsonrpc":"2.0","id":2}

```

The response reports the currently known status and (when complete) the audit summary.

### 3. List recent audits for a repository

```bash
curl -sS \
  -X POST "https://api.noa.logicalintelligence.com/api/mcp" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  --data '{
    "jsonrpc": "2.0",
    "id": 3,
    "method": "tools/call",
    "params": {
      "name": "list_repo_audits",
      "arguments": {
        "owner": "logical-intelligence",
        "repo": "noa-docs",
        "githubToken": "'"${GITHUB_TOKEN}"'"
      }
    }
  }'

# Example output:
#event: message
#data: {"result":{"structuredContent":{"owner":"logical-intelligence","repo":"noa-docs","jobs":[{"jobId":"...","status":"completed","ref":"main","createdAt":"2025-10-03T21:46:47.294Z","completedAt":"2025-10-03T21:58:20.345Z","analysisResultId":44}]},"content":[{"type":"text","text":"logical-intelligence/noa-docs\nTotal audits: 1\n\nJobs:\n1. ✓ COMPLETED - main\n   Job ID: ...\n   Created: 2025-10-03T21:46:47.294Z\n   Completed: 2025-10-03T21:58:20.345Z\n\nUse `get_repo_audit` with a job ID to retrieve detailed status."}]},"jsonrpc":"2.0","id":3}

```

Each entry in the returned `jobs` array includes the job ID, status, target ref, timestamps, and any error details.
