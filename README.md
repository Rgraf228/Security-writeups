# Security Write-ups

Documented investigations from my home lab — a self-hosted environment 
running Proxmox, Ubuntu, and Wazuh (SIEM) used to practice real-world 
vulnerability detection, investigation, and remediation.

Each write-up follows a consistent format: Detection → Research → 
Investigation → Remediation → Verification → Lessons Learned — modeled 
on how a SOC analyst would document and close out a finding.

## Write-ups

- [CVE-2026-27606 — Rollup Path Traversal RCE](./CVE-2026-27606-rollup-investigation.md)
- [CVE-2015-7985 — Valve Steam Weak Install Folder Permissions](./CVE-2015-7985-steam-investigation.md)
- [CVE-2019-14743 — Valve Steam Client Registry Permission (Disputed)](./CVE-2019-14743-steam-investigation.md)
- [MITRE ATT&CK T1078 — Windows Logon Success Alert Triage](./mitre-t1078-windows-logon-investigation.md)
- [CVE-2024-52006 — Git Credential Helper Carriage Return Confusion](./CVE-2024-52006-git-investigation.md)
- [CVE-2024-52005 — Git Sideband Channel ANSI Escape Sequence Injection](./CVE-2024-52005-git-investigation.md)
- [CVE-2015-4016 — Valve Steam Remote Denial of Service](./CVE-2015-4016-steam-investigation.md)
- [CVE-2026-53571 — Vite server.fs.deny Bypass via Windows Alternate Paths](./CVE-2026-53571-vite-server-fs-deny-bypass.md)
