# Migrating from Bitwarden Cloud to Vaultwarden

This guide covers migrating an existing Bitwarden paid subscription (including a
family organization) to a self-hosted Vaultwarden instance. The end-user
experience after migration is changing one URL setting in their Bitwarden client.

---

## Prerequisites

- Vaultwarden is running at `https://bitwarden.<yourdomain>` (Caddy is proxying,
  TLS cert is valid, login page is accessible)
- `SIGNUPS_ALLOWED=true` is set in `/opt/vaultwarden/vaultwarden.env` on the
  vaultwarden LXC
- You have admin access to both the existing Bitwarden family organization and
  the Vaultwarden `/admin` panel

---

## Step 1 — Export vaults from Bitwarden cloud

Do this while the Bitwarden subscription is still active.

**Each family member's personal vault:**
1. Log into `bitwarden.com` with their account
2. Tools → Export Vault → Format: `JSON` → confirm master password → download
3. Store the file somewhere secure (it contains all passwords in plaintext)

**Family organization vault (admin only):**
1. Log into `bitwarden.com` as the org admin
2. In the left panel, switch to the organization view
3. Tools → Export Vault → Format: `JSON` → download

---

## Step 2 — Create accounts on Vaultwarden

For each family member, go to `https://bitwarden.<yourdomain>` and create an
account.

- Use the **same email address** as their existing Bitwarden account
- Set the **same master password** — this means end users don't need to remember
  new credentials

---

## Step 3 — Import personal vaults

For each family member's account:
1. Log into `https://bitwarden.<yourdomain>`
2. Tools → Import Data
3. Select format: **Bitwarden (json)**
4. Upload their personal vault export file
5. Verify a few items look correct

---

## Step 4 — Recreate the family organization

Organizations are not included in personal vault exports — the org vault must be
imported separately.

1. Log into Vaultwarden as your (the admin's) account
2. Create a new Organization — name it whatever you like
3. With the organization selected in the left panel, go to Tools → Import Data
4. Select format: **Bitwarden (json)** → upload the org vault export
5. Verify org items are present

**Inviting members without SMTP:**

Vaultwarden sends org invitations by email, but SMTP is not configured on this
server. Use the admin panel instead:

1. Go to `https://bitwarden.<yourdomain>/admin` → log in with your admin token
2. In the Vaultwarden web vault, invite the other family members to the org
   (they will appear as "Invited" with no email sent)
3. In the `/admin` panel → Users → find each invitee → manually confirm them
4. Back in the web vault, assign them the appropriate org role (Member, Manager, etc.)

---

## Step 5 — Switch clients to the new server

Send this to each family member:

> Open your Bitwarden app → Settings → Account → **Server URL**
> → change to `https://bitwarden.<yourdomain>`
> → log in with your usual email and master password

This applies to browser extensions, mobile apps, and desktop apps. Each client
may need to log out first before the server URL field becomes editable.

After logging in, their full vault (personal items + org items) should appear
exactly as before.

---

## Step 6 — Lock down signups

Once all family members have successfully logged in:

1. SSH to the vaultwarden LXC:
   ```bash
   ssh vaultwarden
   ```

2. Edit `/opt/vaultwarden/vaultwarden.env` and set:
   ```
   SIGNUPS_ALLOWED=false
   ```

3. Restart Vaultwarden:
   ```bash
   docker compose -f /opt/vaultwarden/docker-compose.yml restart
   ```

---

## Step 7 — Verify and cancel Bitwarden subscription

Before cancelling:
- Confirm each family member can log in and see their full vault on Vaultwarden
- Confirm org items are accessible to all org members
- Confirm browser extension / mobile app is working day-to-day for a day or two

Once satisfied, cancel the Bitwarden family subscription at `bitwarden.com`.
Individual Bitwarden free-tier accounts can be kept or deleted — they have no
effect on your Vaultwarden server.

---

## Notes

- **2FA:** Any TOTP authenticator apps (Google Authenticator, Authy, etc.)
  remain valid — TOTP secrets are stored in the vault and migrate with the vault
  export. However, the 2FA *enrollment* in the Vaultwarden account is separate:
  each user will need to re-enroll their authenticator in
  Vaultwarden under Account Settings → Security → Two-step Login.
- **SMTP:** Not configured. Email-based features (new device alerts, email 2FA,
  password hints) are unavailable. TOTP is the recommended 2FA method.
- **Attachments:** File attachments are **not included** in the standard JSON
  export. If any vault items have file attachments, these must be manually
  re-uploaded after migration.
