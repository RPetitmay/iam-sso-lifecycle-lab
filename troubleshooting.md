# Troubleshooting Log

Real issues hit while building this lab: the symptom, the root cause, and the fix.

> **Method:** after each failure I checked **Reports → System Log** for the failed event's `reason` field before changing any settings.

---

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

---

## Quick reference

| Error | Layer | Where to look |
|---|---|---|
| Not allowed to access this app | Assignment / authentication policy | App → Assignments; Security → Authentication Policies |
| 400 Bad Request | Request configuration | OIDC Debugger fields vs. app settings |
| Policy evaluation failed | Authorization server | Security → API → Authorization Servers → Access Policies |
