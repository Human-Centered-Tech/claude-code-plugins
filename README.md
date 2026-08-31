# HCT Shared Claude Code Skills

Shared Claude Code plugins for the Human Centered Tech team.

The goal of this repo is to host a Claude Code plugin which can enable us to share the same set of organizational AI agent skills. We will no doubt have personal ways of doing things which we record in our own personal skills folders, as well as per project skills files. But if there are things which we are finding we want our agents to do across the board for the whole organization, or if we just want to share knowledge that we think would be helpful for other people to add to their agent behavior, this is the place for it. In general, our goal is to grow rapidly in practical wisdom about the art of software development and of facilitating excellent workflows for businesses, so this is repository for that collection of wisdom as it pertains to agent behavior.

## Installation

```
/plugin marketplace add Human-Centered-Tech/claude-code-plugins
/plugin install hct-shared-skills@hct-marketplace
```

## Available Skills

| Skill | Description |
|-------|-------------|
| `cross-platform` | Audit and fix a Node.js project for cross-platform compatibility (Windows + macOS + Linux) |
| `explain` | Explain a concept, mechanism, or piece of code in clear, concise language, end to end |
| `health-check-authwall` | Fix deploy health checks that fail against auth-protected apps (302/307 redirects) |
| `load-secrets` | Load project secrets from HCT's Vaultwarden into environment variables, without exposing the values |
| `mock-database-drizzle` | In-memory mock database pattern for testing Drizzle ORM services without Postgres |
| `nestjs-scoped-api-keys` | Add scoped API key auth alongside session auth in NestJS (hashing, guards, project scoping) |
| `onboard-team-member` | Create accounts across all HCT self-hosted services for a new team member in one command |
| `postgres-best-practices` | Reference guide and audit checklist for PostgreSQL schema design, query performance, operations, and security |

## Operational Notes

### Railway Custom Domain Setup

When provisioning a custom domain on Railway, you need **both** DNS records:

1. **CNAME record** — Point your subdomain to the Railway-provided target (e.g., `0mxhr5ec.up.railway.app`). Note: this target differs from your service's default `*.up.railway.app` domain.
2. **TXT verification record** — Railway requires a TXT record for domain ownership verification. Without it, the Let's Encrypt SSL certificate will not provision and you'll get SSL errors. Check the Railway dashboard for the exact TXT record value after adding the custom domain.

SSL certs typically provision within minutes once both records are in place and DNS has propagated. You can flush Google's DNS cache at `https://dns.google/cache` if needed.

## Contributing

1. Clone this repo
2. Add or edit skills in `skills/<name>/SKILL.md`
3. Bump the version in `.claude-plugin/plugin.json`
4. Push to `main`
