# Infisiclaw Secret Folder Templates

Template-driven secret folder creation for Infisical.  Each template
defines a set of secrets (env vars) that get created as a folder in
an Infisical project when instantiated.

## Template Format

Each `.yaml` file in a template directory defines one template:

```yaml
name: hermes-full          # unique slug (used as folder name)
description: "Full Hermes Agent secret set"
version: 1

# Optional: inherit from other templates (merged in order)
extends:
  - base/model-providers
  - base/auth-tokens

# Secret definitions
secrets:
  # Simple placeholder — user fills in the value
  - name: OPENROUTER_API_KEY
    placeholder: "sk-or-..."          # visible hint, not the real value
    description: "OpenRouter API key for model access"
    group: model-providers             # logical grouping

  # Auto-generated value — random string created on instantiation
  - name: SESSION_SECRET
    generate: random                   # random 32-char hex
    description: "Signing key for session tokens"
    group: auth

  # Static default — created with this value (user can override)
  - name: DEFAULT_MODEL
    value: "anthropic/claude-sonnet-4"
    description: "Default LLM model"
    group: config

  # Value from a prompt — user is asked during creation
  - name: DISCORD_BOT_TOKEN
    prompt: "Enter your Discord bot token"
    description: "Discord bot token for gateway"
    group: platform
```

## Secret Types

| Field        | Behavior |
|-------------|----------|
| `value`     | Static value — created as-is |
| `placeholder` | Empty value with a hint string shown during creation |
| `generate`  | Auto-generate: `random` (32-char hex), `uuid` (v4), `base64` (32 bytes) |
| `prompt`    | Ask the user for the value during instantiation |

If both `value` and `placeholder` are set, `placeholder` is the hint
shown to the user and `value` is the default they can accept or override.

## Usage

```bash
# List available templates
infisiclaw templates list

# Create a folder from a template
infisiclaw create --template hermes-full \
  --project <project-id> \
  --env prod \
  --path /agents/kitt

# Create with environment-specific overrides
infisiclaw create --template openclaw-base \
  --project <project-id> \
  --env prod \
  --set OPENROUTER_API_KEY=sk-or-real-key \
  --set DEFAULT_MODEL=anthropic/claude-opus-4
```

## Inheritance

Templates can `extends` other templates.  Secrets are merged in order:
base first, then the inheriting template.  If a secret name appears
in both, the inheriting template's definition wins (override).

## Planned: Cross-Manager Backup Sync

On folder creation, optionally mirror secrets to a secondary secret
manager as a backup.  Targets (in priority order):

1. **Bitwarden / Vaultwarden** — via the `bw` CLI or Vaultwarden API.
   Uses the `bws` (Bitwarden Secrets Manager) SDK for machine-to-machine
   access, or the `bw` CLI with a service account token for self-hosted
   Vaultwarden instances.  Secrets are written to a matching folder
   structure in the Bitwarden vault.

2. **HashiCorp Vault** — via the `vault` CLI or HTTP API.  Secrets are
   written to the `secret/` engine under a path matching the Infisical
   folder structure (e.g. `secret/infisiclaw/hermes/prod`).

Sync options in the template or CLI:

```bash
# Create with Bitwarden backup
infisiclaw create --template hermes-full \
  --project <id> --env prod --path /agents/kitt \
  --sync bitwarden \
  --sync-vault "infisiclaw-backups"

# Create with Vault backup
infisiclaw create --template hermes-full \
  --project <id> --env prod --path /agents/kitt \
  --sync vault \
  --sync-path "secret/infisiclaw/hermes"
```

**Why this matters:** Infisical's machine identity is read-only (E2EE
write path requires ciphertext the CLI can't produce at the API layer —
see `references/infisical-write-block.md`).  A sync to a second manager
gives you a human-writable backup that survives an Infisical outage or
migration, without needing Infisical write access.

**Template field:**

```yaml
sync:
  - manager: bitwarden
    vault: "infisiclaw-backups"     # Bitwarden vault/collection name
    folder: "{{ path }}"            # template variable, defaults to Infisical path
  - manager: vault
    engine: "secret"                # KV engine path
    prefix: "infisiclaw"            # prepended to the Infisical path
```

## Directory Structure

```
templates/
├── base/                    # reusable building blocks
│   ├── model-providers.yaml
│   ├── auth-tokens.yaml
│   ├── database.yaml
│   ├── storage.yaml
│   └── platform-tokens.yaml
├── hermes/                  # Hermes Agent integration
│   └── full.yaml
├── openclaw/                # OpenClaw integration
│   └── base.yaml
└── opencode/                # OpenCode integration
    └── base.yaml
```
