# Feature: De-gated Folder-Level RBAC for Machine Identities

## Problem

Infisical gates **groups** and **RBAC** behind the Enterprise plan.
For a multi-agent setup (Hermes gateway, OpenClaw, OpenCode, etc.),
you need per-folder access control at the machine identity level
*without* an Enterprise license.

**Current behavior (non-enterprise):**
- Machine identities get a single role per project (admin/member/viewer)
- No folder-level scoping — an identity with project read access sees
  *everything* in that project
- No groups — can't organize identities by function

**What we need:**
- Machine identities with folder-scoped permissions within a single project
- Group-based identity management (e.g., "hermes-agents", "ci-agents")
- Granular read/write per secret path per environment
- Project-wide "shared" folders accessible to all agents
- Agent-specific folders accessible only to that agent's identity

## Target Architecture

### Folder Layout (per project)

```
/project/
├── shared/                    # project-wide, all identities can read
│   ├── model-providers/       # API keys shared across agents
│   ├── platform-tokens/       # shared Discord/Slack tokens
│   └── config/                # shared config values
├── agents/                    # per-agent scoped
│   ├── kitt/                  # Hermes agent "KITT"
│   │   ├── auth/              # KITT-specific auth tokens
│   │   └── tools/             # KITT-specific tool keys
│   ├── smithers/              # Hermes agent "Smithers"
│   │   ├── auth/
│   │   └── tools/
│   └── opencode-default/      # OpenCode agent
│       └── auth/
└── integrations/              # integration-specific
    └── infisiclaw/            # template-created secrets
```

### Permission Model

Each machine identity gets a set of **scope rules**:

| Identity | Environment | Secret Path | Permission |
|----------|-------------|-------------|------------|
| kitt | prod | `/shared/*` | read |
| kitt | prod | `/agents/kitt/**` | read, write |
| kitt | prod | `/agents/smithers/**` | — (no access) |
| smithers | prod | `/shared/*` | read |
| smithers | prod | `/agents/smithers/**` | read, write |
| smithers | prod | `/agents/kitt/**` | — (no access) |
| ci-runner | prod | `/shared/config/*` | read |
| ci-runner | prod | `/agents/*` | — (no access) |

### Implementation Changes

#### 1. De-gate Groups (license-fns.ts)

```typescript
// backend/src/ee/services/license/license-fns.ts
// Change the default plan features:
groups: true,    // was: false
rbac: true,      // was: false
```

This unlocks the Groups UI and RBAC management for all plan tiers.

#### 2. Add Folder-Scope to Identity-Project Membership

New schema field on `IdentityProjectMembership`:

```typescript
// backend/src/db/schemas/identity-project-membership.ts
folderScopes: z.array(z.object({
  environment: z.string(),
  secretPath: z.string(),      // glob pattern: /shared/*, /agents/kitt/**
  permission: z.enum(["read", "readwrite"]),
})).optional()
```

When `folderScopes` is empty/undefined → identity has full project access
(legacy behavior). When set → identity is restricted to matching paths.

#### 3. Enforce Folder Scopes in Secret Access

In `backend/src/services/secret/secret-service.ts`, when a machine
identity reads secrets, intersect the requested `secretPath` against
the identity's `folderScopes`:

```typescript
// Pseudocode
function filterByFolderScopes(secrets, identityFolderScopes, environment) {
  if (!identityFolderScopes?.length) return secrets; // legacy: no restriction

  return secrets.filter(secret => {
    return identityFolderScopes.some(scope =>
      scope.environment === environment &&
      picomatch.isMatch(secret.path, scope.secretPath)
    );
  });
}
```

#### 4. API Endpoints for Scope Management

```
POST   /api/v2/identity-project/:id/folder-scopes     # add scope
DELETE /api/v2/identity-project/:id/folder-scopes/:idx # remove scope
GET    /api/v2/identity-project/:id/folder-scopes      # list scopes
```

#### 5. UI: Identity → Project → Folder Permissions

In the Identity details page, under "Project Memberships", add a
"Folder Permissions" section where admins can:

- Add folder scope rules (environment + path glob + permission level)
- See a visual tree of accessible paths
- Test: "What can this identity see?" → shows filtered secret list

### Migration Path

1. **Phase 1** (fork): De-gate groups + RBAC, add `folderScopes` field
2. **Phase 2** (fork): Add folder scope management API + UI
3. **Phase 3** (upstream PR): Propose de-gating groups/RBAC as
   community contribution (may be rejected — that's fine, the fork
   has it)

### Hermes Plugin Integration

The `infisical-secrets` plugin doesn't need changes — it already
authenticates as a machine identity and fetches secrets from a
configured `project_id + env + path`.  The folder scoping happens
server-side in Infisical: the identity's token only returns secrets
it has access to.

The plugin just needs the right `path` in its config:

```yaml
secrets:
  infisical:
    enabled: true
    project_id: "<uuid>"
    env: "prod"
    path: "/agents/kitt"       # identity only sees its own folder
    # path: "/shared"          # or read from the shared folder
```

Multiple source entries for multiple folders:

```yaml
secrets:
  sources: [infisical-shared, infisical-kitt]
  infisical-shared:
    enabled: true
    project_id: "<uuid>"
    env: "prod"
    path: "/shared"
  infisical-kitt:
    enabled: true
    project_id: "<uuid>"
    env: "prod"
    path: "/agents/kitt"
```

## Open Questions

1. Should folder scopes be defined on the identity-project membership
   (as proposed) or as a separate "identity-scope" entity?
2. Should groups inherit folder scopes (a group-level scope applies
   to all identities in the group)?
3. Should temporary/scoped access (time-limited folder access) be
   supported from day one?
