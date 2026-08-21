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

## Design Decisions

### 1. Scope ownership: separate entity ✅

Folder scopes should be a **separate `IdentityFolderScope` entity**,
not embedded in the identity-project membership.  Reasons:

- **Multiple scopes per identity** — an identity might need 5+ folder
  rules (shared read, own folder readwrite, integration read, etc.).
  Embedding them bloats the membership row.
- **Audit trail** — separate entity gets its own `createdAt`,
  `updatedAt`, `createdBy` for tracking who granted which scope.
- **Reusability** — scopes can be templated.  "Kitt's standard scope"
  is a reusable definition applied to multiple projects.
- **Easier enforcement** — the secret-service DAL queries
  `IdentityFolderScope.find({ identityId, projectId, environment })`
  and intersects with the requested path.  No need to parse embedded
  arrays.

```typescript
// backend/src/db/schemas/identity-folder-scope.ts (new)
{
  id: string;                    // PK
  identityId: string;            // FK → Identity
  projectId: string;             // FK → Project
  environment: string;           // "prod", "staging", etc.
  secretPath: string;            // glob: "/shared/*", "/agents/kitt/**"
  permission: "read" | "readwrite";
  createdBy: string;             // actor who granted this scope
  createdAt: Date;
  updatedAt: Date;
}
```

### 2. Group inheritance ✅

Groups should **inherit folder scopes**.  When a group has a scope,
every identity in that group inherits it (subject to identity-level
overrides).

**Resolution order** (most specific wins):

1. **Identity-level scope** — directly assigned to the identity
   via `IdentityFolderScope`.  Highest priority.
2. **Group-level scope** — assigned to a group the identity belongs
   to.  Merged across all groups; if two groups grant different
   permissions on the same path, the **most permissive** wins
   (readwrite > read).
3. **Project default** — if no scopes exist for an identity (neither
   direct nor group-inherited), the identity gets full project access
   (legacy behavior, zero breaking change).

**Example:**

```
Group "hermes-agents":
  - /shared/* → read
  - /shared/config/* → readwrite

Identity "kitt" (member of "hermes-agents"):
  - /agents/kitt/** → readwrite    (identity-level)
  - /shared/* → read               (inherited from group)
  - /shared/config/* → readwrite   (inherited from group)
```

```typescript
// backend/src/db/schemas/group-folder-scope.ts (new)
{
  id: string;
  groupId: string;               // FK → Group
  projectId: string;
  environment: string;
  secretPath: string;
  permission: "read" | "readwrite";
  createdBy: string;
  createdAt: Date;
  updatedAt: Date;
}
```

**UI:** In the Groups page → Project Memberships → "Folder Permissions"
tab.  In the Identity page → "Inherited from groups" section showing
what scopes come from group membership (with a link to the source
group).

### 3. Time-limited access: deferred ❌

Not in scope for the initial implementation.  The existing
`TemporaryPermissionMode` infrastructure in the codebase can be
extended later if needed.  For now, all folder scopes are permanent
until manually revoked.
