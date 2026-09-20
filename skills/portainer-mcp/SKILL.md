---
name: portainer-mcp
description: Use the Portainer MCP server to inspect and manage Portainer environments, stacks, services, users, teams, and access groups, or run raw Docker/Kubernetes proxy calls through Portainer. Use when you need to query Portainer state, fetch stack files, update stacks, or manage environments/teams/access groups without manual curl calls.
metadata:
  source_repo: lancer1977/portainer-mcp
  migrated_from: lancer1977/dev-forge (skills/portainer-mcp, see dev-forge#2613)
---

# Portainer MCP

Use this skill when Portainer is the control plane and this repo's `portainer-mcp` binary is registered as an MCP server.

## Capability

This repo builds the Go implementation of the Model Context Protocol server for Portainer (`cmd/portainer-mcp`). It exposes Portainer operations as MCP tools:

- List and inspect environments, environment groups, environment tags
- List and manage access groups (Portainer's term for Endpoint Groups) and their user/team accesses
- List and manage teams, team members, users, and user roles
- List stacks, fetch stack files, create/update stacks
- Read Portainer settings
- Raw Docker API proxy calls (`dockerProxy`) and Kubernetes API proxy calls (`kubernetesProxy`) against a given environment

The full, versioned tool contract (names, parameters, JSON Schema, read/write annotations) is defined in `internal/tooldef/tools.yaml` — treat that file, not this document, as the source of truth when it disagrees.

## Intended Use

Use when you need to:

- Inspect Portainer state (environments, stacks, teams, users, access groups)
- Fetch or update a stack's compose/manifest content
- Manage environment grouping, tagging, and access control
- Issue a Docker or Kubernetes API call through Portainer's proxy instead of a direct socket/API connection

Prefer read-only operations first (`listEnvironments`, `listStacks`, `getStackFile`, `getSettings`, etc.) before any create/update tool. Make changes deliberately and verify results.

## Activation Trigger

Triggered when:

- User mentions "portainer" and you have MCP tool access
- User asks about environment/stack/team/user state on Portainer
- User wants to update a stack, manage access groups, or run a Docker/Kubernetes proxy call through Portainer

## Prerequisites

The `portainer-mcp` binary must be built and registered as an MCP server:

1. Build from source: `cd ~/code/portainer-mcp && make build` — produces `dist/portainer-mcp` (or download a release binary; see this repo's `README.md` → "Installation").
2. Register it with your MCP client (Claude Desktop, Claude Code, etc.), pointing `command` at the built binary and passing:
   - `-server [IP]:[PORT]` — Portainer instance host:port
   - `-token [TOKEN]` — a Portainer API access token (admin token recommended)
   - optionally `-tools /path/to/tools.yaml` — override the embedded tool definitions (defaults to a `tools.yaml` created next to the binary)
   - optionally `-read-only` — expose only read (list/get) tools; no docker/kubernetes proxy tools
   - optionally `-disable-version-check` — bypass the startup check that pins this build to a specific Portainer server version (see this repo's README → "Portainer Version Support")
3. No `PORTAINER_USER`/`PORTAINER_PASSWORD` env-var path exists in this implementation — authentication is API-token only, passed via the `-token` flag.

See this repo's `README.md` for the full Claude Desktop config example and version-compatibility table.

## Tool surface

Representative tools from `internal/tooldef/tools.yaml` (~32 top-level tools total; run `make build && dist/portainer-mcp -tools /tmp/tools.yaml` or read the YAML directly for the exhaustive, versioned list):

| Tool | Purpose |
| ---- | ------- |
| `listEnvironments` | List all Portainer environments |
| `listEnvironmentGroups` / `createEnvironmentGroup` / `updateEnvironmentGroupName` / `updateEnvironmentGroupEnvironments` | Manage environment groups |
| `listEnvironmentTags` / `createEnvironmentTag` / `updateEnvironmentTags` | Manage environment tags |
| `listAccessGroups` / `createAccessGroup` / `addEnvironmentToAccessGroup` / `removeEnvironmentFromAccessGroup` | Manage access groups (Endpoint Groups) |
| `updateAccessGroupName` / `updateAccessGroupUserAccesses` / `updateAccessGroupTeamAccesses` | Update access group membership/permissions |
| `updateEnvironmentUserAccesses` / `updateEnvironmentTeamAccesses` | Per-environment user/team access |
| `listTeams` / `createTeam` / `updateTeamName` / `updateTeamMembers` | Manage teams |
| `listUsers` / `updateUserRole` | Manage users |
| `listStacks` / `getStackFile` / `createStack` / `updateStack` | Inspect and manage stacks |
| `getSettings` | Read Portainer instance settings |
| `getKubernetesResourceStripped` | Read a Kubernetes resource with noise stripped |
| `dockerProxy` | Raw Docker API call proxied through Portainer for a given environment |
| `kubernetesProxy` | Raw Kubernetes API call proxied through Portainer for a given environment |

Tool names and parameter shapes are fixed by the embedded schema — do not rely on descriptions alone; only tool *descriptions* (not names/params) are meant to be customizable via a `-tools` override file.

## Constraints

- Authentication is API-token only (`-token` flag) — there is no username/password mode in this implementation.
- Prefer `-read-only` mode when write access isn't needed for the task at hand.
- Always inspect (`listStacks` / `getStackFile`) before `updateStack`.
- Do not change tool names or parameter definitions in a custom `-tools` file — only descriptions; changing names/params breaks tool registration.
- Do not claim success without verifying the MCP tool's response.
- This server is pinned to a specific supported Portainer server version per release (see README table); `-disable-version-check` may partially or fully break write operations against a mismatched Portainer version.

## Expected Output

- Inspection results as structured data (JSON per the tool's schema)
- Stack files as raw compose/manifest content
- Docker/Kubernetes proxy responses as raw API JSON
- Success/failure confirmation for create/update operations

## Related files

- `~/code/portainer-mcp/README.md` — installation, Claude Desktop config example, version-check/read-only flags, version-compatibility table
- `~/code/portainer-mcp/internal/tooldef/tools.yaml` — authoritative, versioned tool/parameter definitions
- `~/code/portainer-mcp/cmd/portainer-mcp/mcp.go` — server entrypoint and CLI flag handling
- `~/code/portainer-mcp/Makefile` — `make build` / `make test` / `make inspector` targets

## Migration note

This skill was migrated from `lancer1977/dev-forge` (`skills/portainer-mcp`) as part of the dev-forge retirement (see [lancer1977/dev-forge#2613](https://github.com/lancer1977/dev-forge/issues/2613)). The original dev-forge copy documented a **different, Python-based** MCP wrapper (`mcps/portainer-mcp`, snake_case tools like `portainer_stack_get`) that is not present in this repo; this rewrite replaces that tool surface and install path with this repo's actual Go implementation and build contract. dev-forge's copy is expected to become a breadcrumb pointing here in a follow-up dev-forge PR.
