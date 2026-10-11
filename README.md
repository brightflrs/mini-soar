
<div align="center">

# 🛡️ Mini SOAR

**Enrich suspicious IPs, decide with a playbook, and block with human approval. Pure Python.**

![Python](https://img.shields.io/badge/Python-3.8%2B-3776AB?logo=python&logoColor=white)
![GUI](https://img.shields.io/badge/GUI-Tkinter-informational)
![Intel](https://img.shields.io/badge/Threat%20Intel-AbuseIPDB-orange)
![Platform](https://img.shields.io/badge/Firewall-Windows%20netsh%20%7C%20ufw-blue)
![Dependencies](https://img.shields.io/badge/dependencies-none-brightgreen)

</div>

---

> **An alert is not a decision.** Detection tells you an IP is attacking. Someone still has to look it up, judge the risk, block it, and write down why. This tool automates that loop, with guardrails so it can never lock you out of your own network.

A **SOAR** (Security Orchestration, Automation and Response) tool takes an alert and runs the response steps for you. This is a small, from-scratch version built to learn how it works. It is a natural follow-up to my [SSH SOC Detector](https://github.com/<your-username>/ssh-soc-detector): the detector finds the attacker, Mini SOAR responds.

<!-- Add screenshots: save them in /docs, then uncomment -->
<!-- ![Main window](docs/screenshot-main.png) -->
<!-- ![Approval popup](docs/screenshot-popup.png) -->

## 📑 Table of Contents

- [How it works](#-how-it-works)
- [The playbook](#-the-playbook)
- [Safety guardrails](#-safety-guardrails)
- [Getting started](#-getting-started)
- [Getting an AbuseIPDB key](#-getting-an-abuseipdb-key)
- [Going live (real firewall blocks)](#-going-live-real-firewall-blocks)
- [Audit log](#-audit-log)
- [Configuration](#%EF%B8%8F-configuration)
- [Roadmap](#%EF%B8%8F-roadmap)
- [Disclaimer](#%EF%B8%8F-disclaimer)

## ⚙️ How it works

```
 IP in  ──►  Enrich  ──►  Playbook  ──►  Approve?  ──►  Act  ──►  Audit log
 (typed or   AbuseIPDB    score-based    popup window    firewall   every decision
  from file) score,       decision       Block/Ignore    rule       recorded
              reports,
              country, ISP
```

1. **Input:** type an IP, or load a text/log file (it extracts every IPv4 inside)
2. **Enrich:** looks up the abuse confidence score, report count, country, and ISP on AbuseIPDB
3. **Decide:** the playbook picks an action from the risk score
4. **Approve:** for gray-zone scores, a popup window asks you to **Block** or **Ignore**
5. **Act:** creates a firewall rule (Windows `netsh`, or `ufw` on Linux)
6. **Record:** writes every decision to an audit log

## 🧠 The playbook

| Abuse score | Decision | What happens |
|-------------|----------|--------------|
| **90 – 100** | 🔴 AUTO-BLOCK | Blocked immediately (public IPs only) |
| **30 – 89** | 🟡 ASK | Approval popup: Block or Ignore |
| **0 – 29** | 🟢 LOG | Logged only, no action |

Thresholds are constants at the top of `mini_soar.py`.

## 🔒 Safety guardrails

Automation that can block things must not lock you out of your own network. So:

- **Dry-run is ON by default.** The firewall command is shown, never executed, until you turn it off
- **Allowlist.** Router, DNS, and your own machine can never be blocked. Loopback, link-local, multicast, and reserved ranges are always protected
- **Private/lab IPs are never auto-blocked.** They always need a human approval
- **Auto-expiry.** Blocks expire after 24 hours (checked while the app is running), with one-click **Unblock**
- **Admin check.** Live mode refuses to run without Administrator/root instead of failing silently
- **Closing the popup = Ignore.** No accidental blocks
- **Audit trail.** Every enrich, approve, ignore, block, and unblock is logged

## 🚀 Getting started

**Requirements:** Python 3.8+. No packages to install (standard library only; Tkinter ships with Python on Windows/macOS).

```powershell
git clone https://github.com/<your-username>/mini-soar.git
cd mini-soar
python mini_soar.py
```

**Try it without an API key.** Without a key, scores are simulated and labeled `DEMO`. Safe to test in dry-run:

| IP | Demo result |
|----|-------------|
| `45.45.3.2` | 🔴 AUTO-BLOCK (dry-run) |
| `11.22.33.4` | 🟡 Approval popup |
| `203.0.113.5` | 🟢 Logged only |

## 🔑 Getting an AbuseIPDB key

1. Create a free account at [abuseipdb.com](https://www.abuseipdb.com)
2. Go to **Account → API → Create Key**
3. Set it as an environment variable (never commit it):

```powershell
$env:ABUSEIPDB_KEY = "your-key-here"; python mini_soar.py
```

Free accounts have a daily limit on checks. Private IPs are never sent to AbuseIPDB (they use simulated scores).

## 🔥 Going live (real firewall blocks)

1. **Edit `ALLOWLIST`** in `mini_soar.py`: add your gateway, DNS, and your own IP. Find them with `ipconfig`
2. Open PowerShell **as Administrator**
3. Run the app, then **uncheck Dry-run**. The header badge turns red (`● LIVE FIREWALL`)

Check or remove the rules it created:

```powershell
Get-NetFirewallRule -DisplayName "SOAR-Block*" | Select-Object DisplayName, Action, Direction
Remove-NetFirewallRule -DisplayName "SOAR-Block*"
```

> ⚠️ Only test live blocks on IPs you don't need. Blocks are **inbound** rules.

## 📝 Audit log

Every action is appended to `soar_audit.jsonl` (one JSON object per line):

```json
{"time": "2026-10-11T19:14:51", "event": "enriched", "ip": "146.70.211.178", "decision": "ask", "score": 45}
{"time": "2026-10-11T19:15:03", "event": "approved", "ip": "146.70.211.178", "by": "analyst"}
{"time": "2026-10-11T19:15:03", "event": "blocked", "ip": "146.70.211.178", "by": "analyst", "dry": false}
```

Active blocks are saved in `soar_blocks.json`. Both files are git-ignored.

## 🛠️ Configuration

Top of `mini_soar.py`:

```python
AUTO_BLOCK_SCORE = 90     # score >= this -> auto-block (public IPs only)
ASK_SCORE = 30            # score >= this -> popup asking you to approve/ignore
BLOCK_HOURS = 24          # blocks expire after this long
ALLOWLIST = {"127.0.0.1", "192.168.1.1"}   # NEVER block these
```
## 🗺️ Roadmap

- [ ] Auto-feed IPs from the SSH Detector (detection → enrichment → response)
- [ ] Extra enrichment: VirusTotal and GeoIP
- [ ] Run as a background service so block expiry works even when the window is closed
- [ ] CSV/HTML incident report export
- [ ] Unit tests for the playbook and allowlist logic

## ⚠️ Disclaimer

For learning and defensive use on systems you own or are authorized to manage. Blocking IPs can disrupt legitimate traffic. Test in dry-run first and review the allowlist before going live.

## 👤 Author

**Bright H. Flores**, IT student & aspiring SOC analyst
🔗 [LinkedIn](https://linkedin.com/in/bright-flores-113053270)
