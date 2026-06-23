# MITRE ATT&CK Mapping: INC-2026-007

Mapping of the INC-2026-007 findings to MITRE ATT&CK Enterprise techniques (v15). NexaCorp is a fictitious BeCode lab scenario.

| Finding | Tactic | Technique ID | Technique Name | Observed evidence |
|---|---|---|---|---|
| I1 | Initial Access / Persistence | T1078 | Valid Accounts | Login with the unauthorized but valid m.renard account (foothold not removed after INC-2026-005) |
| I2 | Collection | T1119 | Automated Collection | Scripted sequential enumeration of /portal/employees.php?id=1..47 |
| I2, I3 | Collection | T1213 | Data from Information Repositories | Reading employee records from the portal directory |
| I3 | Exfiltration | T1567 | Exfiltration Over Web Service | Bulk employee export downloaded over the application admin endpoint (export_users.php) |

## Notes on the mapping

- The engagement is an authorization-abuse incident, not a software-exploit incident : the primary technique is T1078 (Valid Accounts), because the attacker held a valid login and required no exploit, malware or stolen session.
- The directory walk is mapped to T1119 (Automated Collection) for the scripted, machine-paced enumeration, and T1213 (Data from Information Repositories) for the nature of the data read.
- The bulk export is mapped to T1567 (Exfiltration Over Web Service) : the data left over the application's own web channel.
- Finding I4 (debug token disclosed in an HTML comment) is an information-disclosure weakness (CWE-615 / CWE-200) and is not mapped to an adversary technique.
- Framework version : MITRE ATT&CK Enterprise v15.
