---
layout: post
title: "AWS Cloud Incident Response — ExfilCola"
tags: [Incident Response]
excerpt: "A simulated AWS breach via the Wiz Cloud Hunting Games CTF — IAM role abuse, CloudTrail forensics, lateral movement across accounts, and recovering deleted auth logs with overlayfs."
assets_dir: aws-cloud-incident-response-exfilcola
---

{% assign asset_base = '/assets/' | append: page.assets_dir %}

> **Wiz Cloud Hunting Games CTF** · Cloud Forensics · MITRE ATT&CK · AWS IR  
> *Techniques: IAM role abuse · CloudTrail analysis · Lateral movement · Lambda code injection · Overlayfs log recovery*

---

## What happened

This write-up covers my investigation of the Wiz Cloud Hunting Games CTF — a hands-on cloud incident response challenge built around a simulated AWS breach. The scenario: a fictional startup called ExfilCola receives a data extortion email from a threat group calling themselves FizzShadows, claiming to have stolen their proprietary soda recipe. The task is to determine whether the claim is real, trace the full attack chain, and understand what failed architecturally.

What makes this CTF worth writing up isn't the fictional wrapper — it's that the attack techniques are entirely realistic. No zero-days, no gimmicks. Just IAM misconfiguration, role abuse, and an attacker moving quietly through a cloud environment using APIs that look completely normal to default monitoring.

The investigation ran backwards — from the ransom email to the exfiltration, then back through the identity chain to the entry point. What made this tractable was logging hygiene: S3 data events and CloudTrail were both enabled, leaving the identity trail intact even after the attacker attempted to cover their tracks on the EC2 host. The sharpest moment was the overlayfs recovery — auth logs had been deleted, but the underlying filesystem layer preserved them, one unmount command away from a dead end becoming a confirmed lateral movement path.

The sections below map the full attack chain to MITRE ATT&CK, surface the detection gaps, and draw out the architectural failures that made this breach possible.

---

## ATT&CK coverage

> [View full ATT&CK Navigator layer]({{ asset_base | append: '/attack-navigator/layer.json' | relative_url }})

![MITRE ATT&CK Heatmap]({{ asset_base | append: '/attack-navigator/heatmap.png' | relative_url }})

The attack covered a broad range of the ATT&CK cloud matrix — most phases from initial access through to exfiltration and defence evasion. What stood out was how much ground was covered using only legitimate AWS APIs. Every action the attacker took — assuming roles, listing buckets, updating Lambda code, writing to Secrets Manager — is indistinguishable from normal operations without behavioural baselines. That's the core challenge of cloud IR: the attacker's tools are your tools.

The defence evasion technique was particularly notable. Deleting auth logs from the EC2 host is a reasonable countermeasure against forensic investigation — but it assumes the filesystem is what it appears to be. Overlayfs preserved the underlying layer, and that assumption cost the attacker their anonymity.

| Technique ID | Name | Phase | Evidence |
|---|---|---|---|
| [T1078.004](https://attack.mitre.org/techniques/T1078/004/) | Valid Accounts: Cloud Accounts | Initial Access | Compromised IAM principal; initial access visible in CloudTrail `userIdentity` / session context |
| [T1526](https://attack.mitre.org/techniques/T1526/) | Cloud Service Discovery | Discovery | ListBuckets, ListObjects reconnaissance |
| [T1069.003](https://attack.mitre.org/techniques/T1069/003/) | Permission Groups Discovery: Cloud Groups | Discovery | IAM and permission-group style discovery via AWS APIs (CloudTrail) |
| [T1548](https://attack.mitre.org/techniques/T1548/) | Abuse Elevation Control Mechanism | Privilege Escalation | AssumeRole chain across IAM identities |
| [T1098.001](https://attack.mitre.org/techniques/T1098/001/) | Account Manipulation: Additional Cloud Credentials | Persistence | PutSecretValue — IAM keys stored in Secrets Manager |
| [T1530](https://attack.mitre.org/techniques/T1530/) | Data from Cloud Storage Object | Exfiltration | GetObject — bulk read of recipe bucket |
| [T1021.004](https://attack.mitre.org/techniques/T1021/004/) | Remote Services: SSH | Lateral Movement | SSH into ssh-fetcher host |
| [T1059](https://attack.mitre.org/techniques/T1059/) | Command and Scripting Interpreter | Execution | logger.py — outbound exfiltration via fetcher.exfilcola.io:8443 |
| [T1070.002](https://attack.mitre.org/techniques/T1070/002/) | Indicator Removal: Clear Linux Logs | Defence Evasion | Auth log deletion — recovered via overlayfs |

---

## Key findings

**Finding 1 — Identity columns are the wrong place to look after a role pivot**

The first pivot point in the investigation: searching CloudTrail by the compromised ARN returned a single result — a ListBuckets call with a blank `useridentity_username`. A direct IAM user always populates that field. A blank username with an AssumedRole identity type means the attacker was operating under temporary STS credentials — and had already moved on.

The break came from searching `requestparameters` rather than the identity columns. When AssumeRole is called, the target role is recorded in the parameters, not in the caller's identity. Identity columns tell you who made the call. Parameters tell you what they were becoming. Every hop in the lateral movement chain required this same technique — and every hop would have been a dead end without it. This is the most transferable lesson from the investigation: cloud identity pivots are invisible to analysts searching the wrong field.

**Finding 2 — Lambda weaponisation is harder to detect than new resource creation**

Once the attacker reached the target EC2 instance (challenge lab host, role `lambdaWorker`), they called `UpdateFunctionCode` against an existing Lambda named `credsrotator` — a legitimate credential rotation function with elevated Secrets Manager permissions. The injected code executed `PutSecretValue`, storing stolen IAM access keys in a Secrets Manager entry used for key material in the scenario.

This technique is more operationally sound than spinning up new infrastructure. The Lambda was already trusted, its permissions were pre-existing, and its execution pattern blended into normal operations. There was no new resource to detect, no unusual IAM entity — just a function that had always existed doing something slightly different. Without code signing or alerting on `UpdateFunctionCode` events from unexpected principals, this change was effectively invisible.

**Finding 3 — Deleted logs aren't always gone**

On the compromised EC2 host, auth logs had been deleted — a reasonable countermeasure. But the filesystem was running overlayfs, which maintains a lower read-only layer beneath the writable upper layer. Unmounting the upper layer exposed the preserved auth logs underneath, recovering the source address seen in auth data and confirming the SSH lateral movement path.

The lesson isn't just forensic technique — it's that attacker OPSEC assumptions about cloud infrastructure are often wrong. Deleting a file on what appears to be a standard Linux filesystem doesn't account for container runtimes, layered filesystems, or snapshot-based storage. Cloud environments preserve more than attackers expect.

---

## Architectural lessons

The breach was enabled by a cluster of compounding failures, none of which were individually catastrophic but together removed every natural containment point in the environment. The most significant was IAM design. Roles were over-trusted — policies allowed assumption from broad principals without source conditions, and session policies were never applied at assumption time. This meant that every role compromise was a full compromise: the attacker inherited complete permissions with no restriction on scope or duration. Combined with S3 permissions that granted read access at the bucket level rather than the object prefix level, a single compromised credential was sufficient to exfiltrate everything. The Lambda execution role compounded this — `credsrotator` had Secrets Manager write access scoped to `*` rather than the specific ARNs it actually needed to rotate. The principle of least privilege existed nowhere in this environment in practice.

The detection gap was equally significant. No alerts fired during the entire intrusion — not on the AssumeRole chain, not on the bulk S3 reads, not on the `UpdateFunctionCode` call, not on the outbound connection to `fetcher.exfilcola.io:8443`. GuardDuty was either absent or unconfigured for anomaly baselines. CloudTrail data events for S3 — which is where the exfiltration evidence lived — are not enabled by default and carry additional cost, which is a genuine architectural trade-off: the visibility that makes this investigation possible is the visibility that most organisations don't pay for until after an incident. The design fix isn't just technical — it's a risk decision about what logging costs versus what a breach costs. In an environment handling proprietary data, that calculation should be straightforward.

---

## Supporting artifacts

| Artifact | Description |
|---|---|
| [`attack-navigator/layer.json`]({{ asset_base | append: '/attack-navigator/layer.json' | relative_url }}) | ATT&CK Navigator layer — import at [attack.mitre.org](https://mitre-attack.github.io/attack-navigator/) |
| [`queries/cloudtrail.sql`]({{ asset_base | append: '/queries/cloudtrail.sql' | relative_url }}) | CloudTrail SQL queries used during investigation |
| [`queries/s3_events.sql`]({{ asset_base | append: '/queries/s3_events.sql' | relative_url }}) | S3 data event queries |
| [`evidence/`]({{ asset_base | append: '/evidence/' | relative_url }}) | Sanitised log snippets and screenshots |

---

*Part of my [security portfolio](https://github.com/am-netizen) — mapping hands-on work to real-world attack patterns.*
