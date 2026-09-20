---
name: portainer-mcp
description: Operate Portainer through the `portainer` MCP server. Primary: `mcp-portainer` (PyPI, launched via `uvx`) — the operator's live, registered setup. Secondary: this repo's own Go implementation (`portainer-mcp`), built from source, as an alternative/fallback. Use when you need to query Portainer state, fetch stack files, update stacks, or manage environments/teams/access groups without manual curl calls.
metadata:
  source_repo: lancer1977/portainer-mcp
  migrated_from: lancer1977/dev-forge (skills/portainer-mcp, see dev-forge#2613)
---

# Portainer MCP

Use this skill when Portainer is the control plane and a `portainer` MCP server is registered. The operator's live setup is the third-party `mcp-portainer` Python package launched via `uvx` — that is the primary path. This repo's own Go server is kept as a documented alternative, e.g. to build and run from source.

## Live setup (mcp-portainer via uvx)

This is what's actually registered under user scope today.

- **Registration** (names shown, never real values):
  ```
  claude mcp add portainer -s user \
    -e PORTAINER_URL=https://<portainer-host> \
    -e PORTAINER_TLS_VERIFY=0 \
    -e PORTAINER_API_KEY=<api-key> \
    -- uvx --from mcp-portainer~=2.41.0 mcp-portainer
  ```
- **Env vars** (names only, never values):
  - `PORTAINER_URL` — base URL of the Portainer instance.
  - `PORTAINER_TLS_VERIFY` — set to `0` because the LAN instance uses a self-signed certificate; TLS verification is intentionally off for this deployment.
  - `PORTAINER_API_KEY` — the Portainer API access token.
- **Tool prefix**: all tools from this server surface as `mcp__portainer__*`.
- **Standing rule — MCP-only, never source the token**: always use the `mcp__portainer__*` tools; never `source` the API key into a shell or curl with it directly (see memory rule "Portainer MCP-only, never source token"). If a task seems to need raw `curl`, that's a signal to find the equivalent MCP tool instead.

### Known gotchas in this portfolio

- **`docker_proxy` needs an explicit `Content-Type` header** — omitting it causes calls to fail; always set it explicitly on proxied Docker API requests.
- **Swarm services update via `POST /services/{id}/update`, not via Stack.** Updating a Swarm service's image/config does not go through the Stack update tools — use the service-update endpoint directly.
- **`StackCreateDockerStandaloneString` drops Traefik labels on r620.** r620 runs Swarm, not standalone — using the standalone string-create path there silently loses Traefik labels. Confirm swarm vs. standalone before choosing which `StackCreate*` tool to use.
- **Portainer's cert is self-signed** — any raw `curl` against the instance needs `-k`; an HTTP `000`/TLS error from `curl` does not mean the service is down, just that verification failed.

## Alternative: this repo's Go server (build from source)

This repo builds a separate Go implementation of a Portainer MCP server (`cmd/portainer-mcp`). Use it only when the `mcp-portainer` (uvx) path above isn't appropriate — e.g. developing against this repo directly, or needing a capability specific to its tool contract.

### Capability

Exposes Portainer operations as MCP tools:

- List and inspect environments, environment groups, environment tags
- List and manage access groups (Endpoint Groups) and their user/team accesses
- List and manage teams, team members, users, and user roles
- List stacks, fetch stack files, create/update stacks
- Read Portainer settings
- Raw Docker (`dockerProxy`) and Kubernetes (`kubernetesProxy`) API proxy calls against a given environment

The full, versioned tool contract (names, parameters, JSON Schema, read/write annotations) is defined in `internal/tooldef/tools.yaml` — treat that file, not this document, as the source of truth when it disagrees.

### Build and register

1. Build from source: `cd ~/code/portainer-mcp && make build` — produces `dist/portainer-mcp` (or download a release binary; see this repo's `README.md` → "Installation").
2. Register it with your MCP client, pointing `command` at the built binary and passing:
   - `-server [IP]:[PORT]` — Portainer instance host:port
   - `-token [TOKEN]` — a Portainer API access token (admin token recommended)
   - optionally `-tools /path/to/tools.yaml` — override the embedded tool definitions
   - optionally `-read-only` — expose only read (list/get) tools; no docker/kubernetes proxy tools
   - optionally `-disable-version-check` — bypass the startup check pinning this build to a specific Portainer server version
3. Authentication is API-token only (`-token` flag) — no username/password env-var path exists in this implementation, unlike `mcp-portainer`'s env-based config above.

See this repo's `README.md` for the full config example and version-compatibility table.

### Tool surface (differs from mcp-portainer)

Representative tools from `internal/tooldef/tools.yaml` (~32 top-level tools; run `make build && dist/portainer-mcp -tools /path/to/tools.yaml` or read the YAML directly for the exhaustive list). Tool names here are **camelCase** (`listEnvironments`, `dockerProxy`, `getStackFile`, ...) and differ from the `mcp-portainer` tool names surfaced under the `mcp__portainer__*` prefix above — check the actually-registered server's own listing rather than assuming names carry over.

| Tool | Purpose |
| ---- | ------- |
| `listEnvironments` | List all Portainer environments |
| `listEnvironmentGroups` / `createEnvironmentGroup` / `updateEnvironmentGroupName` / `updateEnvironmentGroupEnvironments` | Manage environment groups |
| `listEnvironmentTags` / `createEnvironmentTag` / `updateEnvironmentTags` | Manage environment tags |
| `listAccessGroups` / `createAccessGroup` / `addEnvironmentToAccessGroup` / `removeEnvironmentFromAccessGroup` | Manage access groups |
| `updateAccessGroupName` / `updateAccessGroupUserAccesses` / `updateAccessGroupTeamAccesses` | Update access group membership/permissions |
| `updateEnvironmentUserAccesses` / `updateEnvironmentTeamAccesses` | Per-environment user/team access |
| `listTeams` / `createTeam` / `updateTeamName` / `updateTeamMembers` | Manage teams |
| `listUsers` / `updateUserRole` | Manage users |
| `listStacks` / `getStackFile` / `createStack` / `updateStack` | Inspect and manage stacks |
| `getSettings` | Read Portainer instance settings |
| `getKubernetesResourceStripped` | Read a Kubernetes resource with noise stripped |
| `dockerProxy` | Raw Docker API call proxied through Portainer for a given environment |
| `kubernetesProxy` | Raw Kubernetes API call proxied through Portainer for a given environment |

Only tool *descriptions* (not names/params) are customizable via a `-tools` override file — changing names/params breaks tool registration.

### Constraints

- Prefer `-read-only` mode when write access isn't needed for the task at hand.
- Always inspect (`listStacks` / `getStackFile`) before `updateStack`.
- This server is pinned to a specific supported Portainer server version per release; `-disable-version-check` may partially or fully break write operations against a mismatched version.

### Related files

- `~/code/portainer-mcp/README.md` — installation, config example, version-check/read-only flags, version-compatibility table
- `~/code/portainer-mcp/internal/tooldef/tools.yaml` — authoritative, versioned tool/parameter definitions
- `~/code/portainer-mcp/cmd/portainer-mcp/mcp.go` — server entrypoint and CLI flag handling
- `~/code/portainer-mcp/Makefile` — `make build` / `make test` / `make inspector` targets

## Intended use (either server)

Use when you need to:

- Inspect Portainer state (environments, stacks, teams, users, access groups)
- Fetch or update a stack's compose/manifest content
- Manage environment grouping, tagging, and access control
- Issue a Docker or Kubernetes API call through Portainer's proxy instead of a direct socket/API connection

Prefer read-only operations first before any create/update tool. Make changes deliberately and verify results. Do not claim success without verifying the MCP tool's response.

## Activation trigger

Triggered when:

- User mentions "portainer" and an MCP `portainer` tool is available
- User asks about environment/stack/team/user state on Portainer
- User wants to update a stack, manage access groups, or run a Docker/Kubernetes proxy call through Portainer

## Expected output

- Inspection results as structured data (JSON per the tool's schema)
- Stack files as raw compose/manifest content
- Docker/Kubernetes proxy responses as raw API JSON
- Success/failure confirmation for create/update operations

## Secret handling

- Never source `PORTAINER_API_KEY` (or the Go server's `-token` value) into a shell, log it, or paste the value into chat/commits — reference the env var / flag by name only.
- Always operate through the registered MCP tools (`mcp__portainer__*` for the live setup); do not fall back to raw `curl` with a manually-sourced token as a shortcut.
- If a raw HTTP call is genuinely required for the Go server's self-signed instance, use `curl -k` and still avoid printing the token in the command as a bare argument.

## Migration note

This skill was migrated from `lancer1977/dev-forge` (`skills/portainer-mcp`) as part of the dev-forge retirement (see [lancer1977/dev-forge#2613](https://github.com/lancer1977/dev-forge/issues/2613)). It was later revised per operator decision to lead with the live `mcp-portainer` (uvx) setup actually registered in this portfolio, keeping this repo's Go server documented as a secondary/alternative path rather than the primary one.
