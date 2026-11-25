# 02 – OIDC Integration

**Goal:** sign a user in to an application over OpenID Connect and inspect the ID token Okta issues.

The "application" is **OIDC Debugger** (oidcdebugger.com), a tool that acts as a relying party and displays the token it receives.

## 1. App integration

**Applications → Create App Integration → OIDC – Web Application**

| Setting | Value | Reason |
|---|---|---|
| Name | OIDC Debugger Lab | — |
| Client ID | Okta-generated | No legacy system requires a custom ID |
| Grant types | Authorization Code; Implicit (hybrid) → Allow ID Token | Implicit used **only** to view the ID token directly |
| Client Credentials | Off | Machine-to-machine; no user involved |
| Sign-in redirect URI | `https://oidcdebugger.com/debug` | Replaced the localhost default (least privilege) |
| Sign-out redirect URI | *(removed)* | Not used |
| Assignment | Group: **Engineering** | Group-based access control |
| Self Service | Disabled | Access is admin-granted, not user-requested |

## 2. Groups claim

**Security → API → Authorization Servers → default → Claims**

| Name | Include in | Value type | Filter |
|---|---|---|---|
| groups | ID Token, Always | Groups | Matches regex `.*` |

## 3. Authentication policy

Added a rule **above** the catch-all in the app's authentication policy:

- **Rule:** Engineering access
- **Condition:** group membership includes Engineering
- **Access:** allowed after successful authentication
- **Factors:** any 2 factor types; the org accepts only phishing-resistant possession factors (Okta Verify FastPass or passkey)

## 4. Authorization server access policy

**Security → API → Authorization Servers → default → Access Policies**

- **Policy:** assigned to the OIDC Debugger Lab client
- **Rule:** grant types Authorization Code + Implicit; any user assigned the app; any scopes

## 5. Request

| Field | Value |
|---|---|
| Authorize URI | `https://<org>.okta.com/oauth2/default/v1/authorize` |
| Scope | `openid profile email` |
| Response type | `id_token` |
| Response mode | `form_post` |

## 6. Results

### ✅ Alice (Engineering): token issued

Decoded at jwt.io:

| Claim | Meaning |
|---|---|
| `iss` | Issuer: the Okta default authorization server |
| `aud` | Audience: this app's Client ID |
| `sub` | Stable unique user ID |
| `exp` / `iat` | Expiry / issued-at timestamps |
| `email`, `name` | Profile claims from the `email` and `profile` scopes |
| `groups` | Group membership (includes `Engineering`) |
| `nonce` | Ties the token to this request (replay protection) |

### ❌ Carol (Sales): denied

Carol is not assigned the app, so Okta blocked the request. This is the expected negative test.

## 7. Takeaway: three layers of access control

```
Request ──▶ [1] App assignment ──▶ [2] Authentication policy ──▶ [3] Auth server access policy ──▶ Token
              (Carol fails here)      (how strong is MFA?)          (which grants/scopes allowed?)
```

📸 *Screenshots: `screenshots/oidc-app-settings.png`, `screenshots/id-token-decoded.png`, `screenshots/carol-denied.png`*
