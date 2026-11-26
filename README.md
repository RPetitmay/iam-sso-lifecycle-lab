# IAM SSO & Lifecycle Lab

A hands-on identity and access management lab built on **Okta (Integrator Free Plan)**. It covers single sign-on over OIDC and SAML, MFA and authentication policy, group-based access control, and automated joiner/mover/leaver provisioning.

> All users, groups, and data in this lab are fictional. Org URLs, client IDs, and tokens are redacted.

## Contents

- [Part 1 – Tenant and Directory Setup](#part-1--tenant-and-directory-setup)
- [Part 2 – OIDC Integration](#part-2--oidc-integration)
- [Troubleshooting Log](#troubleshooting-log)
- [Production Considerations](#production-considerations)

## Goals

- Stand up an identity provider (IdP) with a structured directory
- Integrate applications using **OIDC** and **SAML**
- Enforce **phishing-resistant MFA** and group-based access policies
- Automate the **identity lifecycle** (joiner / mover / leaver)
- Document design decisions and troubleshooting like a production rollout

## Progress

| Phase | Status |
|---|---|
| Tenant, custom domain, and directory | ✅ Complete |
| OIDC integration (OIDC Debugger) | ✅ Complete |
| SAML integration | ⬜ Planned |
| Conditional access / authentication policies by group | ⬜ Planned |
| Joiner / mover / leaver automation (SCIM / API) | ⬜ Planned |

## Architecture

```
 ┌──────────────┐   1. sign-in request    ┌──────────────────────┐
 │   User        │ ─────────────────────▶ │  Okta (IdP)          │
 │ (Alice, Eng)  │                         │  ─ Directory/Groups  │
 └──────────────┘ ◀── 3. ID token ──────── │  ─ Auth policies     │
        │                                  │  ─ Auth server       │
        │ 4. token presented               └──────────────────────┘
        ▼                                            ▲
 ┌──────────────┐        2. MFA (FastPass / passkey) │
 │ App (SP)      │ ───────────────────────────────────┘
 │ OIDC Debugger │
 └──────────────┘
```

## Directory Design

| Group | Purpose | Example user |
|---|---|---|
| Engineering | Access to engineering apps | Alice Chen, Bob Martinez |
| Sales | Sales tools only | Carol Smith |
| HR | HR systems; future source of lifecycle events | Evan Brooks |
| IT-Admins | Elevated access; stricter MFA | Dana Lee |

## Key Concepts Demonstrated

- **Authentication vs. authorization:** Okta proves *who* Alice is (password + phishing-resistant MFA), then policy decides *what* she can access.
- **Three layers of access control.** A request must pass all three:
  1. **App assignment:** is the user assigned the app (via group)?
  2. **Authentication policy:** how strongly must they authenticate?
  3. **Authorization server access policy:** which clients, grant types, and scopes may receive tokens?
- **Least privilege:** only Engineering is assigned the app; unused redirect URIs removed; Self Service left disabled.

---

## Part 1 – Tenant and Directory Setup

### 1. Custom domain and admin email

Okta's Integrator Free Plan requires a **business email address** at signup; personal providers such as Gmail are rejected. To keep the lab separate from any employer, I:

1. Registered a custom domain through **Namecheap** (WHOIS privacy on, 2FA enabled on the registrar account).
2. Configured **Namecheap email forwarding**:
   - `admin@<domain>` → personal inbox (Okta admin account)
   - `*@<domain>` catch-all → personal inbox (receives mail for test users)
3. Verified delivery with a test email before signing up.

**Why it matters:** the admin email is the recovery path for the whole tenant. Whoever controls the domain controls the IdP, so the registrar account is protected with MFA.

### 2. Okta org

- Plan: **Okta Integrator Free Plan** (not the 30-day trial)
- Admin MFA: **Okta Verify** enrolled at first login

### 3. Groups

Created under **Directory → Groups**:

| Group | Description |
|---|---|
| Engineering | Engineering staff; assigned engineering apps |
| Sales | Sales staff |
| HR | HR staff; future authoritative source for lifecycle |
| IT-Admins | IT administrators; target for stricter policy |

### 4. Users

Created under **Directory → People → Add person**, activated immediately with an admin-set password:

| Name | Username | Group |
|---|---|---|
| Alice Chen | alice.chen@<domain> | Engineering |
| Bob Martinez | bob.martinez@<domain> | Engineering |
| Kenny Vital | kenny.vital@<domain> | Engineering |
| Carol Smith | carol.smith@<domain> | Sales |
| Evan Brooks | evan.brooks@<domain> | HR |
| Dana Lee | dana.lee@<domain> | IT-Admins |

### 5. MFA at first sign-in

The org's default **authenticator enrollment policy** required each user to enroll a security method on first login. Alice was prompted to enroll before reaching her dashboard, which confirms MFA is enforced from day one.

📸 *Screenshots: `screenshots/groups.png`, `screenshots/people.png`, `screenshots/mfa-enrollment.png`*

---

## Part 2 – OIDC Integration

**Goal:** sign a user in to an application over OpenID Connect and inspect the ID token Okta issues.

The "application" is **OIDC Debugger** (oidcdebugger.com), a tool that acts as a relying party and displays the token it receives.

### 1. App integration

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

### 2. Groups claim

**Security → API → Authorization Servers → default → Claims**

| Name | Include in | Value type | Filter |
|---|---|---|---|
| groups | ID Token, Always | Groups | Matches regex `.*` |

### 3. Authentication policy

Added a rule **above** the catch-all in the app's authentication policy:

- **Rule:** Engineering access
- **Condition:** group membership includes Engineering
- **Access:** allowed after successful authentication
- **Factors:** any 2 factor types; the org accepts only phishing-resistant possession factors (Okta Verify FastPass or passkey)

### 4. Authorization server access policy

**Security → API → Authorization Servers → default → Access Policies**

- **Policy:** assigned to the OIDC Debugger Lab client
- **Rule:** grant types Authorization Code + Implicit; any user assigned the app; any scopes

### 5. Request

| Field | Value |
|---|---|
| Authorize URI | `https://<org>.okta.com/oauth2/default/v1/authorize` |
| Scope | `openid profile email` |
| Response type | `id_token` |
| Response mode | `form_post` |

### 6. Results

#### ✅ Alice (Engineering): token issued

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

#### ❌ Carol (Sales): denied

Carol is not assigned the app, so Okta blocked the request. This is the expected negative test.

### 7. Takeaway: three layers of access control

```
Request ──▶ [1] App assignment ──▶ [2] Authentication policy ──▶ [3] Auth server access policy ──▶ Token
              (Carol fails here)      (how strong is MFA?)          (which grants/scopes allowed?)
```

📸 *Screenshots: `screenshots/oidc-app-settings.png`, `screenshots/id-token-decoded.png`, `screenshots/carol-denied.png`*

---

## Troubleshooting Log

Real issues hit while building this lab: the symptom, the root cause, and the fix.

> **Method:** after each failure I checked **Reports → System Log** for the failed event's `reason` field before changing any settings.

### 1. Okta signup rejected personal email

- **Symptom:** "Business email required" on the Integrator Free Plan signup.
- **Cause:** Okta free orgs only accept corporate-domain email addresses.
- **Fix:** registered a custom domain and set up email forwarding (`admin@<domain>` → personal inbox). Avoided using a work email so the lab stays independent of any employer.

### 2. Engineering group not shown during app assignment

- **Symptom:** the group picker in the app wizard appeared empty.
- **Cause:** the field is search-as-you-type and shows nothing until you type.
- **Fix:** typed the group name. Alternatively, skip assignment in the wizard and use **App → Assignments → Assign to Groups**.

### 3. Test user forced to set up security methods

- **Symptom:** Alice was prompted to enroll an authenticator on first sign-in.
- **Cause:** expected behavior; the default authenticator enrollment policy requires MFA.
- **Fix:** enrolled an authenticator. MFA was left on deliberately.

### 4. "You are not allowed to access this app"

- **Symptom:** Alice authenticated successfully but was blocked from the app.
- **Cause:** the app's **authentication policy** had no rule allowing her; the catch-all rule did not grant access.
- **Fix:** added an `Engineering access` rule **above** the catch-all: group = Engineering, access allowed after 2-factor authentication.
- **Lesson:** Okta evaluates rules top-down and the first match wins.

### 5. Phishing-resistant factors only

- **Symptom:** the rule summary showed only Okta Verify FastPass or passkey as acceptable second factors.
- **Cause:** the org accepts only phishing-resistant possession factors. FastPass needs Okta Verify on the *same device* as the browser.
- **Fix:** used a passkey / FastPass-capable device. Relaxing to push notifications is possible but was not needed.

### 6. 400 Bad Request at the authorize endpoint

- **Symptom:** "400 Bad Request – Your request resulted in an error."
- **Cause:** the OIDC request did not match the app configuration (Authorize URI, redirect URI, or response type).
- **Fix:** used the non-admin org domain with `/oauth2/default/v1/authorize`, confirmed by loading `/.well-known/openid-configuration`.

### 7. "Policy evaluation failed for this request"

- **Symptom:** the request got past app sign-on but no token was issued.
- **Cause:** the **default authorization server** had no access policy rule covering this client and the **Implicit** grant type.
- **Fix:** added an access policy assigned to the client, with a rule allowing Authorization Code + Implicit for any user assigned the app.
- **Lesson:** app sign-on policy and authorization server policy are separate layers, and both must allow the request.

### 8. Self Service warning

- **Symptom:** "Self Service cannot be enabled for this app because it is not enabled for the org."
- **Cause:** informational; Self Service is off org-wide.
- **Fix:** none needed. Left disabled on purpose, since access is granted by group assignment.

### Quick reference

| Error | Layer | Where to look |
|---|---|---|
| Not allowed to access this app | Assignment / authentication policy | App → Assignments; Security → Authentication Policies |
| 400 Bad Request | Request configuration | OIDC Debugger fields vs. app settings |
| Policy evaluation failed | Authorization server | Security → API → Authorization Servers → Access Policies |

---

## Production Considerations

- Use **Authorization Code + PKCE** instead of the implicit flow (implicit was used here only to inspect ID tokens easily).
- Manage Okta configuration as code (e.g., the Okta Terraform provider) for review and repeatability.
- Separate admin accounts from day-to-day accounts; require phishing-resistant MFA for all admins.
- Forward the Okta System Log to a SIEM for alerting on failed policy evaluations and admin changes.
- Keep a break-glass admin account with documented recovery steps.

## Tools

Okta Integrator Free Plan · OIDC Debugger · jwt.io · Namecheap (custom domain + email forwarding)
