---
layout: post
title: "AWS Cloud Incident Response — ExfilCola"
tags: [Incident Response]
excerpt: "A simulated AWS breach via the Wiz Cloud Hunting Games CTF — IAM lateral movement, Lambda code injection, CloudTrail forensics, overlayfs log recovery, and cron-based persistence."
---

> **Wiz Cloud Hunting Games CTF** · Cloud Forensics · MITRE ATT&CK · AWS IR  
> *Techniques: IAM lateral movement · Lambda code injection · CloudTrail analysis · Overlayfs log recovery · Cron persistence*

---

## What happened

A fictional startup called ExfilCola receives a data extortion email from a threat group calling themselves FizzShadows. They claim to have stolen the company's proprietary soda recipe and are demanding 75 Bitcoin to keep it private. No prior alert fired. No detection triggered. The investigation starts from a ransom note and a set of AWS logs.

This write-up covers my investigation of the Wiz Cloud Hunting Games CTF — a hands-on cloud incident response challenge built around a realistic AWS breach scenario. What makes it worth documenting isn't the fictional wrapper. It's that every technique the attacker used — IAM lateral movement, Lambda code injection, SSH pivoting, log deletion — is entirely realistic and maps cleanly to what defenders encounter in production environments.

The investigation ran backwards, as IR always does. Starting from the confirmed exfiltration, tracing back through a compromised IAM identity, a manipulated Lambda, an EC2 host, SSH lateral movement from a PostgreSQL service, and eventually to a persistent cron-based foothold that had been quietly harvesting credentials the entire time. The sections below map the full chain to MITRE ATT&CK, call out the architectural failures at each stage, and draw out what should have existed instead.

---

## ATT&CK coverage

The attack spans the full ATT&CK kill chain — from initial access through to exfiltration and persistence — using only legitimate AWS APIs and standard Linux tooling. No exploits, no zero-days. Every action the attacker took is indistinguishable from normal operations without behavioural baselines or anomaly detection in place. That's what makes cloud intrusions hard to catch in real time and why logging hygiene matters so much after the fact.

| Technique ID | Name | Tactic | Observed |
|---|---|---|---|
| [T1078](https://attack.mitre.org/techniques/T1078/) | Valid Accounts | Initial Access | Compromised IAM user credentials used to gain foothold |
| [T1053](https://attack.mitre.org/techniques/T1053/) | Scheduled Task/Job | Persistence | Cron job under postgres user — base64-encoded pg_sched script |
| [T1021](https://attack.mitre.org/techniques/T1021/) | Remote Services | Lateral Movement | SSH from PostgreSQL host into ssh-fetcher EC2 |
| [T1552](https://attack.mitre.org/techniques/T1552/) | Unsecured Credentials | Credential Access | IAM credentials harvested and streamed to external C2 |
| [T1548](https://attack.mitre.org/techniques/T1548/) | Abuse Elevation Control Mechanism | Privilege Escalation | AssumeRole chain across IAM identities to reach lambdaWorker and Moe.Jito |
| [T1069](https://attack.mitre.org/techniques/T1069/) | Permission Groups Discovery | Discovery | ListAttachedUserPolicies — enumerating accessible permissions |
| [T1098](https://attack.mitre.org/techniques/T1098/) | Account Manipulation | Persistence | CreateAccessKey / DeleteAccessKey — credential manipulation |
| [T1059](https://attack.mitre.org/techniques/T1059/) | Command and Scripting Interpreter | Execution | logger.py — outbound SSL socket to fetcher.exfilcola.io:8443 |
| [T1526](https://attack.mitre.org/techniques/T1526/) | Cloud Service Discovery | Discovery | ListBuckets, ListObjects — scoping ExfilCola's storage |
| [T1530](https://attack.mitre.org/techniques/T1530/) | Data from Cloud Storage Object | Exfiltration | Bulk GetObject on recipe bucket — PutObject to plant ransom letter |
| [T1070](https://attack.mitre.org/techniques/T1070/) | Indicator Removal | Defence Evasion | Auth log deletion from /var/log — recovered via overlayfs |

---

## The attack chain

Working backwards from the extortion email to the initial entry point, the full chain looked like this:

```
PostgreSQL host — initial foothold (method not captured in logs)
        ↓
Cron persistence established — pg_sched script under postgres user
IAM credentials harvested via ncat → FizzShadows C2 (34.118.239.100:4444)
        ↓
SSH lateral movement into ssh-fetcher EC2 (i-0a44002eec2f16c25)
Auth logs deleted from /var/log to cover tracks
        ↓
IAM lateral movement through harvested credentials
lambdaWorker role reached
        ↓
UpdateFunctionCode → credsrotator Lambda injected with malicious code
IAM keys exfiltrated via logger.py — SSL socket to fetcher.exfilcola.io:8443
        ↓
Moe.Jito credentials obtained via IAM lateral movement
ListAttachedUserPolicies — scoping available permissions
AssumeRole → S3Reader/drinks session
        ↓
Reconnaissance: ListBuckets, ListObjects
GetObject — bulk read of recipe bucket
PutObject — ransom letter planted in soda-vault
        ↓
Ransom email sent to ExfilCola
```

> The initial access method into the PostgreSQL host was not captured in the available log sources — itself a gap in visibility that reflects a missing detective control.

---

## Key findings

**Finding 1 — Identity pivots disappear if you search the wrong field**

Querying CloudTrail for the S3Reader ARN returned a single result: a `ListBuckets` call with a blank `useridentity_username`. A blank username on an `AssumedRole` identity type means temporary STS credentials — the attacker had already moved on from their original identity. Searching the identity columns further would have been a dead end.

The break came from searching `requestparameters` instead. When an `AssumeRole` call is made, the target role is recorded in the parameters, not in the caller's identity fields. Filtering for the S3Reader role name and session name `drinks` across both request and response parameters surfaced the event and identified the underlying user: `Moe.Jito`. Identity columns show who called. Parameters show what they became. Every hop in a lateral movement chain is invisible without that distinction.

**Finding 2 — Lambda code injection weaponises trust that already exists**

Tracing further back, the `UpdateFunctionCode` event against the `credsrotator` Lambda — a legitimate credential rotation function — was called from the `lambdaWorker` role attached to the compromised EC2 instance. The function already held elevated Secrets Manager permissions by design. Injecting malicious code into it rather than creating new infrastructure meant the attacker inherited existing trust and an execution pattern indistinguishable from normal operations.

There was no new resource to detect. No unusual IAM entity. No code signing. No alerting on `UpdateFunctionCode` callers. The Lambda ran as it always had — just doing something different.

**Finding 3 — Deleting logs is only effective if you understand the filesystem**

On the EC2 host, auth logs had been removed from `/var/log` — a reasonable countermeasure. But running `findmnt` revealed the filesystem was overlayfs with multiple mounts stacked at the same path. The attacker's deletion only affected the top-most writable layer. Running `umount /var/log` peeled it back, exposing the read-only lower layer with the original auth logs intact.

Filtering those logs for accepted SSH connections under `postgresql-user` confirmed lateral movement from the PostgreSQL host into ssh-fetcher. The attacker's OPSEC assumption about log deletion didn't account for containerised filesystem layers — and that assumption is what broke the chain open.

**Finding 4 — The persistence mechanism revealed the full operation**

On the PostgreSQL host, a cron job under the `postgres` user referenced a file called `pg_sched` — named to blend in as a legitimate scheduler task. Base64-decoding it produced a bash script that enumerated attached IAM policies across the environment looking for overpermissioned roles, searched for SSH keys to spread laterally, streamed harvested credentials via `ncat` to the C2, and cleared its own bash history on exit.

Credentials embedded in the script authenticated to a REST API on the C2 host at `34.118.239.100`. The stolen recipe file was listed there, confirming the exfiltration claim and providing the path to remediation — a single authenticated DELETE request against the attacker's own server.

---

## Architectural lessons

**IAM permissions had drifted beyond their intended scope.** The `S3Reader` role held `PutObject` permissions beyond its read function — something that may not have been intentional but had real consequences when the role was compromised. The `credsrotator` Lambda's execution role had Secrets Manager write access scoped to `*` rather than the specific ARNs required for credential rotation. The `lambdaWorker` role had no trust condition on who could call `UpdateFunctionCode`. Scoping each of these to their actual functional requirements — rather than what they might ever need — would have reduced the attacker's options at multiple points in the chain. Regular IAM access reviews and permission boundary enforcement are practical ways to catch this kind of drift before it becomes a liability.

**Cron-based persistence was not surfaced by available monitoring.** A scheduled task under the `postgres` user was harvesting credentials and maintaining an outbound channel to an external host. Based on the available evidence, this activity was not detected during the period it was active. Host-based controls such as auditd or file integrity monitoring on cron directories, combined with alerting on scheduled task creation under service accounts, would provide visibility into this class of technique. This is particularly relevant on hosts that handle credentials or have network access to sensitive resources.

**Outbound connections to external hosts were not flagged in the affected systems.** Two separate outbound channels were established — `logger.py` to an external host on port 8443, and the `pg_sched` script to a C2 on port 4444. Based on the available evidence, neither was detected at the network layer. Reviewing egress controls and alerting on connections to unknown external destinations on sensitive hosts would reduce the window in which this kind of activity goes unnoticed.

**Two controls materially aided the investigation and are worth highlighting.** CloudTrail management events being enabled and intact allowed the full IAM lateral movement chain to be reconstructed — without them, the identity behind the S3Reader session would have been unattributable. S3 data events being enabled — which is not the AWS default and requires a deliberate configuration decision — made it possible to confirm the exfiltration, identify the responsible ARN, and quantify what was taken. These are good examples of logging decisions that directly translate into investigative capability when they are needed. The overlayfs filesystem structure on the EC2 host also proved significant: auth logs the attacker had deleted were preserved in a lower filesystem layer and recovered via `umount`, confirming the SSH lateral movement path.

---

## Architectural gaps — summary

| Gap | What happened | What should exist |
|---|---|---|
| Host-level monitoring | Cron persistence ran undetected on PostgreSQL host | EDR or auditd on all hosts; alerting on cron job creation under service accounts |
| Credential hygiene | Plaintext credentials hardcoded in a cron script | Secrets Manager for all credentials; no secrets on disk |
| Egress filtering | Unrestricted outbound from EC2 and PostgreSQL host | Security group egress rules; alerting on connections to unknown external IPs |
| S3 role permissions | S3Reader held PutObject — name and permissions misaligned | IAM roles scoped strictly to declared function; regular permission audits |
| Lambda code integrity | No controls on UpdateFunctionCode callers | Code signing enforced; CloudWatch alert on unexpected UpdateFunctionCode |
| Lambda execution role | credsrotator had broad Secrets Manager write access | Permissions scoped to specific secret ARNs only |
| STS session policies | Assumed sessions inherited full role permissions | Session policies applied at assumption time to restrict scope |
| Detection coverage | No alerts fired end-to-end across the intrusion | GuardDuty with anomaly baselines; CloudTrail S3 data events enabled |

---

*Part of my [security portfolio](https://github.com/am-netizen) — mapping hands-on work to real-world attack patterns.*
