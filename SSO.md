---
layout: default
title: Invenzo ITAM — SSO Setup (Google, Azure AD, Okta, OIDC, SAML)
description: Step-by-step guide to wiring an external identity provider into Invenzo
---

# SSO Setup

Invenzo supports three single-sign-on protocols out of the box:

| Protocol | Use for | Examples |
|---|---|---|
| **OIDC** (OpenID Connect) | Modern OAuth-based IdPs | Google · Microsoft Entra ID (Azure AD) · Okta · Auth0 · Keycloak · Authentik |
| **SAML 2.0** | Classic enterprise IdPs | ADFS · PingFederate · Shibboleth · older Okta tenants |
| **LDAP** | On-prem directories | Active Directory · OpenLDAP |

This guide walks through **Google** as the primary example (most common ask). The same Invenzo-side steps apply to any OIDC provider — only the Google Cloud Console part changes.

<small>© <a href="https://mitvaris.com">Mitvaris</a>. All rights reserved.</small>

---

## Self-hosted deployments — what you need to know first

Invenzo is **self-hosted**, so the SSO flow runs entirely between **the user's browser** and **your Invenzo host** — Google / Azure / Okta never need to reach into your network. But the user's browser DOES need to reach Invenzo, and the IdP needs to accept your hostname as a valid redirect URI. That creates three practical constraints:

1. **HTTPS is non-negotiable.** Every major OIDC provider (Google, Microsoft, Okta) rejects `http://` redirect URIs except `http://localhost`. Caddy auto-TLS handles this for free on the default install. If you're running behind a hostname Caddy can't issue a cert for (split-DNS internal, Active Directory-only domain, IP address), you'll need to point Caddy at a real cert from your internal PKI or Let's Encrypt.
2. **The redirect URI must be reachable from EVERY user's browser**, not from the internet. So `https://invenzo.corp.local` works fine for laptops on the corporate VPN — they don't need a public DNS record. But laptops outside the network can't sign in via Invenzo until they're on VPN. Same applies to mobile devices.
3. **Some IdPs reject "private" hostnames at registration time.** Google accepts arbitrary hostnames in the redirect URI as long as it's HTTPS (it has no way to know `invenzo.corp.local` is private). Azure AD and Okta accept any HTTPS hostname too. **The IdP only ever checks string-equality at redirect time** — it doesn't try to reach your URL itself.

**Net result**: Self-hosted Google/Azure/Okta SSO works perfectly for users who can reach the Invenzo host. No public DNS / public IP / port-forwarding required. It's the user's browser that does all the redirects, not the IdP's servers.

If your environment can't run HTTPS at all (legacy lab setup), use **LDAP / Active Directory** instead — no redirect URIs involved, just direct LDAP bind.

---

## How Invenzo's SSO works

1. Admin configures an **SSO Provider** record in Invenzo (Settings → Security → SSO Providers).
2. The Login page renders one button per active provider — `Continue with <Provider Name>`.
3. Clicking the button redirects to the IdP (Google / Azure / etc.) where the user signs in with their existing account.
4. The IdP redirects back to Invenzo's callback URL with an authorisation code.
5. Invenzo exchanges the code for a token, fetches the user's profile (email, name), and either:
   - **Auto-provisions** a new Invenzo user with the configured `default_role` (recommended: `viewer`), OR
   - Links to an existing Invenzo user whose `email` matches.
6. User lands on the Dashboard, fully signed in.

Optional **role mapping** can promote specific emails or whole domains to `operator` or `admin` automatically — see [Role mapping](#role-mapping) below.

---

## Google SSO — full walkthrough

### Phase 1 — Create OAuth credentials in Google Cloud Console

1. Go to **https://console.cloud.google.com** → pick or create a project.
2. Left sidebar → **APIs & Services → OAuth consent screen**.
   - Choose **External** (or **Internal** if you're on Google Workspace and want to restrict sign-in to your own domain).
   - **App name**: `Invenzo` · **Support email**: yours.
   - Scopes: leave default. Invenzo only requests `openid`, `email`, `profile`.
   - **Test users** (External only): add your own Gmail address while the app is in "Testing" mode. Without this, only listed emails can sign in.
   - Save → "Back to dashboard".
3. Left sidebar → **APIs & Services → Credentials** → **+ Create Credentials → OAuth client ID**.
   - **Application type**: `Web application`
   - **Name**: `Invenzo Web Client`
   - **Authorized redirect URIs** — for now, add a placeholder (you'll update it in Phase 2):
     ```
     https://<your-invenzo-host>/api/v1/sso/oidc/00000000-0000-0000-0000-000000000000/callback
     ```
   - Click **Create**.
4. Google shows a modal with **Client ID** and **Client secret** — copy both into a temp file. You won't see the secret again after closing this modal.

### Phase 2 — Configure the provider in Invenzo

1. Sign into Invenzo as an admin.
2. Navigate to **Settings → Security tab → SSO Providers** section.
3. Click **+ Add Provider**.
4. Fill in:

| Field | Value |
|---|---|
| **Name** | `Google` (this label appears on the login button: "Continue with Google") |
| **Provider type** | `OIDC` |
| **Auto-provision users** | ✅ on (creates Invenzo users automatically on first sign-in) |
| **Default role** | `viewer` (safest — admins can promote individuals later) |
| **Active** | ✅ on |

5. In the **OIDC config** form, set:

| Key | Value |
|---|---|
| `client_id` | (paste from Phase 1.4) |
| `client_secret` | (paste from Phase 1.4) |
| `auth_endpoint` | `https://accounts.google.com/o/oauth2/v2/auth` |
| `token_endpoint` | `https://oauth2.googleapis.com/token` |
| `userinfo_endpoint` | `https://openidconnect.googleapis.com/v1/userinfo` |
| `scopes` | `openid email profile` |
| `redirect_uri` | leave blank — Invenzo auto-builds this from the provider's UUID + your hostname |

6. **Save**. Invenzo creates the provider row and assigns it a UUID.
7. Back on the SSO providers list, click your new `Google` provider → copy the **Callback URL** Invenzo displays. It looks like:
   ```
   https://invenzo.yourcompany.com/api/v1/sso/oidc/4f8a0c…/callback
   ```

### Phase 3 — Update Google with the real callback URL

1. Back to Google Cloud Console → **APIs & Services → Credentials** → click your `Invenzo Web Client`.
2. Under **Authorized redirect URIs**, replace the placeholder with the **exact** Callback URL you copied in Phase 2.7.
3. **Save**.

### Phase 4 — Test the flow

1. Sign out of Invenzo. You're back on the Login page.
2. The right-side form now shows a button labelled **`Continue with Google`** (or whatever Name you gave the provider).
3. Click it → Google's account picker opens → choose your Gmail → consent → Google redirects back to Invenzo.
4. Invenzo provisions your user with the `default_role` (`viewer`) and signs you in.

### Phase 5 — Promote yourself to admin

Auto-provision lands new SSO users as `viewer` by design. To grant yourself admin rights:

1. Sign in via the **password** of the original install-time admin (the one you set during `install.sh`).
2. **Settings → Users** → find your Gmail address → set role to `admin` → **Save**.
3. Sign out → sign in via Google again → you now have admin.

---

## Role mapping

If you want every `@yourcompany.com` employee to land as `operator` (or specific people as `admin`) on first sign-in instead of getting stuck at `viewer`, configure **Role Mapping** on the provider:

```json
{
  "by_email": {
    "ceo@yourcompany.com": "admin",
    "you@gmail.com": "admin"
  },
  "by_email_domain": {
    "yourcompany.com": "operator"
  }
}
```

Resolution order (most specific wins):

1. `by_email` exact match → uses that role
2. `by_email_domain` match → uses that role
3. Fall back to `default_role` on the provider

---

## Microsoft Entra ID (Azure AD) — quick reference

Same flow as Google, just different endpoints. In Phase 1, register an **App registration** in Entra ID (Azure Portal → Microsoft Entra ID → App registrations → New registration):

| Invenzo field | Value |
|---|---|
| `client_id` | Application (client) ID from the App registration |
| `client_secret` | New client secret from **Certificates & secrets** |
| `auth_endpoint` | `https://login.microsoftonline.com/<tenant-id>/oauth2/v2.0/authorize` |
| `token_endpoint` | `https://login.microsoftonline.com/<tenant-id>/oauth2/v2.0/token` |
| `userinfo_endpoint` | `https://graph.microsoft.com/oidc/userinfo` |
| `scopes` | `openid email profile` |

Replace `<tenant-id>` with your tenant ID (or `common` for multi-tenant). Set the App registration's **Redirect URI** to the same Invenzo Callback URL pattern as Google.

---

## Okta — quick reference

In Okta Admin → **Applications → Create App Integration → OIDC – OpenID Connect → Web Application**.

| Invenzo field | Value |
|---|---|
| `client_id` | from Okta app's General settings |
| `client_secret` | from Okta app's General settings |
| `auth_endpoint` | `https://<your-okta-domain>/oauth2/default/v1/authorize` |
| `token_endpoint` | `https://<your-okta-domain>/oauth2/default/v1/token` |
| `userinfo_endpoint` | `https://<your-okta-domain>/oauth2/default/v1/userinfo` |
| `scopes` | `openid email profile` |

`<your-okta-domain>` looks like `acmecorp.okta.com`. Set Okta's **Sign-in redirect URI** to Invenzo's Callback URL.

---

## SAML 2.0 (ADFS, PingFederate, Shibboleth)

Use **Provider type = SAML** in Invenzo. Required `config` keys:

| Key | What goes here |
|---|---|
| `idp_entity_id` | The IdP's EntityID (from its metadata XML) |
| `idp_sso_url` | The IdP's SAML SSO URL (HTTP-POST or HTTP-Redirect binding) |
| `idp_x509_cert` | The IdP's signing certificate (PEM, no header lines) |
| `sp_entity_id` | What Invenzo presents as its EntityID — typically `https://<your-invenzo-host>/api/v1/sso/saml` |

The Invenzo callback (ACS URL) is `https://<your-invenzo-host>/api/v1/sso/saml/<provider_id>/acs`. Register that with your IdP as the Assertion Consumer Service URL.

---

## LDAP / Active Directory

Use **Provider type = LDAP**. Unlike OIDC/SAML, LDAP authenticates the user's **existing username/password** that they type into the Invenzo login form (no redirect to an IdP). Required `config`:

| Key | Value |
|---|---|
| `server_url` | `ldaps://dc1.yourcompany.com:636` (use LDAPS, not LDAP) |
| `bind_dn` | `CN=Invenzo Service,OU=Service Accounts,DC=yourcompany,DC=com` |
| `bind_password` | (the service account's password — encrypted at rest in Invenzo) |
| `user_search_base` | `OU=Users,DC=yourcompany,DC=com` |
| `user_search_filter` | `(sAMAccountName={username})` |
| `email_attr` | `mail` |
| `name_attr` | `displayName` |

The login page detects LDAP providers and shows them as buttons that consume whatever's already in the username + password fields.

---

## Common gotchas

- **`redirect_uri_mismatch` from Google/Azure/Okta** — the IdP requires an EXACT match including protocol (`https://`), no trailing slash, and the exact provider UUID. Copy the Callback URL from Invenzo's provider detail page; don't hand-type it.
- **HTTPS required by all major IdPs except for `localhost`** — Invenzo behind Caddy auto-TLS (the default) or any real cert is fine. Plain HTTP redirect URIs only work for `http://localhost`.
- **Test mode (Google External)** — while the OAuth consent screen is in "Testing" status, only listed Test users can sign in. Either add yourself to Test users OR click "Publish app" in the OAuth consent screen (Google won't review unless you request sensitive scopes).
- **Workspace-only sign-in** — set Google OAuth consent screen → Application type → **Internal**. Restricts sign-in to your `@yourcompany.com` accounts; no Test users list needed.
- **First SSO user has no admin** — auto-provision creates `viewer`. The original `install.sh` admin must promote you (Settings → Users → set role to `admin`).
- **`invalid_grant` on token exchange** — usually means the `client_secret` doesn't match. Re-copy from Google Cloud Console → Credentials → your OAuth client → "Reset secret" if needed.
- **User can sign in via SSO but everything is read-only** — they're at `viewer`. Either promote them individually in Settings → Users, or set up Role Mapping (see above) to bump them to `operator` automatically.

---

## What's stored where

| Item | Where it lives | Encrypted? |
|---|---|---|
| Provider config (client_id, endpoints) | `sso_providers.config` JSONB | No (not secrets) |
| `client_secret` | `sso_providers.config.client_secret` | Yes — Invenzo's `VAULT_KEY` (AES-256-GCM) |
| LDAP `bind_password` | `sso_providers.config.bind_password` | Yes — same VAULT_KEY |
| SAML `x509_cert` | `sso_providers.config.x509_cert` | Yes (treated as a secret to prevent disclosure) |
| User's IdP token (after exchange) | Not stored — Invenzo immediately mints its own JWT |
| Created user record | `users` table — same as a password user, just no password_hash |

Rotating `VAULT_KEY` re-encrypts all SSO secrets automatically (see [SECRETS.md § Rotation](SECRETS.md#6-rotation)).

---

## API endpoints (for advanced wiring)

If you'd rather create the provider via API instead of the UI:

```bash
# Create OIDC provider (admin token required)
curl -X POST https://invenzo.yourcompany.com/api/v1/sso/providers \
  -H "Authorization: Bearer <admin_jwt>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Google",
    "provider_type": "oidc",
    "default_role": "viewer",
    "auto_provision": true,
    "is_active": true,
    "config": {
      "client_id": "...",
      "client_secret": "...",
      "auth_endpoint": "https://accounts.google.com/o/oauth2/v2/auth",
      "token_endpoint": "https://oauth2.googleapis.com/token",
      "userinfo_endpoint": "https://openidconnect.googleapis.com/v1/userinfo",
      "scopes": "openid email profile"
    }
  }'

# Then `GET /api/v1/sso/providers/<id>` returns the generated UUID + Callback URL.
```

Login flow URLs:

| Endpoint | Purpose |
|---|---|
| `GET /api/v1/sso/oidc/<provider_id>/login` | Browser redirects here from the Login button → goes to IdP |
| `GET /api/v1/sso/oidc/<provider_id>/callback?code=...&state=...` | IdP redirects back here → Invenzo issues a one-time code |
| `POST /api/v1/sso/exchange` | UI POSTs `{ code }` → Invenzo returns `{ access_token, role, username }` |

---

<p style="text-align:center;color:#888;font-size:12px;margin-top:40px">
  Invenzo ITAM · Self-hosted IT Asset Management · Commercial License
</p>
