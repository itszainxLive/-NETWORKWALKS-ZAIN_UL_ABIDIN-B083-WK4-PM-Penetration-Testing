# Penetration Testing Report — Mediroza General Hospital

**Batch:** B083 | Week 4
**Author:** Zain ul Abidin
**Target:** https://medirozahospital.com
**Engagement Type:** Black-box Penetration Test
**Duration:** 5 Days
**Authorization:** Written permission granted by the client to NetworkWalks for this security assessment. All testing was limited to the target domain; no social engineering or denial-of-service techniques were used.

> Screenshot paths in this report point to the individual milestone folders (`module1-initial-access/screenshots/`, `module2-cracking-encryption/screenshots/`, `module3-data-exposure/screenshots/`). Adjust the paths below if your repository uses different folder names.

---

## 01 — Executive Summary

A black-box penetration test was carried out against the Mediroza General Hospital Patient Portal (`medirozahospital.com`) over a 5-day engagement. Starting from unauthenticated reconnaissance alone, the assessment progressed through three chained findings that together resulted in a **complete compromise of patient confidentiality and exposure of the hospital's internal HR and financial data**:

1. A **SQL Injection vulnerability** in the Patient Portal login form allowed authentication to be bypassed entirely, granting access to the "My lab reports" area and 3 confidential, password-protected patient pathology reports.
2. All 3 retrieved PDF reports were protected with **weak, dictionary-crackable passwords**, allowing their full contents to be recovered in minutes using a standard wordlist.
3. Metadata left behind in one of the decrypted PDFs disclosed the existence of a **legacy, unauthenticated database backup** (`/old/mediroza_db_backup_2019.sql`) on the production server, exposing salary and national ID data for all 30 hospital staff, plus the hospital's full shareholder register.

**Overall risk to the client is rated Critical.** An unauthenticated external attacker could, using only a web browser and publicly available tools, fully bypass patient authentication, recover confidential medical records, and exfiltrate sensitive HR and corporate ownership data — all without triggering typical access controls.

---

## 02 — Scope and Methodology

**Target:** `https://medirozahospital.com` (Patient Portal, `/patient/*`)

**Scope:** Full black-box penetration test limited to the target domain. Identify vulnerabilities, exploit them to demonstrate real impact, and document all findings.

**Tools Used:**

| Tool | Purpose |
|---|---|
| `curl` | HTTP reconnaissance, manual request crafting, SQL injection testing, directory/file probing |
| `grep` | Parsing HTML responses for links, error strings, and authentication behaviour |
| `pdf2john.pl` / `John the Ripper` | Initial PDF hash extraction attempt |
| `pdfcrack` | Password recovery for encrypted PDF reports |
| `qpdf` | PDF decryption using recovered passwords |
| `exiftool` | Metadata analysis of decrypted PDF files |

**Methodology (approach taken):**

1. **Reconnaissance** — mapped the target's redirect chain and enumerated exposed links/assets on the public-facing Patient Portal page.
2. **Authentication Analysis** — analysed the login form's error-handling behaviour to look for information leakage.
3. **Input Handling Testing** — tested the login form for SQL Injection, escalating from an error-based probe to a full authentication bypass.
4. **Post-Exploitation Data Recovery** — cracked the encryption on all files retrieved through the bypass.
5. **Deep Analysis** — re-examined all retrieved evidence (including file metadata) for secondary exposures, which led to discovery of an unauthenticated backup file.

**Limitations Encountered:**

- `John the Ripper`'s default PDF format did not reliably load the extracted hash (`No password hashes loaded`) despite a structurally valid hash, and `hashcat` could not be used in the test VM due to no available OpenCL/GPU runtime. `pdfcrack`, run directly against the PDF files, was used instead and successfully recovered all 3 passwords.
- Directory listing was disabled on `/patient/reports/` and `/patient/error_log`, requiring the secondary exposure to be found via metadata analysis rather than direct enumeration.

---

## 03 — Findings and Proof of Exploitation

### Finding 1 — SQL Injection / Authentication Bypass (Milestone 1)

**Location:** `POST /patient/login.php` (`username`, `password` parameters)

Reconnaissance identified the Patient Portal's structure and exposed internal paths, including `login.php`.

![Reconnaissance](module1-initial-access/01-reconnaissance.png)
![Exposed Entry Points](module1-initial-access/02-exposed-entry-points.png)

Submitting a single quote (`'`) in the `username` field returned a distinct, informative error message, indicating unsafe handling of user input server-side.

![Authentication Analysis](module1-initial-access/03-authentication-analysis.png)

Testing the field directly with `curl` confirmed a raw MySQL syntax error was being returned to the client, confirming SQL Injection:

```
Warning: mysqli_query(): You have an error in your SQL syntax; check the
manual that corresponds to your MySQL server version for the right syntax
to use near '''' at line 1
```

Injecting a classic authentication-bypass payload into both `username` and `password` (`' `) returned the Patient Portal's authenticated content.

![SQL Injection Testing](module1-initial-access/04-input-handling-sql-error.png)
![Proof of Access - Login Page](module1-initial-access/05-proof-of-access-login.png)

This granted unauthenticated access to the "My lab reports" area, exposing 3 confidential patient pathology reports (S. Dlamini, P. Reddy, E. Thompson).

![Lab Reports Access](module1-initial-access/06-lab-reports-access.png)

---

### Finding 2 — Weak Encryption on Confidential Patient Reports (Milestone 2)

**Location:** `patient_report_1.pdf`, `patient_report_2.pdf`, `patient_report_3.pdf`

All 3 PDF reports retrieved via Finding 1 were password-protected (Standard V2.3, 128-bit encryption). Each file's password was recovered using `pdfcrack` against the `rockyou.txt` wordlist within minutes:

| File | Password Recovered |
|---|---|
| `patient_report_1.pdf` | `123456` |
| `patient_report_2.pdf` | `password` |
| `patient_report_3.pdf` | `!@#$%^&` |

![Password Cracking Evidence](module2-cracking-encryption/m2-report1-cracking.png)

The weak, dictionary-guessable passwords meant the encryption provided negligible real-world protection for the patient data inside.

![Decrypted Content](module2-cracking-encryption/m2-report1-opened.png)

---

### Finding 3 — Unauthenticated Exposure of HR and Shareholder Database Backup (Milestone 3)

**Location:** `https://medirozahospital.com/old/mediroza_db_backup_2019.sql`

Metadata analysis of the decrypted PDF reports revealed a `Comments` field left by an internal staff member in `patient_report_3.pdf`:

```
Author   : j.malik
Comments : DB backup moved to /old before site migration, do not delete
```

![Metadata Comment Leak](module3-data-exposure/m3-metadata-comment.png)

Requesting the disclosed path returned a full, unauthenticated directory listing:

![Directory Listing Exposed](module3-data-exposure/m3-old-directory-listing.png)

The listed file, `mediroza_db_backup_2019.sql`, was downloaded without any authentication challenge. It contained a plaintext dump of the `mediroza_hr` database, including:

- **`staff` table** — 30 records with full name, job title, department, email, phone number, national ID number, and monthly salary.
- **`shareholders` table** — 10 records with shareholder name, ownership percentage, shares held, and share class.

![SQL Backup Downloaded](module3-data-exposure/m3-sql-backup-downloaded.png)

---

## 04 — Risk Rating

| # | Finding | Severity | Justification |
|---|---|---|---|
| 1 | SQL Injection / Authentication Bypass | **Critical** | Allows complete, unauthenticated bypass of patient authentication and direct access to confidential medical records; trivially exploitable with a single crafted request. |
| 2 | Weak PDF Encryption on Patient Reports | **High** | Confidential medical data was protected only by dictionary-guessable passwords, providing negligible real-world confidentiality once the files were obtained. |
| 3 | Unauthenticated Database Backup Exposure | **Critical** | Full, unauthenticated download of a database backup containing staff national ID numbers, salaries, and corporate shareholder data — no exploitation skill required, only knowledge of the path. |

**Overall Engagement Risk: Critical.**

---

## 05 — Recommendations and Remediation

### Finding 1 — SQL Injection

- Use parameterised queries / prepared statements for all database access in `login.php` and across the application; never concatenate user input directly into SQL statements.
- Implement server-side input validation and a strict allow-list for authentication parameters.
- Disable verbose database error messages in production; log errors server-side only.
- Deploy a Web Application Firewall (WAF) rule set (e.g. OWASP CRS) as a defence-in-depth measure.

### Finding 2 — Weak PDF Encryption

- Enforce a strong, randomly generated password policy for all password-protected patient documents (minimum length, no dictionary words, no reused defaults such as `123456` or `password`).
- Consider moving away from PDF password protection alone and instead gate access to reports exclusively behind authenticated application sessions with server-side authorization checks.
- Where PDF encryption is still used, prefer AES-256 (Standard V5/R6) over the weaker legacy RC4-based Standard V2.3 scheme observed here.

### Finding 3 — Unauthenticated Backup Exposure

- Immediately remove `/old/mediroza_db_backup_2019.sql` and any other legacy files from the public webroot; store backups outside of any web-accessible directory.
- Disable directory listing (autoindex) server-wide unless explicitly required for a specific, intentional use case.
- Establish a formal decommissioning/migration checklist that includes secure deletion or off-server archival of legacy data, rather than leaving it "for later" in-place.
- Encrypt database backups at rest and restrict access via authentication, regardless of their location.
- Provide staff with guidance on not leaving operational notes (such as backup locations) inside metadata fields of documents that are served to end users.

---

## Summary

| Milestone | Deliverable | Status |
|---|---|---|
| M1 — Initial Access | Proof of access + 3 retrieved PDF files | ✅ Complete |
| M2 — Data Extraction | Recovered contents of all 3 encrypted files | ✅ Complete |
| M3 — Attack (Data Exposure) | Documented evidence of salary + shareholder exposure | ✅ Complete |
| M4 — Pentest Report | This report | ✅ Complete |
