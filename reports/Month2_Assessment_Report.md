# Month 2 Security Assessment Report

**Prepared for:** NexaCorp Industries - Board review, 30 June 2026
**Prepared by:** Johan-Emmanuel Hatchi, SOC Analyst L1, BeCode Corp
**Reference:** BCC-2026 / Month 2 Assessment (INC-2026-004 to INC-2026-007)
**Date:** 23 June 2026
**Classification:** Confidential, do not distribute outside BeCode Corp

> Audience note : this report is written for management. Technical evidence is kept in the appendix; the body translates the findings into business impact.

---

## 1. Executive summary

Over four weeks in June 2026, NexaCorp's web application layer was the target of **four connected security incidents (INC-2026-004 to INC-2026-007)**. They are not four separate events : they are one adversary, operating from a single internet source, progressing step by step from an initial break-in to a confirmed theft of employee personal data.

**Key numbers for the month:**

- **4 incidents**, one continuous campaign, same external source throughout.
- **1 employee credential** stolen and cracked (a developer account), then reused to gain a foothold.
- **1 unauthorized "backdoor" account** created inside the portal and, critically, **never removed**.
- **1 employee session** hijacked.
- **47 employee records** exfiltrated (name, corporate email, department, job title, phone number, salary band), including senior executives and the company's own Data Protection Officer.

**Is a GDPR notification required? Yes.** Incident INC-2026-007 is a confirmed exfiltration of identifiable personal data affecting 47 data subjects. Under GDPR Article 33, NexaCorp must notify the supervisory authority (for a Belgian establishment, the Data Protection Authority / Autorite de protection des donnees) without undue delay and, where feasible, within **72 hours** of confirming the breach. Communication to the affected individuals (Article 34) should be assessed in parallel.

**The single most important point.** The root cause of the final and most damaging incident is **not** a new technical flaw. It is an **operational failure : the attacker's backdoor account was left in place** after the earlier incidents were "closed". NexaCorp correctly applied every technical fix recommended along the way - the SQL injection was patched, the compromised IP was blocked, the developer's password was rotated, and the cross-site scripting flaw was mitigated. Those fixes worked. They simply did not matter, because the attacker still held a valid login that nobody removed. The final breach required no exploit at all.

**Recommended immediate actions:**

1. Remove the unauthorized account and audit the portal for any other account that does not belong.
2. Proceed with the GDPR breach notification.
3. Enforce proper authorization checks on the portal (who is allowed to see and export what).
4. Add behaviour-based monitoring : the entire campaign was largely invisible to the current alerting.

---

## 2. Findings of the month

### Finding 04 - SQL Injection (INC-2026-004)
- **Risk rating: Critical**
- **Technique:** SQL injection against the employee portal login / lookup form.
- **Component:** the portal database (employee portal, bru-web-01).
- **Data impacted:** the full user table was extracted, including a developer's account whose weak password was cracked offline. That credential was then reused to log in to the server over SSH, giving the attacker a first foothold.
- **Business impact:** confirmed theft of stored credentials and an authenticated entry point into NexaCorp's environment.

### Finding 05 - Command Injection and Web Shell (INC-2026-005)
- **Risk rating: Critical**
- **Technique:** operating-system command injection through the portal's diagnostic tools, leading to remote code execution. A file-viewer flaw (LFI) was also probed but disclosed nothing.
- **Exploitation path:** the attacker reused the foothold from INC-2026-004, ran system commands as the web server account, read the server's account file, and planted a hidden "web shell" for repeated access.
- **Persistence established:** a web shell on the server, and an interactive SSH session. This is the stage at which an **unauthorized account was introduced into the portal** that becomes the pivot for the rest of the campaign.
- **Business impact:** full remote control of the portal server and durable, attacker-controlled access.

### Finding 06 - Cross-Site Scripting and Session Hijacking (INC-2026-006)
- **Risk rating: High**
- **Technique:** stored cross-site scripting (XSS) in the new NexaPortal feedback feature.
- **Victim account:** an employee (Office Management) whose active session was stolen when they viewed the malicious feedback entry.
- **Session data exposed:** the victim's session identifier was sent to an attacker-controlled server, letting the attacker act as that employee without a password.
- **Business impact:** account impersonation. NexaCorp responded with the correct fixes (HttpOnly cookies, Content Security Policy), which closed this specific technique.

### Finding 07 - IDOR and Broken Access Control (INC-2026-007)
- **Risk rating: Critical**
- **Technique:** Insecure Direct Object Reference (IDOR) plus broken access control on NexaPortal.
- **Scope of exposure:** the unauthorized account from INC-2026-005, still active, logged in and read the entire employee directory one record at a time (47 records), then reached an administrator page that was not properly restricted.
- **Admin access confirmed:** the administrative export page returned data to a non-administrator session, and a full employee export was downloaded.
- **Business impact:** confirmed exfiltration of 47 employees' personal data. This is the incident that triggers the GDPR obligation.

---

## 3. Attack chain narrative : one adversary, four steps

The four incidents are a single, connected progression carried out by the same actor from the same external source:

1. **INC-2026-004 (SQL injection).** The attacker breaks into the portal database and steals employee credentials. One developer password is weak, is cracked, and is reused to log in to the server.
2. **INC-2026-005 (command injection and web shell).** Using that access, the attacker runs commands on the server, plants a web shell, and **inserts an unauthorized "backdoor" account** into the portal. The foothold is now durable and identity-based.
3. **INC-2026-006 (XSS and session hijacking).** The backdoor account is used to plant a malicious script that steals a colleague's session. NexaCorp detects the outbound theft and applies the recommended fixes (HttpOnly, Content Security Policy), blocks the attacker's known address, and rotates the cracked password. **The backdoor account is not removed.**
4. **INC-2026-007 (IDOR and broken access control).** The backdoor account simply logs in again, walks the employee directory, opens an unprotected admin page, and exports 47 employees' personal data. No exploit is needed.

**The connecting thread is the un-removed backdoor account.** Each individual fix was correct, but the campaign continued because the attacker's identity inside the application survived every remediation. Closing an incident must include removing the access the attacker established, not only patching the flaw that was used.

---

## 4. Risk matrix

Each finding is plotted by business Impact against Likelihood of recurrence. Findings in the upper-right are the most urgent.

| Finding | Impact | Likelihood | Risk rating |
|---|---|---|---|
| Finding 04 - SQL Injection | Very High | High | Critical |
| Finding 05 - Command Injection / Web Shell | Very High | High | Critical |
| Finding 06 - XSS / Session Hijacking | High | Medium (technique now mitigated) | High |
| Finding 07 - IDOR / Broken Access Control | Very High | High | Critical |

(A 5x5 visual matrix is included in the per-incident reports; the table above is the board-level summary.)

---

## 5. Recommendations (prioritised)

**Immediate (this week):**

1. **Remove the unauthorized account and audit the portal user list** against the official IT account register; remove anything that does not match. (Addresses the root cause of INC-2026-007.)
2. **Proceed with the GDPR Article 33 notification** and assess Article 34 communication to the 47 affected employees.

**Short term (this month):**

3. **Enforce authorization on the portal** : verify that a user may see each record they request (fixes IDOR, Finding 07) and that only administrators can reach administrative pages and exports (fixes broken access control, Finding 07).
4. **Eliminate the injection classes at the source** : parameterised database queries (Finding 04) and removal or strict control of the diagnostic command and file features (Finding 05).
5. **Enforce strong credential hygiene** : block weak and breached passwords, migrate password storage to a modern salted hash, and require multi-factor authentication (Finding 04).

**Strategic (this quarter):**

6. **Add behaviour-based detection.** The campaign was largely invisible to the current monitoring, which alerts on noisy scanners but not on a valid account abusing its access. Add detection for directory enumeration, non-administrator access to admin and export functions, and credential reuse.
7. **Change the incident-closure process** : an incident is not closed until the access the attacker established (accounts, web shells, scheduled tasks) is removed and verified, not only the exploited flaw patched.

---

## 6. Appendix (technical)

**Indicators of compromise (campaign):**

- Attacker source address used across INC-2026-004, 005 and 007.
- Unauthorized portal account (the backdoor) introduced at INC-2026-005 and reused at INC-2026-006 and 007.
- Web shell planted on the portal server (INC-2026-005).
- External collection server receiving the stolen session (INC-2026-006).
- Portal endpoints abused at INC-2026-007 : employee directory lookup (IDOR), administrative area, employee export.

**Detection coverage (INC-2026-007 window).** The SIEM raised 211 alerts, all against unrelated scanner traffic (Wazuh rules 31101, 31108, 31151). The actual attack produced no alerts, because every request succeeded under a valid, authenticated session. This is itself a finding and is the basis for recommendation 6.

**Source material.** Per-incident technical detail, exact timestamps, payloads and evidence references are in the individual incident reports INC-2026-004 through INC-2026-007. Raw evidence (logs, network captures, exports) is retained by BeCode Corp and is not redistributed.
