# NetworkWalks Cybersecurity Internship — Week 4
## Penetration Testing Project: Mediroza General Hospital

**Batch:** B083 | **Intern:** Kwanele Dube (MrHim) | **Intern ID:** NW-83-CFM  
**Target:** https://medirozahospital.com  
**Engagement Type:** Black-box Penetration Test | **Duration:** 5 Days  
**Authorization:** Written authorization granted by NetworkWalks on behalf of the client

> ⚠️ This project was conducted in a controlled, authorized environment for educational purposes only. Techniques demonstrated here were performed with explicit written permission and must never be used against systems without explicit written authorization.

---

## 🎯 Milestones Overview

| Milestone | Objective | Status |
|---|---|---|
| **M1 — Initial Access** | Gain unauthorized access and retrieve 3 confidential patient PDF lab reports | ✅ Complete |
| **M2 — Encryption Cracking** | Crack the encryption on all 3 retrieved files | ✅ Complete |
| **M3 — Data Exposure** | Find critical data exposure — staff salaries & shareholder details | ✅ Complete |
| **M4 — Pentest Report** | Write a professional penetration testing report | ✅ Complete |

---

## 🛠️ Tools Used

| Tool | Purpose |
|---|---|
| `whois` | Domain registration and registrar information |
| `dnsrecon` | DNS enumeration — SOA, NS, MX, A, TXT, SRV records |
| `curl` | HTTP header fingerprinting and authenticated session replay |
| `gobuster` | Directory and file brute-force enumeration |
| `cewl` | Custom wordlist generation from target site content |
| `sqlmap` | Automated SQL injection testing |
| `hydra` | Credential brute-forcing (false positive documented) |
| `pdfcrack` | PDF password recovery |

---

## M1 — Initial Access

### Phase 1: Passive Reconnaissance

**WHOIS Lookup** — identified domain registration, registrar (NameCheap), name servers (DNS1/DNS2.NAMECHEAPHOSTING.COM), creation date (2026-08-14), and expiry date (2027-08-14).

![WHOIS Lookup](https://github.com/user-attachments/assets/03077bb7-cf6e-40cb-8c5e-28ac7d2de73d)

*Figure 1: WHOIS lookup output for the target domain.*

---

**DNS Enumeration** — `dnsrecon` revealed SOA/NS records pointing to Namecheap hosting, MX records hosted by jellyfish.systems, A record resolving to 199.188.201.16, SPF/DMARC TXT records, and multi-level DNS hierarchy.

![DNS Recon](https://github.com/user-attachments/assets/31454a1d-b043-4151-880d-e136daecbe69)

*Figure 2: DNS enumeration summary using dnsrecon.*

---

### Phase 2: Active Reconnaissance

**HTTP Header Fingerprinting** — `curl -I` on the root domain confirmed the server was running **LiteSpeed**; PHP was not exposed at root level. The staff login page leaked **PHP/8.2.33** via `x-powered-by` header.

![HTTP Headers](https://github.com/user-attachments/assets/db4ad2c4-2dcf-4033-ad6d-4c7c1e2ec3d6)

*Figure 3: HTTP response headers from the live web server.*

![Staff Login Headers](https://github.com/user-attachments/assets/00f49f0a-3db2-4c21-8988-d00246246bad)

*Figure 4: Staff-login response headers revealing PHP version leakage.*

---

**Site Browsing** — Manually navigated the target. Identified pages: Home, About, Doctors, Contact, Staff Login (`/staff/login.php`), and Patient Portal (`/patient/login.php`). View-source on the Doctors page revealed:

```html
<meta name="generator" content="Mediroza CMS 1.4.2">
```

This was investigated as a possible FUEL CMS rebrand (CVE-2018-16763 RCE, CVE-2018-16762 SQLi). It was ruled out after analysis showed `PHPSESSID` (native PHP) rather than `ci_session` (CodeIgniter/FUEL).

![Site Homepage](https://github.com/user-attachments/assets/bdbb746e-3279-4d42-bef9-bf30667fea91)

*Figure 5: Public homepage of the hospital website.*

![View Source: Doctors Page](https://github.com/user-attachments/assets/87fd7c8a-b0fb-4fc6-bdf7-10b1b8f9f512)

*Figure 6: Source code inspection on the Doctors page.*

---

### Phase 3: Attack Surface Mapping

**robots.txt / sitemap.xml** — Both returned HTTP 200 with small content-lengths (132 and 391 bytes respectively). Neither disclosed hidden paths beyond the standard site navigation.

![Robots and Sitemap](https://github.com/user-attachments/assets/3bec2b40-d524-46dd-b285-4accf98b4173)

*Figure 7: robots.txt and sitemap.xml output.*

---

**Directory Enumeration (gobuster) — Pass 1** — `dirb/common.txt` with multiple extensions. Key finding: `/old/` returned Status 301. It also revealed `robots.txt`, `sitemap.xml`, and `/staff/` directories.

![Gobuster Run](https://github.com/user-attachments/assets/d52ea076-f58e-4fa2-b789-c4ead5652c6c)

*Figure 8: First gobuster directory enumeration pass.*

---

**Directory Enumeration — Pass 2 (big.txt with exclude-length)** — The server returns HTTP 200 for all non-existent paths (wildcard soft-404s). `--exclude-length` was used to filter these.

![Wordlist Gobuster](https://github.com/user-attachments/assets/c49cf51e-d359-4f6e-a493-5000301399fc)

*Figure 9: Large-wordlist enumeration with length filtering.*

---

**Enumeration Complete — Key Finding: `/old/` directory** — Gobuster completed confirming `/old/` (Status 301) alongside standard cPanel system aliases. Direct enumeration of `/old/` revealed a publicly accessible directory with sensitive files.

![Finished Enumeration](https://github.com/user-attachments/assets/1112927e-c550-42cc-b15e-cf522ad00857)

*Figure 10: Final file-enumeration results revealing the old directory.*

---

**WAF/Anti-Bot Detection Note** — During enumeration, certain paths (`/uploads/`, `/lab-reports/`) returned misleading responses via curl due to a JavaScript-based bot-detection layer. Browser verification confirmed the paths were blocked by OpenResty WAF.

![OpenResty WAF Discovery](https://github.com/user-attachments/assets/f87c5e59-6363-4d1e-967e-e2099398be75)

*Figure 11: OpenResty WAF/bot-detection behavior discovered during enumeration.*

---

**`/old/` Directory Discovery** — Accessing `/old/` confirmed directory listing was enabled (autoindex). The directory contained a single file: a full, publicly downloadable SQL database backup.

```
/old/mediroza_db_backup_2019.sql
```

![Old Directory Listing](https://github.com/user-attachments/assets/f9e703d9-65fb-4a9e-bc93-f63b35a31b6a)

*Figure 12: Public directory listing showing the exposed SQL backup file.*

---

### Phase 4: Vulnerability Identification

**Staff Login — Tested, No SQLi Found** — Manual single-quote testing, boolean logic probes, and `sqlmap` with direct POST data and `--random-agent` all returned the same result: `all tested parameters appear to be not vulnerable`.

![SQLMap Staff Check](https://github.com/user-attachments/assets/fe8968f9-d2ad-455a-9fa5-e0c2f951f7ee)

*Figure 13: SQLMap check against the staff login interface.*

---

**Patient Login — SQL Injection Confirmed** — Single-quote input in the username field triggered a raw, unhandled MySQL error:

> `Warning: mysqli_query(): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near ''' at line 1`

The patient portal also returns **different error messages** depending on username existence — "Username not found" vs "Incorrect password" — confirming **username enumeration** and the existence of an SQLi vulnerability.

![Patient SQLi Error](https://github.com/user-attachments/assets/6964d407-1551-4cf5-8750-78bbcee280ba)

*Figure 14: SQL injection error response observed on the patient login form.*

![Username Enumeration](https://github.com/user-attachments/assets/25253145-d4f1-4652-9f63-44df4b7baac8)

*Figure 15: Differential login responses exposing username enumeration.*

---

### Phase 5: Exploitation — Authentication Bypass

Using the confirmed SQL injection and enumerated username, the following payload achieved full authentication bypass:

| Field | Value |
|---|---|
| Username | `admin'-- -` |
| Password | `x` (any value) |

The `-- -` comments out the remainder of the SQL `WHERE` clause, bypassing the password check entirely.

![Authentication Bypass](https://github.com/user-attachments/assets/fe164af0-6139-41fc-85c0-522c33cd4190)

*Figure 16: Successful login bypass using the SQL injection payload.*

---

**✅ M1 Complete — Portal access gained. All 3 confidential patient lab reports retrieved:**

- Pathology Report — S. Dlamini (LR-2024-1187, 2024-11-04)
- Pathology Report — P. Reddy (LR-2024-1192, 2024-11-05)
- Pathology Report — E. Thompson (LR-2024-1205, 2024-11-06)

![Portal Reports](https://github.com/user-attachments/assets/d612ed2c-b217-4814-9d65-856af27cec2a)

*Figure 17: Patient portal showing the three retrieved lab reports.*

---

## M2 — Encryption Cracking

All 3 PDFs were downloaded via an authenticated `curl` session using a cookie jar from the SQLi-bypassed login. All 3 confirmed as genuine PDF documents (PDF version 1.4, 1 page each).

**Encryption details (visible in pdfcrack output):** V:2, R:3, Length:128 — RC4 128-bit standard security handler.

![PDF Download / File Info](https://github.com/user-attachments/assets/5fa69c6c-beaa-4922-a8c9-7c496657cead)

*Figure 18: Downloaded PDF reports and file metadata.*

---

**Cracking approach and tooling challenges:**
- `pdf2john.pl` + `john --format=PDF` — hash extracted successfully but john refused to load it (format detection issue, unresolved in this lab environment)
- `hashcat -m 10500` — failed with "Not enough allocatable device memory" even after installing `pocl-opencl-icd`; VM RAM was insufficient for hashcat's buffer allocation
- **`pdfcrack`** — CPU-native, lightweight, no GPU required. Successfully cracked all 3 files against `rockyou.txt`

| File | Patient | Lab Ref | Password Recovered |
|---|---|---|---|
| `patient_report_1.pdf` | S. Dlamini | LR-2024-1187 | `123456` |
| `patient_report_2.pdf` | P. Reddy | LR-2024-1192 | `password` |
| `patient_report_3.pdf` | E. Thompson | LR-2024-1205 | `!@#$%^&` |

![PDFCrack Results](https://github.com/user-attachments/assets/27629115-8d09-4a45-b835-1a016bcd0fed)

*Figure 19: pdfcrack output recovering all three PDF passwords.*

---

**✅ M2 Complete — All 3 PDF passwords recovered. Files opened and patient report contents confirmed:**

![Opened PDFs](https://github.com/user-attachments/assets/ebc2e48b-7e1f-40c8-9cbc-098ff4fefdbb)

*Figure 20: Opened PDF report files after password recovery.*

---

## M3 — Critical Data Exposure

The database backup discovered in M1 (`/old/mediroza_db_backup_2019.sql`) was downloaded and analyzed. The first download attempt was blocked by the WAF (returned an HTML challenge page); a second attempt using a browser User-Agent string succeeded.

![SQL Download](https://github.com/user-attachments/assets/541a4e16-ce36-44da-9fbe-f91d5a2da504)

*Figure 21: Successful download of the exposed database backup.*

---

**Database structure confirmed — two tables identified:**

```sql
CREATE TABLE `staff`;
CREATE TABLE `shareholders`;
```

![Database Tables](https://github.com/user-attachments/assets/b072f651-c295-4ce1-af08-58a6275145b1)

*Figure 22: Structural schema of the recovered database tables.*

---

### Staff Salaries + Critical PII

The `staff` table contained 30 employee records with columns: `id`, `full_name`, `job_title`, `department`, `email`, `phone`, **`national_id`**, **`monthly_salary_zar`**, and `date_joined`.

> ⚠️ **Additional critical finding beyond the milestone scope:** the `national_id` column exposes South African national identity numbers for all 30 staff members in plaintext — a serious POPIA violation requiring immediate remediation.

### Shareholder Details

The `shareholders` table contained 10 records: `shareholder_name`, `share_percent`, `shares_held`, and `share_class` (Ordinary/Preferential).

Top shareholders included Dr. Rajesh Naidoo (18%), Cedar Health Holdings (Pty) Ltd (15%), and Dr. Johan van der Merwe (12%).

![Staff and Shareholders Data](https://github.com/user-attachments/assets/8270e71d-7901-4daa-9356-6c8dfe4be96a)

*Figure 23: Staff salary and shareholder data extracted from the public backup.*

---

**✅ M3 Complete — Staff salaries, national ID numbers, and full shareholder details obtained from unauthenticated public backup.**

---

## M4 — Penetration Testing Report

A full professional penetration testing report was written covering all 8 findings with severity ratings, proof of exploitation, and remediation recommendations.

📄 **[Download the full report (PDF)](Mediroza_Pentest_Report_WK4.pdf)**

### Findings Summary

| # | Finding | Severity |
|---|---|---|
| F1 | SQL Injection — Authentication Bypass (Patient Portal) | 🔴 Critical |
| F2 | Publicly Exposed Database Backup | 🔴 Critical |
| F3 | Weak PDF Encryption / Crackable Passwords | 🟠 High |
| F4 | Exposure of National Identification Numbers | 🟠 High |
| F5 | Directory Listing Enabled (`/old/`) | 🟡 Medium |
| F6 | Missing CSRF Protection on Login Forms | 🟡 Medium |
| F7 | Username Enumeration via Differential Error Messages | 🟢 Low–Medium |
| F8 | Verbose SQL Error Messages Disclosed to Users | 🟢 Low |

---

## 🔑 Key Lessons

- **Tool output is never ground truth** — Hydra returned 16 false-positive "valid passwords" because it attacked HTTP port 80 instead of HTTPS port 443. Always verify surprising results manually before acting on them.
- **The WAF is not the target** — Sustained automated scanning triggered bot-detection mid-engagement, blocking curl/sqlmap/gobuster for a period. Falling back to browser-based manual testing bypassed the defense.
- **Recon pays off** — The `/old/` directory was found through systematic enumeration. The database backup inside it solved M3 and provided patient names later used to verify M1.
- **Different forms, different code** — Staff login and patient login were built independently. Staff login had no SQLi. Patient login did. Never assume two similar-looking forms share the same security posture.
- **Adapt when tools fail** — john and hashcat both hit environment issues. pdfcrack solved the same problem in minutes. Knowing alternatives matters as much as knowing primary tools.

---

## 📱 LinkedIn

Share your thoughts on this penetration testing project and cybersecurity insights:

**LinkedIn Post:** https://www.linkedin.com/feed/update/urn:li:activity:7512054527412527104/

Discuss this work on LinkedIn and connect with the cybersecurity community. Your feedback and engagement are welcome!

---

## ⚠️ Disclaimer

This assessment was performed under explicit written authorization as part of a structured training engagement (NetworkWalks Cybersecurity Internship, Batch B083). No techniques described here were used against systems without explicit authorization. This project is shared for educational purposes only to demonstrate cybersecurity assessment methodologies and defensive security awareness.
