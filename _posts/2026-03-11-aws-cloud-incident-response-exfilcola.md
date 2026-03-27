---
layout: post
title: "AWS Cloud Incident Response — ExfilCola"
categories: [Cloud Security, Incident Response, Security Architecture]
tags: [Incident Response]
excerpt: "A realistic AWS breach simulation covering IAM pivots, Lambda tampering, SSH movement, and what CloudTrail revealed during incident response."
---

> **Wiz Cloud Hunting Games CTF** · Cloud Forensics · MITRE ATT&CK · AWS IR  
> *Techniques: IAM lateral movement, Lambda code injection, CloudTrail analysis, overlayfs log recovery, cron persistence*

---

## What happened

A fictional startup called ExfilCola receives an extortion email from a threat group called FizzShadows. They claim to have stolen the company's proprietary soda recipe and demand 75 Bitcoin. There are no alerts, no detections, and no obvious entry event. The investigation starts with the ransom note and AWS logs.

This write-up covers my investigation of the Wiz Cloud Hunting Games CTF, a hands-on cloud incident response challenge built around a realistic AWS breach. The fictional scenario is not the important part. The attacker behavior is. IAM role pivots, Lambda tampering, SSH movement, and log deletion are all patterns defenders see in real environments.

I worked this backwards from exfiltration to initial foothold. The path moved through a compromised IAM identity, a modified Lambda function, an EC2 host, SSH access from a PostgreSQL service, and a cron-based persistence mechanism that was quietly collecting credentials. The sections below map that chain to ATT&CK and focus on practical control gaps.

---

## ATT&CK coverage

The attack covers most of the ATT&CK cloud kill chain, from initial access to exfiltration and persistence, using legitimate AWS APIs and standard Linux tooling. There are no exotic exploits here. Most actions look normal unless you have baseline behavior and good alerting. That is exactly why cloud incidents are hard to catch early and why strong logs matter.

| Technique ID | Name | Tactic | Observed |
|---|---|---|---|
| [T1078](https://attack.mitre.org/techniques/T1078/) | Valid Accounts | Initial Access | Compromised IAM user credentials used to gain foothold |
| [T1053](https://attack.mitre.org/techniques/T1053/) | Scheduled Task/Job | Persistence | Cron job under postgres user with a base64-encoded `pg_sched` script |
| [T1021](https://attack.mitre.org/techniques/T1021/) | Remote Services | Lateral Movement | SSH from PostgreSQL host into ssh-fetcher EC2 |
| [T1552](https://attack.mitre.org/techniques/T1552/) | Unsecured Credentials | Credential Access | IAM credentials harvested and streamed to external C2 |
| [T1548](https://attack.mitre.org/techniques/T1548/) | Abuse Elevation Control Mechanism | Privilege Escalation | AssumeRole chain across IAM identities to reach lambdaWorker and Moe.Jito |
| [T1069](https://attack.mitre.org/techniques/T1069/) | Permission Groups Discovery | Discovery | `ListAttachedUserPolicies` calls used to enumerate permissions |
| [T1098](https://attack.mitre.org/techniques/T1098/) | Account Manipulation | Persistence | CreateAccessKey / DeleteAccessKey for credential manipulation |
| [T1059](https://attack.mitre.org/techniques/T1059/) | Command and Scripting Interpreter | Execution | `logger.py` opening an outbound SSL socket to `fetcher.exfilcola.io:8443` |
| [T1526](https://attack.mitre.org/techniques/T1526/) | Cloud Service Discovery | Discovery | `ListBuckets` and `ListObjects` calls to scope storage |
| [T1530](https://attack.mitre.org/techniques/T1530/) | Data from Cloud Storage Object | Exfiltration | Bulk `GetObject` reads from the recipe bucket, then `PutObject` for ransom note |
| [T1070](https://attack.mitre.org/techniques/T1070/) | Indicator Removal | Defence Evasion | Auth log deletion from `/var/log`, later recovered via overlayfs |

---

## The attack chain

Working backward from the extortion email to the initial foothold, the chain looked like this:

```
PostgreSQL host (initial foothold, method not captured in available logs)
        ↓
Cron persistence established via `pg_sched` under the postgres user
IAM credentials harvested via ncat → FizzShadows C2 (34.118.239.100:4444)
        ↓
SSH lateral movement into ssh-fetcher EC2 (i-0a44002eec2f16c25)
Auth logs deleted from /var/log
        ↓
IAM lateral movement through harvested credentials
lambdaWorker role reached
        ↓
UpdateFunctionCode on `credsrotator` Lambda with malicious payload
IAM keys exfiltrated via logger.py over SSL to fetcher.exfilcola.io:8443
        ↓
Moe.Jito credentials obtained via IAM lateral movement
ListAttachedUserPolicies to scope available permissions
AssumeRole into S3Reader/drinks session
        ↓
Reconnaissance: ListBuckets, ListObjects
GetObject bulk read of recipe bucket
PutObject used to plant ransom letter in soda-vault
        ↓
Ransom email sent to ExfilCola
```

> The initial access method on the PostgreSQL host was not captured in available logs, which itself points to a visibility gap.

---

## Key findings

**Finding 1: Identity pivots disappear if you search the wrong field**

Querying CloudTrail for the S3Reader ARN returned a single result: a `ListBuckets` call with a blank `useridentity_username`. On an `AssumedRole` identity, that usually means temporary STS credentials. In other words, the actor had already pivoted and identity-only searches would stall.

The useful trail was in `requestparameters`. For `AssumeRole`, the target role is logged in parameters, not caller identity fields. Filtering by the role name and session name `drinks` in request and response parameters surfaced the pivot and identified the underlying user `Moe.Jito`. Identity fields tell you who made the call. Parameters tell you what they became.

**Finding 2: Lambda code injection abuses existing trust**

Tracing further back, the `UpdateFunctionCode` event on `credsrotator` came from the `lambdaWorker` role on the compromised EC2 instance. `credsrotator` was a legitimate function with broad Secrets Manager permissions. By modifying existing code instead of deploying new infrastructure, the attacker stayed inside normal trust boundaries and blended with routine operations.

There was no new resource, no unusual IAM entity, no code-signing enforcement, and no alerting on unexpected `UpdateFunctionCode` callers. The function looked familiar while behaving differently.

**Finding 3: Log deletion failed because of filesystem layering**

On the EC2 host, auth logs were removed from `/var/log`, which is a common anti-forensics move. But `findmnt` showed overlayfs with multiple layers mounted at the same path. The deletion affected only the top writable layer. Unmounting `/var/log` exposed the lower layer, where original auth logs were still intact.

Filtering those recovered logs for accepted SSH events under `postgresql-user` confirmed movement from the PostgreSQL host into `ssh-fetcher`. The attacker assumed a simple filesystem model and that assumption broke the case open.

**Finding 4: Persistence exposed the full operation**

On the PostgreSQL host, a cron job under `postgres` referenced a file named `pg_sched`, likely chosen to look routine. Base64 decoding revealed a bash script that enumerated IAM policies, searched for SSH keys, streamed harvested credentials to C2 over `ncat`, and then cleared shell history.

Credentials embedded in the script authenticated to a REST API on C2 at `34.118.239.100`. The stolen recipe file appeared there, which confirmed exfiltration and pointed directly to remediation with an authenticated DELETE request.

---

## Architectural lessons

**IAM permissions had drifted from intent.** `S3Reader` had `PutObject` even though the role name implied read-only use. `credsrotator` had Secrets Manager write access on `*` instead of specific secret ARNs. `lambdaWorker` lacked strict trust conditions for `UpdateFunctionCode`. Tighter scope and regular permission reviews would have reduced attacker options at each step.

**Cron persistence was not detected.** A scheduled task under `postgres` harvested credentials and maintained outbound C2 traffic. Host telemetry such as auditd or file integrity monitoring on cron directories, plus alerting on job creation under service accounts, would have made this visible much earlier.

**Outbound traffic controls were weak.** Two outbound channels were used: `logger.py` to port 8443 and `pg_sched` to C2 on port 4444. Neither appears to have triggered network-layer detection. Stricter egress policy and anomaly alerts on unknown destinations would reduce dwell time.

**Two controls made the investigation possible.** CloudTrail management events allowed reconstruction of IAM pivots. S3 data events, which are not enabled by default, made it possible to confirm exfiltration and quantify access. Overlayfs behavior on EC2 also helped by preserving auth logs in lower layers after deletion.

---

## Architectural gaps summary

| Gap | What happened | What should exist |
|---|---|---|
| Host-level monitoring | Cron persistence ran undetected on PostgreSQL host | EDR or auditd on all hosts; alerting on cron job creation under service accounts |
| Credential hygiene | Plaintext credentials hardcoded in a cron script | Secrets Manager for all credentials; no secrets on disk |
| Egress filtering | Unrestricted outbound from EC2 and PostgreSQL host | Security group egress rules; alerting on connections to unknown external IPs |
| S3 role permissions | S3Reader held PutObject; name and permissions were misaligned | IAM roles scoped strictly to declared function; regular permission audits |
| Lambda code integrity | No controls on UpdateFunctionCode callers | Code signing enforced; CloudWatch alert on unexpected UpdateFunctionCode |
| Lambda execution role | credsrotator had broad Secrets Manager write access | Permissions scoped to specific secret ARNs only |
| STS session policies | Assumed sessions inherited full role permissions | Session policies applied at assumption time to restrict scope |
| Detection coverage | No alerts fired end-to-end across the intrusion | GuardDuty with anomaly baselines; CloudTrail S3 data events enabled |

---

*Part of my [security portfolio](https://github.com/amahmud01), mapping hands-on work to real-world attack patterns.*
