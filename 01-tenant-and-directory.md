# 01 – Tenant and Directory Setup

## 1. Custom domain and admin email

Okta's Integrator Free Plan requires a **business email address** at signup; personal providers such as Gmail are rejected. To keep the lab separate from any employer, I:

1. Registered a custom domain through **Namecheap** (WHOIS privacy on, 2FA enabled on the registrar account).
2. Configured **Namecheap email forwarding**:
   - `admin@<domain>` → personal inbox (Okta admin account)
   - `*@<domain>` catch-all → personal inbox (receives mail for test users)
3. Verified delivery with a test email before signing up.

**Why it matters:** the admin email is the recovery path for the whole tenant. Whoever controls the domain controls the IdP, so the registrar account is protected with MFA.

## 2. Okta org

- Plan: **Okta Integrator Free Plan** (not the 30-day trial)
- Admin MFA: **Okta Verify** enrolled at first login

## 3. Groups

Created under **Directory → Groups**:

| Group | Description |
|---|---|
| Engineering | Engineering staff; assigned engineering apps |
| Sales | Sales staff |
| HR | HR staff; future authoritative source for lifecycle |
| IT-Admins | IT administrators; target for stricter policy |

## 4. Users

Created under **Directory → People → Add person**, activated immediately with an admin-set password:

| Name | Username | Group |
|---|---|---|
| Alice Chen | alice.chen@<domain> | Engineering |
| Bob Martinez | bob.martinez@<domain> | Engineering |
| Carol Smith | carol.smith@<domain> | Sales |
| Evan Brooks | evan.brooks@<domain> | HR |
| Dana Lee | dana.lee@<domain> | IT-Admins |

## 5. MFA at first sign-in

The org's default **authenticator enrollment policy** required each user to enroll a security method on first login. Alice was prompted to enroll before reaching her dashboard, which confirms MFA is enforced from day one.

📸 *Screenshots: `screenshots/groups.png`, `screenshots/people.png`, `screenshots/mfa-enrollment.png`*
