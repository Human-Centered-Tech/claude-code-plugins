# Load Secrets from Vaultwarden

Load project secrets from HCT's Vaultwarden instance into local environment variables — safely, without exposing values to the LLM or terminal output.

## When to use

- Starting work on any HCT project that needs API keys, database credentials, or other secrets
- Switching between projects or environments (production, staging, client-specific)
- Setting up a new development environment

## Prerequisites

1. **Bitwarden CLI installed:**
   ```bash
   npm install -g @bitwarden/cli
   ```

2. **CLI configured for HCT vault:**
   ```bash
   bw config server https://vault.humancenteredtech.com
   ```

3. **Vaultwarden account** — ask your admin or run the onboard skill if you're new

4. **`bw-env` script** from the hct-tools repo:
   ```bash
   git clone https://github.com/Human-Centered-Tech/hct-tools.git
   # Add hct-tools/scripts to your PATH
   ```

## Usage

### Load secrets for a scope

```bash
eval $(bw-env --scope production)
```

This will:
1. Prompt for your Vaultwarden master password (value never displayed)
2. Fetch all secrets from the "production" folder in your vault
3. Export them as environment variables in your current shell
4. Write a Claude memory note listing the variable **names** (not values)

### List available scopes

```bash
bw-env --list
```

### Common scopes

| Scope | Contents |
|-------|----------|
| `production` | Production API keys, database URLs, SMTP credentials |
| `staging` | Staging environment credentials |
| `hct-internal` | Internal service tokens (Vikunja API, Seafile admin, etc.) |

## How secrets are organized in Vaultwarden

Each scope corresponds to a **folder** in Vaultwarden. Within each folder:

- **Login items**: The item name becomes the env var name, the password field becomes the value
  - Example: Item named `STRIPE_SECRET_KEY` with password `sk_live_...` → `export STRIPE_SECRET_KEY="sk_live_..."`
- **Secure notes**: The item name becomes the env var name, the note body becomes the value
  - Useful for multi-line values like JSON credentials

## Security notes

- Secret values are **never printed** to stdout — they flow through `eval` directly into env vars
- The Claude memory note only records variable **names**, never values
- Your Vaultwarden master password is prompted by the Bitwarden CLI, not by this script
- Sessions expire after inactivity — you may need to re-authenticate periodically

## Troubleshooting

| Issue | Fix |
|-------|-----|
| "Bitwarden CLI not found" | `npm install -g @bitwarden/cli` |
| "Failed to login" | Run `bw login` first, then retry |
| "Folder not found" | Check available folders with `bw-env --list` |
| Variables not set after eval | Make sure you're using `eval $(...)` not just running the script directly |
| Windows path in variable value | Use the Raw Editor in Railway dashboard instead of CLI for path-containing vars |
