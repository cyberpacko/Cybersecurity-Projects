# Project 1 — Detection Lab: Active Directory + SIEM (Purple Team)

**Target roles:** SOC Analyst, Blue Team, Detection Engineer, IAM Support
**Frameworks:** MITRE ATT&CK, Sigma detection rules, NIST SP 800-92 (log management)
**Time:** ~1 weekend to build, ~half a day to attack + detect properly
**Cost:** $0 (all tools free/eval-licensed)

## Why this project

Active Directory is still used by about 90% of businesses ([PMC](https://pmc.ncbi.nlm.nih.gov/articles/PMC9759930/)), and SIEM expertise appears in 78% of SOC analyst job listings, making it the most requested technical skill ([TryHackMe](https://tryhackme.com/resources/blog/what-qualifications-do-employers-actually-recognise-for-soc-analyst-roles)).

This project combines AD administration, log analysis, and attacker technique. Entry-level SOC postings tend to ask for these together: SIEM fluency, Windows fundamentals, and knowledge of attack techniques mapped to MITRE ATT&CK ([Dropzone AI](https://www.dropzone.ai/resources/learning-guides/soc-analyst-career-path-salary-guide-2026-ai-powered-edition)).

I run the attack, then find it in my own logs, so I work as both red team and blue team in one environment.

## Architecture

```
[Windows Server 2025 Eval] --Domain Controller (AD DS)
        |
        | Windows Event Forwarding / Splunk Universal Forwarder
        v
[Splunk Free / Microsoft Sentinel Trial] --SIEM, ingest Security event logs
        |
[Attacker VM] --Kali or Windows client--> password spray against AD
```

## Tools

| Tool | Purpose | License |
|---|---|---|
| VirtualBox / Hyper-V | Hypervisor for the lab | Free |
| Windows Server 2025 Evaluation | Domain Controller | Free 180-day eval |
| Splunk Enterprise (Free tier, 500MB/day) **or** Microsoft Sentinel (free trial + Azure free credits) | SIEM | Free |
| Kali Linux or CrackMapExec/Hydra | Attack simulation | Free, open-source |
| Sysmon + Windows Event Forwarding | Enhanced telemetry | Free (Microsoft Sysinternals) |

## Build steps

1. **Stand up the Domain Controller.** Install Windows Server, promote to a Domain Controller (`Install-ADDSForest`), create a domain (e.g., `lab.local`), add 3–5 user accounts including one deliberately weak-password account.
2. **Install Sysmon** on the DC and any client VMs using a hardened config (e.g., SwiftOnSecurity's `sysmonconfig.xml`) for high-fidelity process/logon telemetry beyond default Windows auditing.
3. **Forward logs to the SIEM.** Install a Splunk Universal Forwarder on the DC pointed at your Splunk indexer, or connect Sentinel via the Azure Monitor Agent / Windows Security Events connector. Confirm `Security`, `System`, and Sysmon event channels are landing in the SIEM (`index=main sourcetype=WinEventLog:Security`).
4. **Simulate the attack.** Run a password spray from the attacker VM against multiple domain accounts using a small, low-and-slow script (see [`scripts/password_spray.py`](scripts/password_spray.py)) — this triggers Event ID 4625 (failed logon) clustering.
5. **Build the detection.** Write a Sigma rule (see [`detection-rules/password_spray_4625.yml`](detection-rules/password_spray_4625.yml)) that fires when a single source IP generates failed logons (4625) against 5+ distinct accounts within 10 minutes — the classic password-spray signature, distinct from brute force (many attempts, one account).
6. **Validate detection fires** in the SIEM against your own attack traffic, capture a screenshot of the alert, and note the MITRE ATT&CK mapping.

## MITRE ATT&CK mapping

| Technique | ID | Detection logic |
|---|---|---|
| Password Spraying | T1110.003 | ≥5 distinct accounts, ≥1 failed logon each, single source, 10-min window |
| Valid Accounts | T1078 | Successful logon (4624) immediately following a spray pattern from same source |



## Sources

- [Now's the time to rethink Active Directory security, PMC](https://pmc.ncbi.nlm.nih.gov/articles/PMC9759930/)
- [What qualifications do employers recognise for SOC roles, TryHackMe](https://tryhackme.com/resources/blog/what-qualifications-do-employers-actually-recognise-for-soc-analyst-roles)
- [SOC analyst career path and salary guide, Dropzone AI](https://www.dropzone.ai/resources/learning-guides/soc-analyst-career-path-salary-guide-2026-ai-powered-edition)

