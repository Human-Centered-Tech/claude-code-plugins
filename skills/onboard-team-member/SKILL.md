# Onboard a New Team Member

Create accounts across all HCT self-hosted services for a new team member in one command.

## When to use

- A new developer, contractor, or consultant is joining the team
- Someone needs access to HCT's internal tools (tasks, files, vault, video, booking)

## Prerequisites

1. **Admin credentials** loaded as env vars:
   ```bash
   eval $(bw-env --scope hct-internal)
   ```
   This should set: `VIKUNJA_API_TOKEN`, `SEAFILE_ADMIN_PASSWORD`, `VAULTWARDEN_ADMIN_TOKEN`

2. **`hct-onboard` script** from the hct-tools repo:
   ```bash
   git clone https://github.com/Human-Centered-Tech/hct-tools.git
   # Add hct-tools/scripts to your PATH
   ```

3. **curl** and **python3** available (standard on macOS/Linux)

## Usage

### Basic onboarding

```bash
hct-onboard \
  --name "Sarah Chen" \
  --email "sarah@humancenteredtech.com" \
  --slug sarah
```

This creates:
- **Vikunja account** at tasks.humancenteredtech.com (with a personal "Bookings" project)
- **Seafile account** at files.humancenteredtech.com
- **Vaultwarden invitation** sent to their email for vault.humancenteredtech.com
- Generates a temporary password and displays it at the end

### With booking page

```bash
hct-onboard \
  --name "Sarah Chen" \
  --email "sarah@humancenteredtech.com" \
  --slug sarah \
  --booking-page
```

This does everything above plus outputs the config block to add to the booking app's `src/lib/consultants.ts`. After adding it, redeploy the booking app and Sarah's booking page will be live at `cal.humancenteredtech.com/sarah`.

### With a specific password

```bash
hct-onboard \
  --name "Sarah Chen" \
  --email "sarah@humancenteredtech.com" \
  --slug sarah \
  --password "initial-temp-password"
```

## What gets created

| Service | Account Type | URL |
|---------|-------------|-----|
| Vikunja | User account + personal Bookings project | tasks.humancenteredtech.com |
| Seafile | User account | files.humancenteredtech.com |
| Vaultwarden | Email invitation (they complete registration) | vault.humancenteredtech.com |
| Jitsi | No account needed | meet.humancenteredtech.com |
| Booking | Config entry (if --booking-page) | cal.humancenteredtech.com/{slug} |

## After onboarding

1. **Send credentials securely** — use Vaultwarden Send (vault.humancenteredtech.com) to share the initial password. Never send passwords via email or Slack.

2. **Remind them to change passwords** on first login to each service.

3. **Add to Vaultwarden groups** — assign them to the appropriate folders/collections for their role (e.g., "production", "client-x").

4. **If --booking-page was used** — add the outputted config block to the booking app source code and redeploy:
   ```bash
   cd /path/to/hct-booking
   # Edit src/lib/consultants.ts
   git add -A && git commit -m "Add Sarah's booking page"
   railway up -d
   ```

## Service URLs to share with new member

```
Your HCT Office Suite:

📅 Tasks & Calendar: https://tasks.humancenteredtech.com
📁 File Sharing:     https://files.humancenteredtech.com
🔐 Password Vault:   https://vault.humancenteredtech.com
📹 Video Meetings:   https://meet.humancenteredtech.com
📆 Your Booking Page: https://cal.humancenteredtech.com/{slug}
```

## Troubleshooting

| Issue | Fix |
|-------|-----|
| "VIKUNJA_API_TOKEN not set" | Run `eval $(bw-env --scope hct-internal)` first |
| "Failed to authenticate with Seafile admin" | Check SEAFILE_ADMIN_PASSWORD is correct — it may have been changed |
| Vaultwarden invitation not received | Check spam folder; verify SMTP is working on the VPS |
| Booking page not showing | Remember to add the config to consultants.ts AND redeploy |
