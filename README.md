# IAM SSO & Lifecycle Lab
 
A hands-on identity and access management lab built on **Okta (Integrator Free Plan)**. It covers single sign-on over OIDC and SAML, MFA and authentication policy, group-based access control, and automated joiner/mover/leaver provisioning.
 
> All users, groups, and data in this lab are fictional. Org URLs, client IDs, and tokens are redacted.
 
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
## Documentation
 
- [01 – Tenant and Directory Setup](docs/01-tenant-and-directory.md)
- [02 – OIDC Integration](docs/02-oidc-integration.md)
- [Troubleshooting Log](docs/troubleshooting.md)
## Production Considerations
 
- Use **Authorization Code + PKCE** instead of the implicit flow (implicit was used here only to inspect ID tokens easily).
- Manage Okta configuration as code (e.g., the Okta Terraform provider) for review and repeatability.
- Separate admin accounts from day-to-day accounts; require phishing-resistant MFA for all admins.
- Forward the Okta System Log to a SIEM for alerting on failed policy evaluations and admin changes.
- Keep a break-glass admin account with documented recovery steps.
## Tools
 
Okta Integrator Free Plan · OIDC Debugger · jwt.io · Namecheap (custom domain + email forwarding)
 
