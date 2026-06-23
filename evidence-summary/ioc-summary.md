# Indicators of Compromise: INC-2026-007

Indicators observed during the investigation of the NexaPortal IDOR and Broken Access Control incident (19 June 2026). NexaCorp is a fictitious BeCode lab scenario on isolated infrastructure; these indicators are not real-world threat intelligence. The `Source` column records where each indicator was observed in the evidence bundle.

## Network indicators

| Type | Value | Source |
|---|---|---|
| Attacker source IP | 172.16.50.10 | nexaportal_auth.log, web_access.log, pcap |
| Failed-login brute-force source (noise) | 172.16.50.40 | nexaportal_auth.log |
| Scanner noise sources | 172.16.50.20-26 | web_access.log, wazuh-alerts.json |
| Cross-incident link | 172.16.50.10 also used in INC-2026-004 and INC-2026-005 | series reports |

## Host and application indicators

| Type | Value | Source |
|---|---|---|
| Target application | NexaPortal (Apache 2.4.57, Debian) | pcap (Server header) |
| Target host | 192.168.10.26 (Wazuh agent lab07-nexaportal) | web_access.log, wazuh-alerts.json |
| IDOR endpoint | /portal/employees.php?id=<N> | web_access.log |
| Broken-access endpoint | /portal/admin/ (HTTP 200 to non-admin session; 302 to unauthorized) | web_access.log |
| Exfiltration endpoint | /portal/admin/export_users.php (HTTP 200, 3858 bytes) | web_access.log, pcap |
| Information disclosure | debug token in /portal/admin/ HTML comment (value `[REDACTED]`) | pcap |

## Behavioral indicators

| Type | Value | Source |
|---|---|---|
| Automated directory enumeration | sequential id=1..47, ~1 record / 2-3 s, single source, all HTTP 200 | web_access.log |
| Session window | login 16:23:01 UTC to export 16:38:01 UTC (15 minutes) | web_access.log |
| User agent | Chrome on Windows (consistent across attacker requests) | web_access.log, pcap |

## Account indicators

| Type | Value | Source |
|---|---|---|
| Unauthorized account (foothold) | m.renard (absent from IT account register; foothold since INC-2026-005) | nexaportal_auth.log, employee_roster.txt |
| Look-alike legitimate account | p.renard (Pierre Renard) - surname the backdoor mimics | employee_roster.txt |
| Cross-incident account in exposed data | j.martin (account dumped via SQLi in INC-2026-004) | employee_directory_export.csv |

## Exposed data

| Type | Value | Source |
|---|---|---|
| Records exfiltrated | 47 distinct employee records | employee_directory_export.csv |
| Fields per record | name, corporate email, department, job title, phone number, salary band | employee_directory_export.csv |
| Highest-sensitivity exposure | top salary band (D1) including the CFO and Sales Director roles, and the Data Protection Officer | employee_directory_export.csv |
| Data protection | personal data under the GDPR; confirmed exfiltration, Article 33 notification applicable | analysis |

> No credentials are stored in clear text in this repository. The debug token recovered from the admin page is redacted; the formal PDF report is the authoritative deliverable.
