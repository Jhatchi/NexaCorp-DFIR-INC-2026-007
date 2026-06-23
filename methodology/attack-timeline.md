# Attack Timeline: INC-2026-007

Chronology of the NexaPortal IDOR and Broken Access Control incident. Log timestamps are UTC (+0000); local time is CEST (UTC+02:00). Sources are noted per row. NexaCorp is a fictitious BeCode lab scenario.

| Time (UTC) | Time (CEST) | Event | Source |
|---|---|---|---|
| 19 Jun 10:11-10:12 | 12:11-12:12 | Credential brute force against /portal/login.php from 172.16.50.40 (admin, root, administrator, ...), all FAILED. Unrelated scanner noise, never succeeded. | nexaportal_auth.log |
| 19 Jun 16:23:01 | 18:23:01 | Successful login of m.renard from 172.16.50.10. Account absent from the IT account register (unauthorized foothold). | nexaportal_auth.log, web_access.log |
| 19 Jun 16:23:03 | 18:23:03 | Dashboard loaded, authenticated session established. | web_access.log |
| 19 Jun 16:23:08 - 16:25:28 | 18:23:08 - 18:25:28 | Automated IDOR enumeration of /portal/employees.php?id=1 through id=47, near-sequential, ~1 record every 2-3 s, every response HTTP 200. 47 distinct employee records accessed. | web_access.log |
| 19 Jun 16:37:50 | 18:37:50 | GET /portal/admin/ returns HTTP 200 to the non-admin session (broken access control; 302 redirect is served to unauthorized requests). | web_access.log |
| 19 Jun 16:38:01 | 18:38:01 | GET /portal/admin/export_users.php returns HTTP 200, 3858 bytes (text/csv) : bulk export of 47 employee records. Exfiltration event. | web_access.log, pcap |

The intrusion spans 15 minutes from login to data export. A pause of about 12 minutes separates the end of the directory enumeration (16:25:28) from the administrative access (16:37:50). The Wazuh SIEM raised 211 alerts during the window, all against unrelated scanner noise; the attacker source (172.16.50.10) produced zero alerts because every request returned HTTP 200 on an authenticated session.

Cross-incident note: 172.16.50.10 is the same source observed in INC-2026-004 (SQL injection) and INC-2026-005 (command injection, web shell, SSH pivot). The m.renard account traces to the foothold established in INC-2026-005 and was never removed.
