# NexaCorp DFIR: INC-2026-007 - IDOR and Broken Access Control

Assessment of an authorization-abuse incident against NexaPortal, NexaCorp's internal employee web portal. On 19 June 2026 an attacker logged in with a valid but unauthorized account (`m.renard`, the foothold left in place since INC-2026-005), walked the entire employee directory through an Insecure Direct Object Reference (IDOR) on `/portal/employees.php?id`, reached an administrative page that returned HTTP 200 to a non-administrator session (broken access control), and downloaded a full employee export. 47 employees' personal data left the portal. No exploit, no malware, no stolen session were needed. Conducted as a solo engagement during the BeCode Brussels Blue & Red Team bootcamp (Mission 07), as the continuation of [INC-2026-001](https://github.com/Jhatchi/NexaCorp-DFIR-INC-2026-001), [INC-2026-002](https://github.com/Jhatchi/NexaCorp-DFIR-INC-2026-002), [INC-2026-003](https://github.com/Jhatchi/NexaCorp-DFIR-INC-2026-003), [INC-2026-004](https://github.com/Jhatchi/NexaCorp-DFIR-INC-2026-004), [INC-2026-005](https://github.com/Jhatchi/NexaCorp-DFIR-INC-2026-005), and [INC-2026-006](https://github.com/Jhatchi/NexaCorp-DFIR-INC-2026-006). This repository also carries the consolidated **Month 2 Assessment Report** (INC-2026-004 to 007).

[![ci](https://github.com/Jhatchi/NexaCorp-DFIR-INC-2026-007/actions/workflows/ci.yml/badge.svg)](https://github.com/Jhatchi/NexaCorp-DFIR-INC-2026-007/actions/workflows/ci.yml)
[![Methodology](https://img.shields.io/badge/methodology-NIST%20SP%20800--61r2-blue.svg)](#methodology)
[![Framework](https://img.shields.io/badge/framework-MITRE%20ATT%26CK-red.svg)](https://attack.mitre.org/)
[![CWE](https://img.shields.io/badge/CWE--639-IDOR-orange.svg)](https://cwe.mitre.org/data/definitions/639.html)
[![License](https://img.shields.io/badge/license-MIT-yellow.svg)](LICENSE)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Johan--Emmanuel%20Hatchi-0A66C2?logo=linkedin&logoColor=white)](https://www.linkedin.com/in/johan-emmanuel-hatchi/)

This repository documents a SOC analyst engagement carried out as part of the BeCode Cybersecurity Bootcamp (promotion 2025-2026). It is an assessment lab: a forensic investigation and report, with no detection-rule (Phase 2) deliverable. It is the seventh incident in the NexaCorp DFIR series and the Month 2 capstone.

## Contents

- [Operational notice](#operational-notice)
- [At a glance](#at-a-glance)
- [Engagement context](#engagement-context)
- [Executive summary](#executive-summary)
- [Kill chain summary](#kill-chain-summary)
- [How to read this repository](#how-to-read-this-repository)
- [Methodology](#methodology)
- [Tools used](#tools-used)
- [Detection engineering](#detection-engineering)
- [Repository layout](#repository-layout)
- [Reproducibility](#reproducibility)
- [Known limits](#known-limits)
- [NexaCorp DFIR series](#nexacorp-dfir-series)
- [Acknowledgments](#acknowledgments)
- [About](#about)
- [License](#license)

---

## Operational notice

This is a training engagement against fictitious infrastructure. NexaCorp Industries is a fictional client used as the scenario for the BeCode Brussels bootcamp. NexaPortal (host `192.168.10.26`) is an isolated lab environment. No production system, customer data, or third party was involved.

All IP addresses, hostnames, and accounts referenced here (`172.16.50.10`, `192.168.10.26`, `m.renard`, `p.renard`, `j.martin`, and similar) are lab-local artifacts, not real-world threat intelligence. Do not feed them to a production SIEM as IOCs.

---

## At a glance

| Field | Value |
|---|---|
| Reference | INC-2026-007 |
| Affected host | NexaPortal (Apache 2.4.57, 192.168.10.26) |
| Attacker | 172.16.50.10 (external; same source as INC-2026-004 and INC-2026-005) |
| Root cause | Un-removed foothold (account m.renard) plus IDOR (CWE-639) and broken access control (CWE-862) |
| Outcome | 47 employees' personal data exfiltrated; GDPR Article 33 notification applicable |
| Date of incident | 19 June 2026 |
| Evidence | web_access.log, nexaportal_auth.log, wazuh-alerts.json, nexaportal_idor.pcap, employee_directory_export.csv, employee_roster.txt |
| Phases | Phase 1 assessment only (no detection-engineering phase) |
| Related incidents | INC-2026-004, INC-2026-005, INC-2026-006 (web application campaign) |

| Investigation output | Value |
|---|---|
| Attacker IP | 172.16.50.10 (external) |
| Unauthorized account | m.renard (absent from the IT account register; foothold since INC-2026-005) |
| IDOR endpoint and parameter | /portal/employees.php, parameter id (CWE-639) |
| Records exposed | 47 distinct employee records |
| Broken-access endpoint | /portal/admin/ (HTTP 200 to a non-admin session; 302 to unauthorized requests) |
| Exfiltration endpoint | /portal/admin/export_users.php (47-record export, GDPR-reportable) |
| SIEM detection gap | 211 Wazuh alerts in the window, 0 on the attacker (authorization abuse is invisible to signature rules) |

---

## Engagement context

**Scenario (fictional).** NexaCorp Industries reported an employee-data exposure on NexaPortal. The engagement was reported by Marc Wauters (IT Infrastructure Manager); the incident occurred on Friday 19 June 2026. NexaPortal had been launched without a security review, and the controls deployed after INC-2026-006 (HttpOnly cookies, Content Security Policy) were live but irrelevant here, because the attacker held a valid login rather than relying on XSS.

**Scope.** This is an assessment lab: a forensic analysis of the evidence bundle (web and authentication logs, network capture, Wazuh alert export, the administrative export CSV, and the IT account register). There is no detection-rule (Phase 2) deliverable; the Wazuh detection gap is documented as a finding.

**Capstone.** The engagement also consolidates the month's four web application incidents (INC-2026-004 to 007) into the [Month 2 Assessment Report](reports/Month2_Assessment_Report.md) for management.

**Educational context.** Delivered during the BeCode Brussels Blue & Red Team bootcamp (November 2025 to September 2026) as Mission 07.

---

## Executive summary

On 19 June 2026, an attacker authenticated to NexaPortal with `m.renard`, an account that does not exist in NexaCorp's IT-provisioned register. It is the same unauthorized foothold first seen in INC-2026-006 and traced to INC-2026-005, and it was never removed. From a single external source (172.16.50.10, the same address used in INC-2026-004 and 005), the session enumerated the employee directory through an IDOR, reading 47 distinct records in just over two minutes, then reached an administrative page that returned HTTP 200 to a non-administrator session and downloaded a full employee export.

The exported data covers 47 employees (name, corporate email, department, job title, phone number, salary band) and is personal data under the GDPR, so a breach notification is applicable. The entire attack was invisible to the SIEM: of 211 Wazuh alerts in the window, none concern the attacker, because every request returned HTTP 200 on an authenticated session. The root cause is not the IDOR flaw in isolation, it is the failure to remove the attacker's account after INC-2026-005.

---

## Kill chain summary

The attacker read and exported employee data using nothing but a valid login:

1. **Valid login**: `m.renard` authenticates from 172.16.50.10 at 16:23:01 UTC. The account is absent from the IT register (foothold from INC-2026-005, never removed).
2. **IDOR enumeration**: from 16:23:08 to 16:25:28 UTC the session walks `/portal/employees.php?id=1` through `id=47`, near-sequential, every response HTTP 200. 47 distinct records accessed.
3. **Broken access control**: at 16:37:50 UTC `/portal/admin/` returns HTTP 200 to the non-admin session (unauthorized requests receive a 302 redirect).
4. **Exfiltration**: at 16:38:01 UTC `/portal/admin/export_users.php` returns the 47-record employee export.

A noise brute-force from a different source (172.16.50.40) failed throughout and is unrelated; the actual breach is the single valid login from 172.16.50.10.

---

## How to read this repository

| If you are a... | Start here | Time |
|---|---|---|
| **Recruiter or hiring manager** | This README + the [report](reports/INC-2026-007_Findings_Report.md) executive summary | 5 min |
| **Management / board** | The [Month 2 Assessment Report](reports/Month2_Assessment_Report.md) (non-technical, business impact, GDPR position) | 10 min |
| **SOC analyst evaluating fit** | [Report](reports/INC-2026-007_Findings_Report.md) technical analysis + [`evidence-summary/ioc-summary.md`](evidence-summary/ioc-summary.md) | 20 min |
| **DFIR practitioner** | Full [report](reports/INC-2026-007_Findings_Report.md) + [`methodology/attack-timeline.md`](methodology/attack-timeline.md) + [`notes/journal.md`](notes/journal.md) | 30 min |
| **Anyone who wants to grep, cite, or diff** | [Markdown source of the report](reports/INC-2026-007_Findings_Report.md) | as needed |

---

## Methodology

The engagement follows standard incident-response frameworks:

- **NIST SP 800-61r2** (incident handling) and **NIST SP 800-86** (forensic techniques): structure the Detection & Analysis work.
- **SANS PICERL**: the Identification stage is the core of the assessment (log and capture correlation).
- **MITRE ATT&CK Enterprise v15**: every finding is mapped to one or more techniques (see [`methodology/attck-mapping.md`](methodology/attck-mapping.md)).
- Weakness classification: **CWE-639** (IDOR) and **CWE-862** (Missing Authorization), under **OWASP Top 10 A01:2021 Broken Access Control**.

**Investigation approach.** Account and source identification (authentication log against the IT register), activity reconstruction (the IDOR walk and admin access from the access log, with the record count cross-checked against the export CSV), exfiltration proof (capture and CSV), and a detection-gap analysis of the Wazuh alerts. See [`notes/journal.md`](notes/journal.md).

**Evidence and timestamps.** Log timestamps are UTC; local time is CEST (UTC+2). Timestamps are given in both where it aids the timeline.

---

## Tools used

- **tshark**: network capture analysis (confirming the export response as `text/csv`, recovering the admin page HTML).
- **jq**: querying and aggregating the Wazuh alert export.
- **grep / awk / sort / uniq**: access-log and authentication-log analysis, distinct-record counting, reconciliation against the account register.
- **base64**: decoding the debug token disclosed in the admin page source.

---

## Detection engineering

**Not applicable for this engagement.** This is an assessment lab with no Phase 2: no Suricata or other detection rules were authored. The detection question is instead handled as a finding. During the incident window the Wazuh SIEM raised 211 alerts, all against unrelated scanner traffic (rules 31101, 31108, 31151); the attacker (172.16.50.10) generated zero alerts, because every request returned HTTP 200 on a valid, authenticated session. A signature ruleset keyed on 4xx errors and known-scanner user agents is structurally blind to authorization abuse. The report recommends behavioural detection (directory enumeration from one source; any non-admin account reaching the admin or export endpoints) as the appropriate control.

---

## Repository layout

```
NexaCorp-DFIR-INC-2026-007/
├── README.md                      This file
├── LICENSE
├── .gitignore
├── .markdownlint.json
├── .github/
│   └── workflows/
│       └── ci.yml                 markdownlint + typography validation
├── reports/
│   ├── INC-2026-007_Findings_Report.pdf   Full findings report
│   ├── INC-2026-007_Findings_Report.md    Markdown source of the report
│   ├── Month2_Assessment_Report.pdf       Month 2 capstone (INC-2026-004 to 007), board audience
│   └── Month2_Assessment_Report.md        Markdown source of the Month 2 report
├── evidence-summary/
│   └── ioc-summary.md             Indicators of compromise
├── methodology/
│   ├── attack-timeline.md         Request-level timeline (UTC and CEST)
│   └── attck-mapping.md           MITRE ATT&CK mapping table
└── notes/
    └── journal.md                 Investigation journal (post-analysis reconstruction)
```

This is an assessment lab, so there is no `detection/` directory. The findings report is provided as a PDF in `reports/`, alongside its Markdown source.

---

## Reproducibility

The evidence bundle is BeCode lab property and is not redistributed. With your own copy, the core findings are reproducible from the logs:

```bash
# Attacker account : a successful login absent from the IT register
grep SUCCESS logs/nexaportal_auth.log | grep -v -f <(awk -F'|' '{gsub(/ /,"",$1);print $1}' reference/employee_roster.txt)

# IDOR scope : distinct employee records read by the attacker
grep '172.16.50.10' logs/web_access.log | grep -oE 'employees\.php\?id=[0-9]+' | sort -u | wc -l

# Broken access control : 200 to the attacker vs 302 to unauthorized scanners on /portal/admin/
grep '/portal/admin/' logs/web_access.log
```

The export size returned by `export_users.php` (3858 bytes) matches `exports/employee_directory_export.csv` exactly; the capture confirms the response as `text/csv`.

---

## Known limits

- **Foothold origin spans incidents.** The `m.renard` account was flagged with undetermined origin in INC-2026-006; this engagement treats it as the foothold established in INC-2026-005 per the campaign chain, while the report notes both the evidence here and that cross-incident framing.
- **No deployed detection.** Being an assessment lab, the report proposes behavioural detection logic but does not ship or validate rules; the proposal is prose, not a tested ruleset.
- **Scope is the captured window.** The assessment covers the 19 June evidence bundle; activity outside that window is out of scope.

---

## NexaCorp DFIR series

- [INC-2026-001](https://github.com/Jhatchi/NexaCorp-DFIR-INC-2026-001): Linux infrastructure compromise (vsftpd backdoor, Caldera C2)
- [INC-2026-002](https://github.com/Jhatchi/NexaCorp-DFIR-INC-2026-002): privilege escalation and persistence (Tor SSH, SUID, backdoor account)
- [INC-2026-003](https://github.com/Jhatchi/NexaCorp-DFIR-INC-2026-003): month-1 cross-incident assessment
- [INC-2026-004](https://github.com/Jhatchi/NexaCorp-DFIR-INC-2026-004): SQL injection (web portal)
- [INC-2026-005](https://github.com/Jhatchi/NexaCorp-DFIR-INC-2026-005): OS command injection and web shell (web portal)
- [INC-2026-006](https://github.com/Jhatchi/NexaCorp-DFIR-INC-2026-006): stored XSS and session hijacking (NexaPortal)
- **INC-2026-007**: this repository (IDOR and broken access control; Month 2 capstone)
- [INC-2026-008](https://github.com/Jhatchi/NexaCorp-DFIR-INC-2026-008): AD reconnaissance and Kerberoasting (first incident of Month 3)

---

## Acknowledgments

- **Thomas B.** (BeCode lab coach): scenario design and publication authorization for portfolio use.
- **MITRE** for the ATT&CK knowledge base used to map every finding.
- **OWASP** for the Top 10 and the access-control guidance referenced in the remediation.

---

## About

Solo DFIR engagement delivered during the [BeCode Brussels](https://becode.org) Blue & Red Team bootcamp (November 2025 to September 2026), Mission 07.

Author: **[Johan-Emmanuel Hatchi](https://github.com/Jhatchi)** ([LinkedIn](https://www.linkedin.com/in/johan-emmanuel-hatchi/)).

Open to cybersecurity internship opportunities starting September 2026 in Belgium. Looking for SOC / DFIR / detection engineering roles where this kind of end-to-end work (log and PCAP forensics, authorization-abuse analysis, GDPR-aware impact assessment, formal client reporting) is in scope.

---

## License

[MIT](LICENSE), 2026 Johan-Emmanuel Hatchi.

The report text and methodology notes are released under MIT: free to copy, adapt, and reuse with attribution. The evidence bundle, lab infrastructure, and original engagement briefings remain BeCode Brussels property and are not redistributed.
