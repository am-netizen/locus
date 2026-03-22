---
layout: post
title: "AWS Cloud Incident Response — ExfilCola"
tags: [Incident Response]
excerpt: "A simulated AWS breach via the Wiz Cloud Hunting Games CTF — IAM abuse, CloudTrail, lateral movement, and recovering deleted auth logs with overlayfs."
assets_dir: aws-cloud-incident-response-exfilcola
---

{% assign asset_base = '/assets/' | append: page.assets_dir %}

> **Wiz Cloud Hunting Games CTF** · Cloud Forensics · MITRE ATT&CK · AWS IR  
> *Techniques: IAM role abuse · CloudTrail analysis · Lateral movement · Overlayfs recovery*

---

## What happened

This write-up covers my investigation of the Wiz Cloud Hunting Games CTF — a hands-on cloud incident response challenge built around a simulated AWS breach. The scenario: a fictional startup called ExfilCola receives a data extortion email from a threat group claiming to have stolen their proprietary soda recipe. Your job is to determine whether the claim is real, trace the attack chain, and remediate. What makes this CTF worth writing up isn't the fictional wrapper — it's that the attack techniques are entirely realistic. No zero-days, no CTF gimmicks — just IAM misconfiguration, role abuse, and an attacker moving quietly through a cloud environment using APIs that look completely normal to default monitoring.

What made this tractable as an investigation was logging hygiene — S3 data events and CloudTrail were both enabled, leaving the identity trail intact even after the attacker attempted to cover their tracks on the EC2 host. The sharpest moment was the overlayfs recovery: auth logs had been deleted, but the underlying filesystem layer preserved them — one unmount command away from a dead end becoming a confirmed lateral movement path. The sections below map the full attack chain to MITRE ATT&CK, surface the detection gaps, and draw out the architectural failures that made this breach possible in the first place.

---

## ATT&CK coverage

> 📎 [View full ATT&CK Navigator layer]({{ asset_base | append: '/attack-navigator/layer.json' | relative_url }})

![MITRE ATT&CK Heatmap]({{ asset_base | append: '/attack-navigator/heatmap.png' | relative_url }})

<!-- Add 1–2 paragraphs here: which techniques appeared, why they matter, what surprised you -->

| Technique ID | Name | Phase | Evidence |
|---|---|---|---|
| [T1078.004](https://attack.mitre.org/techniques/T1078/004/) | Valid Accounts: Cloud Accounts | Initial Access | `[Add: IAM user / event]` |
| [T1069.003](https://attack.mitre.org/techniques/T1069/003/) | Permission Groups Discovery: Cloud Groups | Discovery | `[Add: API calls observed]` |
| [T1098.001](https://attack.mitre.org/techniques/T1098/001/) | Account Manipulation: Additional Cloud Credentials | Persistence | `[Add: evidence]` |
| [T1530](https://attack.mitre.org/techniques/T1530/) | Data from Cloud Storage Object | Exfiltration | `GetObject` on recipe bucket |
| [T1021.004](https://attack.mitre.org/techniques/T1021/004/) | Remote Services: SSH | Lateral Movement | SSH into `ssh-fetcher` host |
| [T1070.002](https://attack.mitre.org/techniques/T1070/002/) | Indicator Removal: Clear Linux Logs | Defense Evasion | Auth log deletion — recovered via overlayfs |

---

## Key findings

<!-- Add 2–3 findings here. Each should be: what you observed + the log evidence that confirmed it + why it matters -->

**Finding 1 — [Title]**

`[Add: your most impactful finding — e.g. IAM role assumption chain, the specific CloudTrail query that cracked it, and what it revealed]`

**Finding 2 — [Title]**

`[Add: second finding — e.g. the overlayfs recovery, what command surfaced the attacker's source IP, and what this tells you about attacker OPSEC assumptions in cloud environments]`

**Finding 3 — [Title, optional]**

`[Add: if applicable — e.g. something about the Lambda function, exposed credentials, or a detection gap you noticed]`

---

## Architectural lessons

<!-- Add 2 paragraphs: first diagnoses the structural failures, second proposes specific design fixes -->

`[Add: paragraph 1 — what architectural failures enabled this attack. Think: over-permissive IAM roles, absence of least-privilege design, missing S3 server access logging, no GuardDuty anomaly baselines. Be specific to what you observed.]`

`[Add: paragraph 2 — specific design changes that would have prevented or contained this. Think: IAM role trust policy scoping, CloudTrail data event costs vs detection value, VPC segmentation, blast radius limits. Frame at least one recommendation as a trade-off to show real-world architectural thinking.]`

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
