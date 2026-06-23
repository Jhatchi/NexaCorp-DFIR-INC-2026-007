# Investigation Journal: INC-2026-007

Working notes for the NexaPortal IDOR and Broken Access Control investigation. Reconstructed from the evidence bundle after the fact; timestamps are UTC unless noted. NexaCorp is a fictitious BeCode lab scenario.

## Starting point

The bundle shipped evidence only, no README : the NexaPortal Apache access log, the portal authentication log, a Wazuh alert export, a network capture, the administrative export CSV, and the IT-provisioned account register. The reporting email framed an automated walk through the employee directory, access to a restricted admin section, and a bulk download, on Friday 19 June. This is an assessment lab : no detection-rule (Phase 2) work.

## Phase 1: who, when, from where

I reconciled the authentication log against the account register first. One success stood out : m.renard at 16:23:01 from 172.16.50.10. m.renard is not in the register; a real p.renard (Pierre Renard) is, which reads as an account named to blend in. The earlier burst of failed logins from 172.16.50.40 (admin, root, ...) is a separate scanner that never succeeded, easy to mistake for the breach. The real entry was a single valid login. The source 172.16.50.10 is the same address from INC-2026-004 and 005, so this is the continuation of the same campaign through a foothold that was never removed.

## Phase 2: the IDOR scope

I pulled every request from 172.16.50.10 in the access log. From 16:23:08 to 16:25:28 the session walked /portal/employees.php?id=1 to id=47, near-sequential, roughly one record every two to three seconds, every response 200. Counting distinct ids gave 47 records (id=9 and id=23 were re-requested, no new data). The point I had to be able to defend is what makes this an attack rather than a busy HR user : an unauthorized account, a single source, an exhaustive sequential sweep of the whole id space, at a machine cadence. That combination, not the raw view count, is the signal.

## Phase 3: broken access and exfiltration

At 16:37:50 the session opened /portal/admin/ and got 200. The same path returns 302 (redirect to login) for unauthorized scanner traffic in the same log, so the 200 to a non-admin session is the proof of a missing authorization check. At 16:38:01 it fetched /portal/admin/export_users.php : 200, 3858 bytes, exactly the size of the export CSV, confirmed as text/csv in the capture. The CSV holds 47 employee records with name, email, department, role, phone and salary band, so this is a confirmed personal-data exfiltration and GDPR-relevant. The capture also revealed a debug token left in an HTML comment on the admin page : minor, but real information disclosure.

## Phase 4: the detection gap

The Wazuh export held 211 alerts, all from the scanner ranges (rules 31101, 31108, 31151). The attacker produced none : valid login, HTTP 200 throughout, ordinary browser user agent. A signature SIEM watching for 4xx and known-scanner user agents cannot see authorization abuse. The fix is behavioural : enumeration of employees.php?id from one source, or any non-admin account hitting the export endpoint.

## Lessons learned

- The root cause is process, not code : the m.renard foothold should have been removed after INC-2026-005. The XSS patch from INC-2026-006 was correct but irrelevant because the attacker no longer needed XSS.
- Separating noise from the breach mattered : the .40 brute force and the scanner alerts are loud and useless; the actual attack is quiet and valid.
- "How many records" is a GDPR number, so the count had to be exact and defensible, cross-checked between the access log (47 distinct ids) and the CSV (47 rows).
- Signature detection has a structural blind spot for authorization abuse; behavioural rules are the answer, and that is a finding in itself.
