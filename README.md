# tangled-mcp

MCP server for [Tangled](https://tangled.org) - a git collaboration platform built on AT Protocol.

reads go through [bobbin](https://docs.tangled.org/bobbin.html), tangled's public XRPC API (`api.tangled.org`) — **no credentials needed**. writes (issues, comments, labels) are atproto records put directly on your PDS and require an app password.

> **note**: this repository is mirrored to [GitHub](https://github.com/zzstoatzz/tangled-mcp) for deployment via [FastMCP Cloud](https://fastmcp.cloud).

## hosted server

a hosted instance runs at **`https://nate-tangled-mcp.fastmcp.app/mcp`** — no install needed:

```bash
claude mcp add --transport http tangled https://nate-tangled-mcp.fastmcp.app/mcp
```

for write access, pass credentials per request via headers:

```bash
claude mcp add --transport http tangled https://nate-tangled-mcp.fastmcp.app/mcp \
  --header "x-tangled-handle: your.handle" \
  --header "x-tangled-password: your-app-password"
```

your PDS is auto-discovered from your handle — self-hosted PDS works with no extra config.

## installation

```bash
git clone https://tangled.org/zzstoatzz/tangled-mcp
cd tangled-mcp
just setup
```

> [!IMPORTANT]
> requires [`uv`](https://docs.astral.sh/uv/) and [`just`](https://github.com/casey/just)

## configuration

credentials are optional — only write tools need them. hosted/multi-tenant deployments can send them per request via `x-tangled-handle` / `x-tangled-password` headers, which take precedence over env. for local use, create `.env`:

```bash
TANGLED_HANDLE=your.handle
TANGLED_PASSWORD=your-app-password
```

## usage

<details>
<summary>MCP client installation instructions</summary>

### claude code

```bash
# read-only (no credentials)
claude mcp add tangled -- uvx tangled-mcp

# with write access
claude mcp add tangled \
  -e TANGLED_HANDLE=your.handle \
  -e TANGLED_PASSWORD=your-app-password \
  -- uvx tangled-mcp
```

### cursor

add to your cursor settings (`~/.cursor/mcp.json` or `.cursor/mcp.json`):

```json
{
  "mcpServers": {
    "tangled": {
      "command": "uvx",
      "args": ["tangled-mcp"],
      "env": {
        "TANGLED_HANDLE": "your.handle",
        "TANGLED_PASSWORD": "your-app-password"
      }
    }
  }
}
```

### codex cli

```bash
codex mcp add tangled \
  --env TANGLED_HANDLE=your.handle \
  --env TANGLED_PASSWORD=your-app-password \
  -- uvx tangled-mcp
```

### other clients

for clients that support MCP server configuration, use:
- **command**: `uvx`
- **args**: `["tangled-mcp"]`
- **environment variables** (optional, for writes): `TANGLED_HANDLE`, `TANGLED_PASSWORD`

</details>

### development usage

```bash
uv run tangled-mcp
```

## tools

repositories are `owner/repo` (e.g. `zzstoatzz.io/tangled-mcp`); handles (with or without `@`) and DIDs both work for the owner. issues are identified by at-uri.

### discovery (no auth)
- `search(query, limit)` - full-text search across repos, issues, and strings
- `list_repos(owner, limit)` - list a user's repositories
- `get_repo(repo)` - metadata: knot, default branch, languages, labels
- `get_record(uri)` - fetch the full record behind any at-uri (strings/pastes, comments, ...)

### git (no auth)
- `list_branches(repo, limit)` / `list_tags(repo, limit)`
- `list_files(repo, path, ref)` - browse the tree
- `read_file(repo, path, ref)` - file contents
- `commit_log(repo, ref, limit)` - recent commits
- `compare(repo, rev1, rev2)` - diff two revisions

### issues & pulls (no auth)
- `list_issues(repo, state, limit)` - filterable by open/closed
- `get_issue(issue)`
- `list_pulls(repo, status, limit)` - filterable by open/closed/merged
- `get_pull(pull)` - single PR with live state (derived from PDS status records, no index lag)
- `list_pipelines(repo, limit)` - CI pipeline runs

### writes (require credentials)
- `create_pull(repo, title, patch | edits, target_branch, body)` - open a PR from `git format-patch` output, or from whole-file `edits` (no clone needed — the server synthesizes the patch)
- `create_issue(repo, title, body, labels)`
- `update_issue(issue, title, body)`
- `set_issue_state(issue, state)` - close/reopen
- `set_pull_state(pull, state)` - close/reopen a PR
- `comment_on_issue(issue, body)`
- `delete_issue(issue)`

## ChatGPT OAuth / OpenID integration plan

### objective

Replace the hosted `x-tangled-handle` / `x-tangled-password` write-authentication path with a ChatGPT-friendly OAuth flow. Keep public repository reads anonymous, and require account connection only for mutation tools.

Target flow:

```text
ChatGPT
   |
   | OAuth 2.1 access token
   v
tangled-mcp
   |
   | server-side AT Protocol OAuth session
   v
user PDS
   |
   v
Tangled records / blobs
```

ChatGPT should never receive or store a Tangled app password.

### 1. make tangled-mcp an OAuth-protected MCP resource

For hosted HTTP use, authenticate with a standard bearer token:

```http
Authorization: Bearer <mcp-access-token>
```

Add MCP protected-resource metadata, validate issuer, expiry, audience/resource, and scopes, and attach the authenticated principal to request context.

Suggested scopes:

```text
tangled:read
tangled:write
```

Public read tools can remain anonymous.

### 2. add an OAuth authorization server / broker

Expose the ChatGPT-facing OAuth endpoints, including:

```text
/.well-known/oauth-protected-resource
/.well-known/oauth-authorization-server
/authorize
/token
```

The broker links an MCP principal to an AT Protocol identity and persisted OAuth session.

### 3. link accounts by Tangled handle

During authorization:

1. collect the user's Tangled/AT Protocol handle
2. resolve handle -> DID
3. resolve DID -> PDS
4. discover the PDS authorization server
5. begin AT Protocol OAuth

The hosted flow must not request an app password.

### 4. replace hosted createSession() with AT Protocol OAuth

The current write path uses `com.atproto.server.createSession` with handle and app password. Replace that hosted path with AT Protocol OAuth Authorization Code + PKCE using a maintained AT Protocol OAuth SDK.

The SDK should handle the AT Protocol OAuth requirements, including PAR, DPoP, token refresh, and distributed authorization-server discovery.

### 5. publish AT Protocol OAuth client metadata

Expose public client metadata, for example:

```text
/oauth-client-metadata.json
```

It should declare:

- public client ID URL
- client name
- redirect URI
- authorization-code grant
- refresh-token grant
- requested AT Protocol scopes

Use a broad compatibility scope initially if necessary to support the existing record and blob operations, then narrow permissions as Tangled-specific scopes become available.

### 6. persist linked AT Protocol sessions

Store encrypted OAuth state server-side. Minimum account-link fields:

```text
mcp_subject
atproto_did
handle
pds_url
authorization_issuer
access_token_encrypted
refresh_token_encrypted
dpop_private_key_encrypted
scope
expires_at
created_at
updated_at
```

Use the DID as the durable AT Protocol identity because handles can change.

### 7. refactor records.login()

Preserve the existing write-tool call shape where possible:

```python
session = await records.login()
```

but change hosted behavior to:

```text
current MCP principal
        |
        v
load linked AT Protocol session
        |
        v
refresh if necessary
        |
        v
return authenticated Session
```

This keeps the existing issue, pull, comment, blob, and record-writing logic largely unchanged.

### 8. support refresh on both layers

There are two independent token lifecycles:

```text
ChatGPT <-> tangled-mcp

tangled-mcp <-> user PDS
```

The ChatGPT-facing OAuth server should issue refresh tokens, and the broker must independently refresh the user's AT Protocol session.

### 9. preserve anonymous reads

Keep the existing public tools usable without login where possible:

```text
search
list_repos
get_repo
get_record
list_branches
list_tags
list_files
read_file
commit_log
compare
list_issues
get_issue
list_pulls
get_pull
list_pipelines
get_pull_patch
get_pull_file
```

Require OAuth for mutations:

```text
create_issue
create_pull
update_issue
set_issue_state
update_pull
set_pull_state
comment_on_issue
comment_on_pull
delete_issue
```

### 10. add MCP safety metadata

Mark read-only tools as read-only and mutations as writes. Mark destructive operations such as delete as destructive where supported so ChatGPT can apply appropriate permission and confirmation behavior.

### 11. expected source changes

Likely structure:

```text
src/tangled_mcp/
    server.py
    records.py
    settings.py
    oauth_routes.py

    auth/
        __init__.py
        mcp_oauth.py
        atproto_oauth.py
        sessions.py
        store.py
```

#### records.py

- retire hosted `x-tangled-handle` / `x-tangled-password`
- retire hosted `createSession()` login
- resolve current MCP principal
- load and refresh linked AT Protocol OAuth sessions
- optionally preserve app-password login for local CLI compatibility

#### settings.py

Add hosted OAuth configuration such as:

```text
PUBLIC_BASE_URL
OAUTH_ISSUER
SESSION_ENCRYPTION_KEY
DATABASE_URL
```

The hosted OAuth path should not require `TANGLED_PASSWORD`.

#### server.py

- expose authenticated principal to write tools
- add per-tool authorization metadata
- add read/write/destructive safety annotations
- preserve anonymous public reads

### 12. implementation stages

#### phase 1: MCP OAuth shell

- protected-resource metadata
- authorization-server metadata
- bearer validation
- MCP principal identity
- anonymous reads preserved

#### phase 2: AT Protocol account linking

- handle -> DID -> PDS discovery
- AT Protocol OAuth via SDK
- encrypted session persistence
- refresh support

#### phase 3: connect write tools

- refactor `records.login()`
- require `tangled:write` on mutations
- preserve existing Tangled record-writing behavior

#### phase 4: ChatGPT hardening

- add tool safety annotations
- verify reconnect and refresh behavior
- verify self-hosted PDS accounts
- remove raw credential headers from hosted deployment
- document local/CLI legacy authentication separately

### implementation constraint

Do not hand-roll AT Protocol OAuth. It includes distributed authorization-server discovery, PKCE, PAR, DPoP, refresh state, and persistent session requirements. Use a maintained AT Protocol OAuth SDK.

## development

```bash
just test   # run tests
just check  # run pre-commit checks
```

see [docs/bobbin-api.md](docs/bobbin-api.md) for notes on tangled's public API.

---

mcp-name: io.github.zzstoatzz/tangled-mcp
