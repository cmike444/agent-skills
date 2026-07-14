# Authentication (OAuth2)

All tastytrade API access uses **OAuth2 access tokens** as `Authorization: Bearer <token>`.
Access tokens live **15 minutes**; refresh tokens are **long-lived and never expire**. There is
no way to extend an access token — refresh to get a new one.

Token endpoint: `POST https://api.tastyworks.com/oauth/token`
(Sandbox: `POST https://api.cert.tastyworks.com/oauth/token`)

Valid scopes: `read`, `trade`, `openid`. `read` and `trade` are "sensitive" scopes and require
the customer to have **2FA** enabled to grant them.

## Table of contents
- [Personal app (single-user) setup](#personal-app-single-user-setup)
- [Get / refresh an access token](#get--refresh-an-access-token)
- [Third-party apps: authorization-code flow](#third-party-apps-authorization-code-flow)
- [Sandbox signup & email confirmation](#sandbox-signup--email-confirmation)
- [Common auth errors](#common-auth-errors)

## Personal app (single-user) setup

For accessing your own account you do **not** need the full authorization-code flow.

1. **Create an OAuth application** at `my.tastytrade.com` → Manage → My Profile → API →
   OAuth Applications → **+ New OAuth client**. Provide full redirect URI(s) (must be complete
   URIs, `https://www.example.com` not `example.com`) and requested scopes (`read`, `trade`,
   `openid`). On create you get a **Client ID** and **Client Secret** — the secret is shown
   **once**; store it securely. Regenerating the secret invalidates active grants.
2. **Generate a personal grant:** on the same page click **Manage → Create Grant**. This yields
   a **refresh token**. If lost/compromised, delete the grant and create a new one (new refresh
   token). There is no other way to recover it.
3. Use the refresh token + client secret to mint access tokens (below).

## Get / refresh an access token

Both minting and refreshing use `grant_type=refresh_token`:

```
POST /oauth/token
Content-Type: application/json

{
  "grant_type": "refresh_token",
  "refresh_token": "<your refresh token>",
  "client_secret": "<your client secret>"
}
```

Response contains a fresh `access_token`. Send it as `Authorization: Bearer <access_token>` on
every subsequent request. When it 401s (after ~15 min), refresh again. (Ref: RFC 6749 §6.)

## Third-party apps: authorization-code flow

Only for **trusted partner** apps serving other tastytrade users (contact
api.support@tastytrade.com for verification). Personal apps are restricted to your own account
until approved.

1. **Authorize:** redirect the user to `https://my.tastytrade.com/auth.html`
   (Sandbox: `https://cert-my.staging-tasty.works/auth.html`) with query params `client_id`
   (required), `redirect_uri` (required), `response_type` (required), `scope` (optional),
   `state` (optional). The user logs in (with 2FA for `read`/`trade`) and approves.
2. **Receive the code:** tastytrade redirects to your `redirect_uri` with `code` and `state`.
3. **Exchange the code:**

   ```
   POST /oauth/token
   {
     "grant_type": "authorization_code",
     "code": "<authorization code>",
     "client_id": "...",
     "client_secret": "...",
     "redirect_uri": "..."
   }
   ```

   Response: `access_token`, `refresh_token` (never expires), `token_type: Bearer`,
   `expires_in` (seconds), and `id_token` (only with the `openid` scope).
4. **Refresh** later with `grant_type=refresh_token` as above.

## Sandbox signup & email confirmation

Create a sandbox user on the tastytrade Sandbox page, then create a customer/account under it.
OAuth2 is fully supported in sandbox (manage your app from the Sandbox page tools).

**Email must be confirmed within 3 days** or calls fail with `unconfirmed_user`. Re-request
confirmation:

```
POST https://api.cert.tastyworks.com/confirmation
Content-Type: application/json

{ "email": "<your email>" }
```

Then click the emailed link. Sandbox users can't be deleted; reset the sandbox password via the
"Reset it here" link on the Sandbox page.

## Common auth errors

| Symptom | Cause / fix |
|---------|-------------|
| `401` with nginx HTML body | Missing/malformed `User-Agent` header. Must be `<product>/<version>`. |
| `401` on a valid-looking request | Access token expired (15 min) or absent. Refresh and resend. |
| `invalid_credentials` | Wrong environment — sandbox creds against production or vice versa. Check the base URL. |
| `unconfirmed_user` | Email not confirmed within 3 days. POST to `/confirmation`. |
| Requests time out entirely | IP blocked ~8h after too many failed logins. Email api.support@tastytrade.com. |
| `quote_streamer.customer_not_found_error` | You registered a username but never opened a brokerage account; required to stream quotes. |
