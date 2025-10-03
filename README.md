# Noa is our bug search agent

We’re currently providing only MCP Server API (alpha version). This API endpoint can be plugged into AI code assistants, or invoked via HTTP directly.

For using Noa via API, you need to provide a GitHub token.

### GitHub token

The first way is to issue a GitHub token directly via GitHub. These tokens can be scoped to particular repos. Noa needs repo_read permissions. Here are up-to-date links for issuing a GitHub token (also known as personal access tokens):
- Documentation: https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens
- Direct link for issuing a new token: https://github.com/settings/personal-access-tokens/new
- Direct link for listing issued tokens and revoking issued tokens if needed: https://github.com/settings/tokens

The second way of sourcing a GitHub token is asking the code assistant to pass its own GitHub token to the Noa tools.

# Provided tools

## `start_repo_audit`

Enqueue audit of a GitHub repository with optional branch/ref and github token for private repos.

### Arguments

| Parameter| type  | required | description|
| -------- |-------|----------|------------|
| `owner`  |string |required|github repo owner|
| `repo`   |string |required|github repo name|
| `ref`    |string |required|github repo ref|
| `githubToken`| string |required|github token|

### Output schema

| Return value| type  | description|
| -------- |-------|------------|
| `jobAuditId`|string |job audit id|
| `owner`  |string |github repo owner|
| `repo`   |string |github repo name|
| `ref`    |string |github repo ref|


## `poll_repo_audit_result`

Fetch audit results.

### Arguments

| Return value| type  | description|
| -------- |-------|------------|
| `jobAuditId`|string |job audit id|

### Output schema

## `list_repo_audits`

List audit jobs for a GitHub repository. 

| Parameter| type  | required | description|
| -------- |-------|----------|------------|
| `owner`  |string |required|github repo owner|
| `repo`   |string |required|github repo name|

### Output schema

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
- It will automatically poll for audit results
“”

# Using Noa via HTTP:

- Check that API is online and responds:

- Check that your GitHub token is suitable and has access to the requested repo:

- Launch audit:
  - `curl -i -X POST https://$VERCEL_BASEURL/api/mcp -H "Content-Type: application/json" -H "Accept: application/json, text/event-stream" -d '{ "jsonrpc": "2.0", "id": 2, "method": "tools/call", "params": {"name" : "log_headers", "arguments" : {}} }'`

