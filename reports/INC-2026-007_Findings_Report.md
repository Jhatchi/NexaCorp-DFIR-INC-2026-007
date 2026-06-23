# INC-2026-007 Findings Report: IDOR and Broken Access Control

**Engagement:** NexaCorp DFIR, IDOR and Broken Access Control on NexaPortal
**Reference:** BCC-2026 / INC-2026-007
**Target system:** NexaPortal (Apache 2.4.57 on Debian, host 192.168.10.26)
**Reported by:** Marc Wauters, IT Infrastructure Manager, NexaCorp Industries
**Analyst:** Johan-Emmanuel Hatchi, SOC Analyst L1, BeCode Corp
**Classification:** Confidential, do not distribute outside BeCode Corp

> Timezone note : log timestamps are UTC (+0000). Local time is CEST (UTC+02:00). Both are given where it aids the timeline.

---

## 1. Executive summary

On 19 June 2026, an attacker logged in to NexaCorp's internal portal (NexaPortal) using a valid account that does not belong to any NexaCorp employee, walked through the entire employee directory one record at a time, reached an administrative page that should have been restricted, and downloaded a full export of employee personal data. No software exploit, no malware and no stolen session were needed : the application simply trusted an authenticated user to only request what it showed them.

The account used, `m.renard`, is not in NexaCorp's IT-provisioned account register. It is the same foothold that has been present since INC-2026-005 and was never removed. The session enumerated 47 distinct employee records through an Insecure Direct Object Reference (IDOR) flaw, then used a broken access control on the admin area to trigger a bulk export of 47 employees' personal data (name, corporate email, department, job title, phone number and salary band).

This is a confirmed exfiltration of personal data and is rated **Critical**. Because identifiable personal data of 47 data subjects left the controller's systems, a GDPR breach notification to the supervisory authority is required (see section 4). The single most important point for NexaCorp management is that the root cause is not the IDOR flaw in isolation : it is the failure to remove the attacker's account after INC-2026-005. The XSS mitigations deployed on 18 June (HttpOnly and Content Security Policy) were correct and effective, but irrelevant here, because the attacker no longer needed XSS : a valid login was enough.

---

## 2. Incident timeline

| Time (UTC) | Time (CEST) | Event | Source |
|---|---|---|---|
| 19/Jun 10:11-10:12 | 12:11-12:12 | Credential brute force against `login.php` from 172.16.50.40 (admin, root, administrator, ...), all FAILED | nexaportal_auth.log |
| 19/Jun 16:23:01 | 18:23:01 | `m.renard` authenticates SUCCESS from 172.16.50.10 (account absent from the IT register) | nexaportal_auth.log, web_access.log |
| 19/Jun 16:23:03 | 18:23:03 | Dashboard loaded, session established | web_access.log |
| 19/Jun 16:23:08-16:25:28 | 18:23:08-18:25:28 | Automated IDOR enumeration of `/portal/employees.php?id=1` through `id=47`, every request HTTP 200 | web_access.log |
| 19/Jun 16:37:50 | 18:37:50 | `GET /portal/admin/` returns HTTP 200 to the non-admin session (broken access control) | web_access.log |
| 19/Jun 16:38:01 | 18:38:01 | `GET /portal/admin/export_users.php` returns HTTP 200, 3858 bytes : bulk export of employee data | web_access.log, pcap, CSV |

The whole intrusion is tight : 15 minutes from login (16:23:01) to data export (16:38:01). A deliberate pause of about 12 minutes separates the end of the directory enumeration (16:25:28) from the admin access (16:37:50).

---

## 3. Technical analysis

### 3.1 Finding 1 - Unauthorized valid account (High)

The portal authentication log records a successful login for `m.renard` at 16:23:01 from 172.16.50.10. Cross-checked against `reference/employee_roster.txt` (NexaCorp's IT-provisioned account register), `m.renard` does not exist. A legitimate employee with a similar surname does exist (`p.renard`, Pierre Renard), which is consistent with an account named to blend in.

`m.renard` is the same unauthorized account first observed during INC-2026-006, and it traces back to the foothold established in INC-2026-005 (web shell and credential reuse on the former portal). The XSS patch deployed on 18 June did not, and could not, address an attacker who already holds a valid login.

The credential brute force seen earlier (10:11-10:12 from 172.16.50.40, all failures) is unrelated scanner noise : it never succeeded. The actual compromise is a single valid login from a different source, 172.16.50.10.

### 3.2 Finding 2 - Insecure Direct Object Reference on the employee directory (High)

The endpoint `/portal/employees.php` selects the employee record to display from a client-supplied integer parameter, `id`, with no check that the requesting user is authorized to view that specific record.

Between 16:23:08 and 16:25:28 the session requested `id=1` through `id=47` in near-sequential order, roughly one record every two to three seconds, every response returning HTTP 200. This pattern - exhaustive, ordered, machine-paced, from a single source - is automated enumeration, not human browsing. **47 distinct employee records** were accessed (the values `id=9` and `id=23` were re-requested, but add no new records).

This is the distinction NexaCorp will be challenged on : a high number of profile views is not, by itself, an attack. What makes this an attack is the combination of an unauthorized account, a single source address, a strictly sequential sweep of the entire id space, and a machine cadence no employee doing their job would produce.

- Weakness : CWE-639 (Authorization Bypass Through User-Controlled Key)
- Category : OWASP A01:2021 Broken Access Control

### 3.3 Finding 3 - Broken access control on the admin area and bulk exfiltration (Critical)

At 16:37:50 the session requested `/portal/admin/` and the server returned HTTP **200**. For an unauthorized or unauthenticated request, the same path returns HTTP **302** (a redirect to the login page) - this is visible elsewhere in the access log for scanner traffic against `/portal/admin/`. The 200 served to a non-admin session is direct proof that the server did not enforce an authorization check on the administrative area.

At 16:38:01 the session requested `/portal/admin/export_users.php` and received HTTP 200 with a 3858-byte response. The size matches `exports/employee_directory_export.csv` exactly, and the network capture confirms the response was delivered as `text/csv`. This is the bulk export and the exfiltration event.

- Weakness : CWE-862 (Missing Authorization), CWE-284 (Improper Access Control)
- Category : OWASP A01:2021 Broken Access Control

### 3.4 Finding 4 - Sensitive token disclosed in admin page source (Low)

The HTML returned by `/portal/admin/` contains a comment exposing an internal build identifier and a debug token (`internal build 2026.06.12 | debug_token=...`). Sensitive build and debugging information should never be emitted to clients. This is information disclosure that aids an attacker.

- Weakness : CWE-615 (Inclusion of Sensitive Information in Source Code Comments), CWE-200 (Exposure of Sensitive Information)

---

## 4. Impact and data protection

The exported CSV contains 47 employee records with the following fields per record : full name, corporate email, department, job title, phone number and salary band. This is personal data under the GDPR.

Exposure severity is not uniform. The export includes senior executives in the highest salary band (D1 : the Chief Financial Officer and the Sales Director), and - notably - the organization's own Data Protection Officer. It also includes the account of Jerome Martin (`j.martin`), the very employee whose credentials were stolen by SQL injection in INC-2026-004, closing the loop on the campaign.

**GDPR position.** This is a confirmed exfiltration of identifiable personal data affecting 47 data subjects. Under Article 33 of the GDPR, the controller must notify the competent supervisory authority (for a Belgian establishment, the Data Protection Authority / Autorite de protection des donnees) without undue delay and, where feasible, within **72 hours** of becoming aware of the breach. Given the categories of data and the confirmed exfiltration, notification should be treated as required, and an assessment of notification to the affected individuals (Article 34) should be made in parallel.

---

## 5. Detection gap (Wazuh)

The SIEM produced 211 alerts during the window, and not one of them concerns this attack. Every alert was raised against background scanner traffic from the 172.16.50.20-26 range, under three rules :

| Rule ID | Level | Description | Alerts |
|---|---|---|---|
| 31101 | 5 | Web server 400 error code | 111 |
| 31108 | 6 | Blacklisted user agent (known scanner) | 93 |
| 31151 | 10 | Multiple web server 400 error codes from same source ip | 7 |

The real attacker, 172.16.50.10, generated **zero** alerts. Every malicious request returned HTTP 200, carried a normal Chrome user agent, and rode an authenticated session. A signature based ruleset that keys on 4xx errors and known-scanner user agents is structurally blind to authorization abuse : nothing about a valid user reading 47 records and exporting them looks like a known bad signature. The application-level brute force from 172.16.50.40 was also unalerted, because Wazuh was consuming the Apache access log rather than the portal's own authentication log.

Detection for this class of attack must be behavioral, for example : a single session retrieving many distinct `employees.php?id=` values in a short window; sequential enumeration of the `id` parameter from one source; or any non-admin account reaching `/portal/admin/export_users.php`. These are recommendations only : this engagement is an assessment and produced no deployed detection rules.

---

## 6. Indicators of Compromise

- Attacker source IP : `172.16.50.10` (same source as INC-2026-004 and INC-2026-005)
- Unauthorized account : `m.renard` (absent from the IT account register; foothold since INC-2026-005)
- Scanner / brute-force noise : `172.16.50.40` (failed login attempts), `172.16.50.20-26` (Nikto / WPScan / generic scanners)
- Abused endpoints : `/portal/employees.php?id=<N>` (IDOR), `/portal/admin/` (broken access control), `/portal/admin/export_users.php` (bulk export)
- Exfiltration artifact : `employee_directory_export.csv` (47 records, 3858 bytes)
- Behavioral indicator : sequential `id=1..47` enumeration, ~1 record / 2-3 s, single source, HTTP 200 throughout
- Host indicator : sensitive debug token disclosed in `/portal/admin/` HTML comment

## 7. MITRE ATT&CK mapping

| Technique | ID | Evidence |
|---|---|---|
| Valid Accounts | T1078 | Login with the unauthorized but valid `m.renard` account |
| Automated Collection | T1119 | Scripted sequential enumeration of the employee directory |
| Data from Information Repositories | T1213 | Reading employee records from the portal directory |
| Exfiltration Over Web Service | T1567 | Bulk export downloaded over the application's admin endpoint |

Framework : MITRE ATT&CK Enterprise (v15).

## 8. Remediation

1. **Remove the foothold first.** Disable and delete `m.renard` immediately, and audit the NexaPortal user table for any other account absent from the IT register. This is the root cause; every other fix is secondary until it is done.
2. **Enforce object-level authorization.** On `/portal/employees.php`, verify that the authenticated user is allowed to view the requested `id` before returning a record. Do not rely on the UI to limit which ids are reachable.
3. **Enforce function-level authorization.** Require and verify an administrative role server-side on every `/portal/admin/` page and action, including `export_users.php`. A redirect for unauthenticated users is not authorization.
4. **Rate-limit and alert on enumeration.** Throttle repeated `employees.php?id=` access from a single session and raise an alert on sequential or high-volume directory access.
5. **Add behavioral detection.** Feed the portal authentication log to the SIEM, and add rules for non-admin access to admin/export endpoints and for directory enumeration patterns.
6. **Stop information disclosure.** Remove debug tokens and internal build markers from client-facing responses.
7. **Treat as a reportable breach.** Proceed with the GDPR Article 33 notification and assess Article 34 communication to affected individuals.

---

## 9. Conclusion

INC-2026-007 is the consequence of an un-removed foothold, not a novel exploit. An attacker holding a valid account abused two authorization flaws (IDOR on the directory, broken access control on the admin area) to exfiltrate the personal data of 47 employees in fifteen minutes, entirely below the SIEM's signature-based visibility. Removing `m.renard`, enforcing authorization at both the object and function level, and adding behavioral detection would close this incident and the class of attack behind it.
