# ChatGPT Migration Plan

This document captures the execution plan for migrating `tangled-mcp` from raw Tangled credential headers to a ChatGPT-friendly OAuth flow.

## Current state

- `main` remains the current working Tangled MCP implementation.
- `openid` is the migration branch.
- The OAuth/OpenID architecture has been planned, but implementation has not started yet.
- Public Tangled reads already work without authentication.
- Tangled writes still use the current app-password flow through `x-tangled-handle` / `x-tangled-password` or local environment variables.
- The existing Tangled read/write behavior, patch synthesis, pull rounds, record formats, and blob handling should remain intact during the migration.

## Migration goal

Replace the hosted raw-header authentication path with a ChatGPT-compatible OAuth flow while preserving anonymous public reads and existing Tangled functionality.

Target architecture:

```text
ChatGPT
   |
   | OAuth 2.1 access token
   v
tangled-mcp
   |
   | authenticated MCP principal
   | linked server-side AT Protocol OAuth session
   v
user PDS
   |
   v
Tangled records / blobs
```

ChatGPT must never receive or store a Tangled app password.

## Phase 1: MCP OAuth shell

This is the next implementation step.

Build the ChatGPT-facing OAuth front door without changing Tangled write behavior yet.

### Deliverables

1. Add MCP protected-resource metadata.
2. Add authorization-server metadata.
3. Add bearer-token validation for protected requests.
4. Establish an authenticated MCP principal in request context.
5. Preserve anonymous access for all existing public read tools.
6. Do not modify Tangled record-writing logic in this phase.

### Expected endpoints

```text
/.well-known/oauth-protected-resource
/.well-known/oauth-authorization-server
/authorize
/token
```

### Expected token behavior

Hosted protected requests use:

```http
Authorization: Bearer <mcp-access-token>
```

Validate:

- issuer
- expiration
- audience/resource
- scopes

Suggested MCP scopes:

```text
tangled:read
tangled:write
```

### Phase 1 completion criteria

- Public reads still work without login.
- Protected test requests reject missing/invalid bearer tokens.
- Valid tokens establish a stable MCP principal.
- No existing Tangled mutation code is changed yet.

## Phase 2: AT Protocol account linking

Once the MCP OAuth shell is stable, connect the authenticated MCP principal to a real Tangled/AT Protocol identity.

### Flow

1. Ask the user for their Tangled/AT Protocol handle during authorization.
2. Resolve handle to DID.
3. Resolve DID to PDS.
4. Discover the PDS authorization server.
5. Start AT Protocol OAuth.
6. Complete Authorization Code + PKCE.
7. Persist the resulting AT Protocol session server-side.

### Important constraint

Do not hand-roll AT Protocol OAuth. Use a maintained AT Protocol OAuth SDK that handles:

- distributed authorization-server discovery
- PKCE
- PAR
- DPoP
- token refresh
- persistent session state

### AT Protocol client metadata

Expose public client metadata, for example:

```text
/oauth-client-metadata.json
```

It should define:

- public client ID URL
- client name
- redirect URI
- authorization-code grant
- refresh-token grant
- requested AT Protocol scopes

Use a broad compatibility scope initially if needed to preserve current record/blob capabilities, then narrow it later.

## Phase 3: Session persistence and write-tool migration

Persist the linked AT Protocol account and replace hosted app-password login with session lookup.

### Minimum persisted account fields

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

Use DID as the durable AT Protocol identity because handles can change.

### Refactor target

Preserve the existing write-tool call shape where possible:

```python
session = await records.login()
```

Change hosted behavior from:

```text
raw credentials
   -> createSession()
   -> temporary access JWT
```

to:

```text
current MCP principal
   -> load linked AT Protocol session
   -> refresh if necessary
   -> return authenticated Session
```

### Preserve existing Tangled write behavior

Do not rewrite:

- issue record creation
- pull record creation
- patch synthesis
- pull-round updates
- comment record formats
- blob upload logic
- issue/pull state records

Only replace the authentication/session source.

## Phase 4: Per-tool authorization and ChatGPT hardening

Add the metadata ChatGPT needs to distinguish reads, writes, and destructive operations.

### Anonymous/read-only tools

Keep these anonymous where possible:

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

### OAuth-required mutation tools

Require `tangled:write` for:

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

### Tool safety metadata

- mark reads as read-only
- mark mutations as writes
- mark destructive operations such as delete as destructive

### Hardening checks

- reconnect behavior
- MCP token refresh
- AT Protocol token refresh
- expired-session recovery
- self-hosted PDS accounts
- account/handle changes
- permission failures
- removal of raw credential headers from hosted deployment
- preservation of local/CLI legacy auth if desired

## Expected source changes

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

### `records.py`

- retire hosted `x-tangled-handle` / `x-tangled-password`
- retire hosted `createSession()` login
- resolve current MCP principal
- load linked AT Protocol OAuth session
- refresh linked session when required
- optionally preserve app-password login for local CLI use

### `settings.py`

Add hosted OAuth configuration such as:

```text
PUBLIC_BASE_URL
OAUTH_ISSUER
SESSION_ENCRYPTION_KEY
DATABASE_URL
```

The hosted path must not require `TANGLED_PASSWORD`.

### `server.py`

- attach authenticated principal to write requests
- add per-tool authorization metadata
- add read/write/destructive annotations
- preserve anonymous public reads

## Migration rule

Treat this as an authentication-front-door replacement, not a rewrite of the Tangled MCP.

```text
old

ChatGPT/client
   -> x-tangled-handle + x-tangled-password
   -> createSession()
   -> PDS

new

ChatGPT
   -> MCP OAuth
   -> authenticated MCP principal
   -> stored AT Protocol OAuth session
   -> PDS
```

## Next build step

Begin with **Phase 1 only**:

1. protected-resource metadata
2. authorization-server metadata
3. bearer-token validation
4. authenticated MCP principal
5. anonymous reads preserved
6. Tangled write behavior untouched

That creates a testable OAuth front door before introducing AT Protocol account linking and session persistence.
