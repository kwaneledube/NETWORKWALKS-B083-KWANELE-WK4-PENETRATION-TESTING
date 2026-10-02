<img width="1024" height="702" alt="02_dnsrecon" src="https://github.com/user-attachments/assets/e97258db-f94e-43ee-9730-b475a4a3408c" />
# NetworkWalks Cybersecurity Internship — Week 4
## Penetration Testing Project: Mediroza General Hospital

**Batch:** B083 | **Intern:** Kwanele Dube (MrHim) | **Intern ID:** NW-83-CFM  
**Target:** https://medirozahospital.com  
**Engagement Type:** Black-box Penetration Test | **Duration:** 5 Days  
**Authorization:** Written authorization granted by NetworkWalks on behalf of the client

> ⚠️ This project was conducted in a controlled, authorized environment for educational purposes only. Techniques demonstrated here were performed with explicit written permission and must never be applied to any system without the same authorization.

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

![whois](M1-Initial-Access/<img width="1024" height="702" alt="01_whois" src="https://github.com/user-attachments/assets/03077bb7-cf6e-40cb-8c5e-28ac7d2de73d" />.png)

---

**DNS Enumeration** — `dnsrecon` revealed SOA/NS records pointing to Namecheap hosting, MX records hosted by jellyfish.systems, A record resolving to 199.188.201.16, SPF/DMARC TXT records, and multiple SRV records confirming cPanel mail infrastructure.

![dnsrecon](M1-Initial-Access/02_<img width="1024" height="702" alt="02_dnsrecon" src="https://github.com/user-attachments/assets/31454a1d-b043-4151-880d-e136daecbe69" />
dnsrecon.png)
<img width="1024" height="702" alt="03_http_headers" src="https://github.com/user-attachments/assets/2b32d7a8-bba3-4fba-be87-8741160d99de" />

---
<img width="1024" height="702" alt="08_gobuster_run" src="https://github.com/user-attachments/assets/d52ea076-f58e-4fa2-b789-c4ead5652c6c" />
<img width="1024" height="768" alt="07_robots_sitemap" src="https://github.com/user-attachments/assets/3bec2b40-d524-46dd-b285-4accf98b4173" />
<img width="1024" height="640" alt="06_view_source_doctors" src="https://github.com/user-attachments/assets/87fd7c8a-b0fb-4fc6-bdf7-10b1b8f9f512" />
<img width="1024" height="640" alt="05_site_homepage" src="https://github.com/user-attachments/assets/bdbb746e-3279-4d42-bef9-bf30667fea91" />
<img width="512" height="320" alt="04_staff_login_headers" src="https://github.com/user-attachments/assets/00f49f0a-3db2-4c21-8988-d00246246bad" />
<img width="1024" height="702" alt="03_http_headers" src="https://github.com/user-attachments/assets/db4ad2c4-2dcf-4033-ad6d-4c7c1e2ec3d6" />

### Phase 2: Active Reconnaissance

**HTTP Header Fingerprinting** — `curl -I` on the root domain confirmed: server running **LiteSpeed**, PHP not exposed at root level. Staff login page leaked **PHP/8.2.33** via `x-powered-by`. No framework-specific cookies or headers — pointing to a custom PHP application rather than a known CMS.

![http_headers](M1-Initial-Access/03_http_headers.png)

![staff_patient_headers](M1-Initial-Access/04_staff_login_headers.png)

---

**Site Browsing** — Manually navigated the target. Identified pages: Home, About, Doctors, Contact, Staff Login (`/staff/login.php`), and Patient Portal (`/patient/login.php`). View-source on `doctors.html` revealed a CMS fingerprint in the meta tag:

```html
<meta name="generator" content="Mediroza CMS 1.4.2">
```

This was investigated as a possible FUEL CMS rebrand (CVE-2018-16763 RCE, CVE-2018-16762 SQLi). Ruled out: cookie analysis showed `PHPSESSID` (native PHP) rather than `ci_session` (CodeIgniter/FUEL default), and all `/fuel/` paths returned genuine 404s.

![site_homepage](M1-Initial-Access/05_site_homepage.png)

![view_source_doctors](M1-Initial-Access/06_view_source_doctors.png)

---

### Phase 3: Attack Surface Mapping

**robots.txt / sitemap.xml** — Both returned HTTP 200 with small content-lengths (132 and 391 bytes respectively). Neither disclosed hidden paths beyond the standard site navigation.

![robots_sitemap](M1-Initial-Access/07_robots_sitemap.png)

---

**Directory Enumeration (gobuster) — Pass 1** — `dirb/common.txt` with multiple extensions. Key finding: `/old/` returned Status 301. Also revealed `robots.txt`, `sitemap.xml`, `/staff/` — no sensitive files directly accessible yet.

![gobuster_run](M1-Initial-Access/08_gobuster_run.png)

---

**Directory Enumeration — Pass 2 (big.txt with exclude-length)** — The server returns HTTP 200 for all non-existent paths (wildcard soft-404s). `--exclude-length` was used to filter these. The full 61,407-entry run confirmed `/old/`, `/patient/`, `/staff/` as real paths — no additional sensitive directories found.

![wordlist_gobuster](M1-Initial-Access/wordlist_gobuster.png)

---

**Enumerat<img width="1024" height="768" alt="wordlist gobuster" src="https://github.com/user-attachments/assets/c49cf51e-d359-4f6e-a493-5000301399fc" />
<img width="1024" height="702" alt="Finished file enumeration" src="https://github.com/user-attachments/assets/1112927e-c550-42cc-b15e-cf522ad00857" />
<img width="1024" height="768" alt="OpenResty WAF discovery" src="https://github.com/user-attachments/assets/f87c5e59-6363-4d1e-967e-e2099398be75" />
ion Complete — Key Finding: `/old/` directory** — Gobuster completed confirming `/old/` (Status 301) alongside standard cPanel system aliases. Direct enumeration of `/old/` revealed a publicly accessible database backup file.

![finished_enumeration](M1-Initial-Access/Finished_file_enumeration.png)

---
<img width="1024" height="768" alt="12   13_staff_and_shareholders_data" src="https://github.com/user-attachments/assets/8270e71d-7901-4daa-9356-6c8dfe4be96a" />
<img width="410" height="256" alt="11_db_tables" src="https://github.com/user-attachments/assets/b072f651-c295-4ce1-af08-58a6275145b1" />
<img width="683" height="427" alt="10_sql_downloaded" src="https://github.com/user-attachments/assets/541a4e16-ce36-44da-9fbe-f91d5a2da504" />
<img width="1024" height="640" alt="22_pdfs_opened" src="https://github.com/user-attachments/assets/ebc2e48b-7e1f-40c8-9cbc-098ff4fefdbb" />
<img width="1024" height="768" alt="21_pdfcrack_results" src="https://github.com/user-attachments/assets/27629115-8d09-4a45-b835-1a016bcd0fed" />
<img width="1024" height="640" alt="19   20_pdf_download_reports_file_info" src="https://github.com/user-attachments/assets/5fa69c6c-beaa-4922-a8c9-7c496657cead" />
<img width="1024" height="640" alt="18_portal_3_reports" src="https://github.com/user-attachments/assets/d612ed2c-b217-4814-9d65-856af27cec2a" />
<img width="1024" height="640" alt="17_auth_bypass" src="https://github.com/user-attachments/assets/fe164af0-6139-41fc-85c0-522c33cd4190" />
<img width="1024" height="640" alt="16_username_enumeration" src="https://github.com/user-attachments/assets/25253145-d4f1-4652-9f63-44df4b7baac8" />
<img width="1024" height="640" alt="15_patient_sqli_error" src="https://github.com/user-attachments/assets/6964d407-1551-4cf5-8750-78bbcee280ba" />
<img width="1024" height="768" alt="14_sqlmap_staff" src="https://github.com/user-attachments/assets/fe8968f9-d2ad-455a-9fa5-e0c2f951f7ee" />
<img width="683" height="427" alt="09_old_directory_listing" src="https://github.com/user-attachments/assets/f9e703d9-65fb-4a9e-bc93-f63b35a31b6a" />

**WAF/Anti-Bot Detection Note** — During enumeration, certain paths (`/uploads/`, `/lab-reports/`) returned misleading responses via curl due to a JavaScript-based bot-detection layer. Browser verification confirmed these were genuine 404s. This WAF behavior affected automated tooling partway through the engagement — documented as a defensive finding (F5 in the report).

![openresty_waf](M1-Initial-Access/OpenResty_WAF_discovery.png)

---

**`/old/` Directory Discovery** — Accessing `/old/` confirmed directory listing was enabled (autoindex). The directory contained a single file: a full, publicly downloadable SQL database backup.

```
/old/mediroza_db_backup_2019.sql
```

![old_directory_listing](M1-Initial-Access/09_old_directory_listing.png)

---

### Phase 4: Vulnerability Identification

**Staff Login — Tested, No SQLi Found** — Manual single-quote testing, boolean logic probes, and `sqlmap` with direct POST data and `--random-agent` all returned the same result: `all tested parameters do not appear to be injectable`. Staff login confirmed **not vulnerable**.

![sqlmap_staff_clean](M1-Initial-Access/14_sqlmap_staff.png)

---

**Patient Login — SQL Injection Confirmed** — Single-quote input in the username field triggered a raw, unhandled MySQL error:

> `Warning: mysqli_query(): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near ''' at line 1`

The patient portal also returns **different error messages** depending on username existence — "Username not found" vs "Incorrect password" — confirming **username enumeration** and the existence of the `admin` account.

![patient_sqli_error](M1-Initial-Access/15_patient_sqli_error.png)

![username_enumeration](M1-Initial-Access/16_username_enumeration.png)

---

### Phase 5: Exploitation — Authentication Bypass

Using the confirmed SQLi and enumerated username, the following payload achieved full authentication bypass:

| Field | Value |
|---|---|
| Username | `admin'-- -` |
| Password | `x` (any value) |

The `-- -` comments out the remainder of the SQL `WHERE` clause, bypassing the password check entirely.

![auth_bypass](M1-Initial-Access/17_auth_bypass.png)

---

**✅ M1 Complete — Portal access gained. All 3 confidential patient lab reports retrieved:**

- Pathology Report — S. Dlamini (LR-2024-1187, 2024-11-04)
- Pathology Report — P. Reddy (LR-2024-1192, 2024-11-05)
- Pathology Report — E. Thompson (LR-2024-1205, 2024-11-06)

![portal_3_reports](M1-Initial-Access/18_portal_3_reports.png)

---

## M2 — Encryption Cracking

All 3 PDFs were downloaded via an authenticated `curl` session using a cookie jar from the SQLi-bypassed login. All 3 confirmed as genuine PDF documents (PDF version 1.4, 1 page each).

**Encryption details (visible in pdfcrack output):** V:2, R:3, Length:128 — RC4 128-bit standard security handler.

![pdf_download](M2-Encryption-Cracking/19___20_pdf_download_reports_file_info.png)

---

**Cracking approach and tooling challenges:**
- `pdf2john.pl` + `john --format=PDF` — hash extracted successfully but john refused to load it (format detection issue, unresolved in this lab environment)
- `hashcat -m 10500` — failed with "Not enough allocatable device memory" even after installing `pocl-opencl-icd` CPU runtime; VM RAM (1GB) insufficient for hashcat's buffer allocation
- **`pdfcrack`** — CPU-native, lightweight, no GPU required. Successfully cracked all 3 files against `rockyou.txt`

| File | Patient | Lab Ref | Password Recovered |
|---|---|---|---|
| patient_report_1.pdf | S. Dlamini | LR-2024-1187 | `123456` |
| patient_report_2.pdf | P. Reddy | LR-2024-1192 | `password` |
| patient_report_3.pdf | E. Thompson | LR-2024-1205 | `!@#$%^&` |

![pdfcrack_results](M2-Encryption-Cracking/21_pdfcrack_results.png)

---

**✅ M2 Complete — All 3 PDF passwords recovered. Files opened and patient report contents confirmed:**

![pdfs_opened](M2-Encryption-Cracking/22_pdfs_opened.png)

---

## M3 — Critical Data Exposure

The database backup discovered in M1 (`/old/mediroza_db_backup_2019.sql`) was downloaded and analyzed. The first download attempt was blocked by the WAF (returned an HTML challenge page); a second authenticated attempt succeeded, confirming the file as ASCII text.

![sql_download](M3-Data-Exposure/10_sql_downloaded.png)

---

**Database structure confirmed — two tables identified:**

```
CREATE TABLE `staff`
CREATE TABLE `shareholders`
```

![db_tables](M3-Data-Exposure/11_db_tables.png)

---

### Staff Salaries + Critical PII

The `staff` table contained 30 employee records with columns: `id`, `full_name`, `job_title`, `department`, `email`, `phone`, **`national_id`**, **`monthly_salary_zar`**, `date_joined`.

> ⚠️ **Additional critical finding beyond the milestone scope:** the `national_id` column exposes South African national identity numbers for all 30 staff members in plaintext — a serious POPIA compliance violation.

### Shareholder Details

The `shareholders` table contained 10 records: `shareholder_name`, `share_percent`, `shares_held`, `share_class` (Ordinary/Preferential).

Top shareholders included Dr. Rajesh Naidoo (18%), Cedar Health Holdings (Pty) Ltd (15%), and Dr. Johan van der Merwe (12%).

![staff_and_shareholders](M3-Data-Exposure/12___13_staff_and_shareholders_data.png)

---

**✅ M3 Complete — Staff salaries, national ID numbers, and full shareholder details obtained from unauthenticated public backup.**

---

## M4 — Penetration Testing Report

A full professional penetration testing report was written covering all 8 findings with severity ratings, proof of exploitation, and remediation recommendations.

📄 **[Download the full report](M4-Report/Mediroza_Pentest_Report_WK4.docx)**

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

- **Tool output is never ground truth** — Hydra returned 16 false-positive "valid passwords" because it attacked HTTP port 80 instead of HTTPS port 443. Always verify surprising results manually before reporting.
- **The WAF is not the target** — Sustained automated scanning triggered bot-detection mid-engagement, blocking curl/sqlmap/gobuster for a period. Falling back to browser-based manual testing bypassed it cleanly.
- **Recon pays off** — The `/old/` directory was found through systematic enumeration. The database backup inside it solved M3 and provided patient names later used to verify M1.
- **Different forms, different code** — Staff login and patient login were built independently. Staff login had no SQLi. Patient login did. Never assume two similar-looking forms share the same security posture.
- **Adapt when tools fail** — john and hashcat both hit environment issues. pdfcrack solved the same problem in minutes. Knowing alternatives matters as much as knowing primary tools.

---

## ⚠️ Disclaimer

This assessment was performed under explicit written authorization as part of a structured training engagement (NetworkWalks Cybersecurity Internship, Batch B083). No techniques described here were used against any system without permission. The target is a fictional training environment — any resemblance to real individuals or organizations is coincidental.
