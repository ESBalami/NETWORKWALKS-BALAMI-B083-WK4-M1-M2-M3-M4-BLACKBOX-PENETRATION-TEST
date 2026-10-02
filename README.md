# NetworkWalks Cybersecurity Program – Week 4
## Penetration Testing Project: Mediroza General Hospital

**Author:** Emmanuel Balami
**Program:** NetworkWalks Cybersecurity Program (Batch B083)
**Week:** 04
**Target:** https://medirozahospital.com
**Engagement Type:** Authorised Black-box Penetration Test
**Duration:** 5 days

## Project Overview

This repository documents a full black-box penetration testing engagement against Mediroza General Hospital's public-facing website, conducted as part of Week 4 of the NetworkWalks Cybersecurity Program. The client provided written authorisation to test their web infrastructure, with testing strictly limited to the target domain — no social engineering, no denial-of-service, no testing outside the agreed scope.

The engagement followed four milestones:

- **M1 — Initial Access:** Attack the website and retrieve 3 confidential patient PDF lab reports.
- **M2 — Data Extraction:** Crack the encryption on all 3 retrieved files.
- **M3 — Critical Data Exposure:** Find the salaries of all hospital employees and the shareholder details of the hospital.
- **M4 — Pentest Report:** Write a professional penetration testing report for the client.

**Full written report:** [`report/Mediroza-Pentest-Report-Week4.docx`](report/Mediroza-Pentest-Report-Week4.docx)

## Summary of Findings

| # | Vulnerability | Risk |
|---|---|---|
| 1 | SQL Injection — authentication bypass on the Patient Portal login | **Critical** |
| 2 | Verbose database error disclosure (raw `mysqli_query()` errors shown to users) | Medium |
| 3 | Sensitive data exposure — unauthenticated database backup in an unprotected legacy directory | **Critical** |
| 4 | `robots.txt` disclosing sensitive paths (`/patient/`, `/staff/`, `/old/`) | Low |
| 5 | Weak, dictionary-crackable passwords protecting patient lab report PDFs | High |
| 6 | Bot-protection bypassable via a single static cookie | Low |

## M1 — Initial Access

Reconnaissance (`whois`, `gobuster`) located `robots.txt`, which explicitly disallowed three paths: `/patient/`, `/staff/`, and `/old/` — effectively handing over a map of the site's most sensitive areas.

The Patient Portal login form (`/patient/login.php`) was tested for SQL injection. A single quote in the username field triggered a raw, unhandled MySQL syntax error, confirming the input was being concatenated directly into a database query. Refining the payload to:

```
Username: admin' --
```

(a closing quote followed by a SQL comment marker with a trailing space) bypassed authentication completely, with no password required, landing directly on `/patient/portal.php` — exposing three patients' confidential lab reports.

**Evidence:** [`evidence/m1-initial-access/`](evidence/m1-initial-access/)
1. WHOIS lookup of the domain
2. gobuster scan locating `robots.txt` and `sitemap.xml`
3. Unauthenticated listing of the `/old/` directory
4. SQL syntax error confirming the injection point
5. Successful authentication bypass landing on the Patient Portal

## M2 — Data Extraction

The three PDFs retrieved from the Patient Portal were password-protected. All three were cracked using `pdfcrack` against the `rockyou.txt` wordlist:

| File | Patient | Password | Lab Ref |
|---|---|---|---|
| patient_report_1.pdf | Sipho Dlamini | `123456` | LR-2024-1187 |
| patient_report_2.pdf | Priya Reddy | `password` | LR-2024-1192 |
| patient_report_3.pdf | Emily Thompson | `!@#$%^&` | LR-2024-1205 |

Each report contains full pathology results tied to an identifiable patient (name, DOB, patient ID, referring doctor, and flagged abnormal results).

**Evidence:** [`evidence/m2-data-extraction/`](evidence/m2-data-extraction/)
1. `pdfcrack` recovering all three passwords
2. Recovered report — Sipho Dlamini (elevated white cell count)
3. Recovered report — Priya Reddy (elevated cholesterol markers)
4. Recovered report — Emily Thompson (low haemoglobin, ferritin, vitamin D)

## M3 — Critical Data Exposure

Independent of the login bypass, the `/old/` directory disclosed by `robots.txt` had directory listing enabled, exposing an unauthenticated database backup: `mediroza_db_backup_2019.sql`.

The backup contained two full tables from the hospital's internal HR database:
- **`staff`** — 30 employee records: full name, job title, department, email, phone, national ID number, monthly salary (ZAR), and date joined. Salaries range from R19,000/month (Receptionist) to R160,000/month (Medical Director).
- **`shareholders`** — 10 shareholders with ownership percentages (4%–18%) and share class, including senior medical staff, an HR director, a network engineer, and two corporate/trust entities (Cedar Health Holdings (Pty) Ltd and the Reddy Family Trust).

**Evidence:** [`evidence/m3-critical-exposure/`](evidence/m3-critical-exposure/)
1. Unauthenticated directory listing exposing the backup file
2. Staff payroll and national ID records from the backup
3. Shareholder capitalisation table from the backup

## M4 — Penetration Testing Report

The full report — Executive Summary, Scope & Methodology, Findings & Proof of Exploitation, Risk Rating, and Recommendations — is in [`report/Mediroza-Pentest-Report-Week4.docx`](report/Mediroza-Pentest-Report-Week4.docx).

## Tools Used

| Tool | Purpose |
|---|---|
| `whois` | Domain registration reconnaissance |
| `whatweb` | Web technology / server fingerprinting |
| `dnsrecon` | DNS record enumeration |
| `gobuster` | Directory and file brute-forcing |
| `curl` | Manual HTTP request crafting, header inspection, cookie handling |
| Manual SQL injection | Authentication bypass testing against login forms |
| `pdfcrack` | Dictionary attack against password-protected PDFs |
| `rockyou.txt` | Password wordlist used for PDF cracking |

## Repository Structure

```
.
├── README.md
├── report/
│   └── Mediroza-Pentest-Report-Week4.docx
└── evidence/
    ├── m1-initial-access/
    │   ├── 1_whois.png
    │   ├── 2_gobuster_robots_sitemap.png
    │   ├── 3_old_directory_listing.png
    │   ├── 4_sqli_error.png
    │   └── 5_portal_access_bypass.png
    ├── m2-data-extraction/
    │   ├── 1_pdfcrack_all_passwords.png
    │   ├── 2_report_dlamini.png
    │   ├── 3_report_reddy.png
    │   └── 4_report_thompson.png
    └── m3-critical-exposure/
        ├── 1_old_directory_listing.png
        ├── 2_sql_dump_staff_table.png
        └── 3_sql_dump_shareholders_table.png
```

## What I Learned

This engagement showed how two independent, low-sophistication root causes — unsanitised SQL input and an unprotected legacy directory — were each enough on their own to fully compromise patient confidentiality and expose an organisation's entire payroll and ownership structure. Neither required advanced exploitation techniques: `robots.txt` itself pointed to both issues, and a handful of manual `curl` requests and one classic SQL injection payload were sufficient to prove complete compromise. It reinforced that basic security hygiene — parameterised queries, no directory listing, no legacy files in the web root, and strong document encryption — closes the overwhelming majority of real-world risk.

## Disclaimer

This project was conducted in a controlled environment for educational purposes only, against a target explicitly authorised for security testing as part of the NetworkWalks Cybersecurity Program. These techniques must never be applied to any system without explicit written permission from the owner.

## Author

**Emmanuel Balami**

NetworkWalks Cybersecurity Program (Batch B083)

Week 4 — Penetration Testing Project: Mediroza General Hospital
