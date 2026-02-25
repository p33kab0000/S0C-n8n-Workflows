# 🛡️ n8n URL & IP Reputation Checker — SOC Automation Workflow

> Automated multi-source threat intelligence lookup that cuts manual indicator triage from 45 minutes down to 90 seconds. Built for SOC analysts who are tired of copy-pasting URLs into browser tabs.

![n8n](https://img.shields.io/badge/Built%20with-n8n-orange?style=flat-square&logo=n8n)
![VirusTotal](https://img.shields.io/badge/API-VirusTotal-blue?style=flat-square)
![URLScan](https://img.shields.io/badge/API-URLScan.io-purple?style=flat-square)
![AbuseIPDB](https://img.shields.io/badge/API-AbuseIPDB-red?style=flat-square)
![Slack](https://img.shields.io/badge/Alerts-Slack-green?style=flat-square)
![Google Sheets](https://img.shields.io/badge/Logging-Google%20Sheets-yellow?style=flat-square)
![License: MIT](https://img.shields.io/badge/License-MIT-lightgrey?style=flat-square)


## 📋 Table of Contents

- [What This Does](#-what-this-does)
- [Workflow Architecture](#-workflow-architecture)
- [The Scoring Algorithm](#-the-scoring-algorithm)
- [Prerequisites](#-prerequisites)
- [Setup Guide](#-setup-guide)
- [How to Use It](#-how-to-use-it)
- [Sample Slack Alert](#-sample-slack-alert)
- [Google Sheets Schema](#-google-sheets-schema)
- [Customization](#-customization)
- [Contributing](#-Contributing)
- [Known Limitations & Roadmap](#-known-limitations--roadmap)
- [License](#-license)



## 🔍 What This Does

This n8n workflow accepts a **URL or IP address** via a webhook POST request, runs it through multiple threat intelligence sources in parallel, calculates a hybrid risk score using a custom algorithm, logs the result, and fires a Slack alert if the indicator is high risk.

**One POST request in. Full triage summary out. No tabs opened.**

### What gets checked

| Indicator Type | Sources Queried |
|---|---|
| 🌐 URL / Domain | VirusTotal + URLScan.io (parallel) |
| 📡 IP Address | AbuseIPDB |

### What gets produced

- ✅ Hybrid risk score (0-100) with scoring method transparency
- ✅ Binary verdict (HIGH / LOW) for immediate analyst action
- ✅ Breakdown of which sources flagged the indicator and why
- ✅ Direct links to full reports and screenshots
- ✅ Automatic logging to Google Sheets (separate tabs for high risk and safe)
- ✅ Slack alert for anything scoring 50 or above


## 🏗️ Workflow Architecture

```
🔔 Webhook Trigger (POST /url-scan)
        |
        v
🧠 Input Classifier — Code node detects URL vs IP via regex
        |
   _____|_____
  |           |
  v           v
🌐 URL Path  📡 IP Path

🌐 URL Path:
  Parallel execution:
    🔎 URLScan.io  →  Wait 30s  →  Get Results
    🦠 VirusTotal  →  Wait 30s  →  Get Report
  🔀 Merge Results
  🧮 Calculate Hybrid Risk Score (Code Node)
  ⚖️  IF Risk Score >= 50:
    🚨 YES  →  Log to High Risk Sheet  →  Slack Alert  →  Respond
    ✅ NO   →  Log to Safe Sheet  →  Respond

📡 IP Path:
  🛡️ AbuseIPDB: GET /check?ipAddress=...
  ⚖️  IF Abuse Score > 50:
    🚨 YES  →  Log to High Risk Sheet  →  Slack Alert  →  Respond
    ✅ NO   →  Log to Safe Sheet  →  Respond
```

### Node Map

| Node | Type | Purpose |
|---|---|---|
| `Webhook Trigger1` | Webhook | Entry point — receives POST requests |
| `Is it URL or IP` | Code | Regex classifier — routes input to correct path |
| `URL/IP` | IF | Branching node — sends to URL or IP pipeline |
| `Perform a scan` | URLScan.io | Submits URL for scanning |
| `Wait for URLScan` | Wait | 30s pause for scan to complete |
| `Get a scan` | URLScan.io | Retrieves scan results (5 retries, 5s intervals) |
| `VirusTotal URL Check` | HTTP Request | Submits URL to VirusTotal |
| `Wait for Virustotal Scan` | Wait | 30s pause for VT analysis |
| `VirusTotal: Get report` | HTTP Request | Retrieves VT analysis report |
| `Merge` | Merge | Combines URLScan and VT results |
| `Calculate Risk Score` | Code | Core scoring algorithm (see below) |
| `Risk Score > 50?` | IF | Routes to high risk or safe path |
| `Log to High Risk Sheet` | Google Sheets | Appends full enrichment data |
| `Log to Safe Sheet` | Google Sheets | Appends basic data for safe indicators |
| `Send Slack Alert2` | Slack | Fires formatted alert for high risk URLs |
| `Check IP - AbuseIPDB1` | HTTP Request | Queries AbuseIPDB for IP reputation |
| `IF - Abuse Score > 50?` | IF | Routes IP to high risk or safe path |
| `Append row in sheet3` | Google Sheets | Logs high risk IPs |
| `Append row in sheet2` | Google Sheets | Logs safe IPs |
| `Send Slack Alert` | Slack | Fires alert for high risk IPs |
| `Respond to Webhoo1` | Respond to Webhook | Returns URL scan summary to caller |
| `Respond to Webhook2` | Respond to Webhook | Returns IP scan result to caller |


## 🧮 The Scoring Algorithm

The scoring logic lives in the `Calculate Risk Score` code node. This is the most important part of the workflow — read this before you customize anything.

### Why not just average the scores?

Averaging creates a dangerous blind spot. If URLScan scores a URL at **85** (high confidence malicious) and VirusTotal scores it at **15** (mostly clean), the average is **50** — borderline, easy to dismiss. But a score of 85 from URLScan means something real. Averaging would hide it.

### How scoring actually works

**Step 1: Source availability check**

| Scenario | Logic Applied |
|---|---|
| Both sources unavailable | Score = 0, flagged for manual review |
| VT only (URLScan down) | `score = (vt_score * 0.85) + 10` — single-source confidence penalty |
| URLScan only (VT down) | `score = (urlscan_score * 0.85) + 10` — same penalty applied symmetrically |
| Both sources available | Proceed to Step 2 |

**Step 2: Corroborated scoring (both sources available)**

| Condition | Method | Reason |
|---|---|---|
| Either source scores >= 70 | **MAX** of the two scores | High-confidence detection — one strong signal is enough |
| Both sources score < 70 | **AVERAGE** of the two scores | Neither source is confident — consensus is appropriate |

**Step 3: High-reputation vendor override**

Even after the hybrid score is calculated, the workflow checks a curated list of high-signal vendors in the VirusTotal results:

```
Kaspersky, BitDefender, ESET, Sophos, Emsisoft,
G-Data, Webroot, Netcraft, Forcepoint ThreatSeeker, Google Safebrowsing
```

| Flagged by high-rep vendors | Score floor applied |
|---|---|
| 2 or more vendors | Minimum score raised to **50** |
| 4 or more vendors | Minimum score raised to **60** |

The reasoning: commodity AV engines produce high false positive rates. A detection from Netcraft and Google Safe Browsing together carries more weight than 15 detections from engines no one has heard of. This institutional knowledge is now baked into the automation.

**Final verdict:** Score >= 50 = 🔴 HIGH. Score < 50 = 🟢 LOW.


## 📦 Prerequisites

Before you import the JSON, make sure you have:

- **n8n** (self-hosted or cloud) — [Get it here](https://n8n.io)
- **VirusTotal API key** — Free tier at [virustotal.com](https://www.virustotal.com/gui/join-us)
- **URLScan.io API key** — Free tier at [urlscan.io](https://urlscan.io/user/signup)
- **AbuseIPDB API key** — Free tier at [abuseipdb.com](https://www.abuseipdb.com/register)
- **Slack workspace** with a channel for alerts and a Slack app/bot token
- **Google Sheets** with two sheets set up (one for high risk, one for safe indicators)

All APIs have generous free tiers. You can run this at reasonable SOC volume without paying anything.


## ⚙️ Setup Guide

### Step 1: Import the workflow

1. Open your n8n instance
2. Go to **Workflows** > **Import from file**
3. Upload `workflow.json` from this repo
4. The workflow will import with placeholder values where credentials are needed

### Step 2: Configure credentials

In n8n, go to **Settings** > **Credentials** and add:

**VirusTotal**
- Type: `VirusTotal API`
- API Key: your VirusTotal key

**URLScan.io**
- Type: `URLScan.io API`
- API Key: your URLScan.io key

**Google Sheets**
- Type: `Google Sheets OAuth2` (recommended) or Service Account
- Follow n8n's Google Sheets credential setup guide

**Slack**
- Type: `Slack API`
- Bot Token: your Slack bot OAuth token (needs `chat:write` scope)

### Step 3: Update the workflow nodes

After importing, update these placeholders in the workflow:

| Node | Field to Update | What to Put |
|---|---|---|
| `Log to High Risk Sheet` | `documentId` | Your Google Sheet ID |
| `Log to High Risk Sheet` | `sheetName` | Your high risk tab name |
| `Log to Safe Sheet` | `documentId` | Your Google Sheet ID |
| `Log to Safe Sheet` | `sheetName` | Your safe results tab name |
| `Append row in sheet3` | `documentId` | Your Google Sheet ID |
| `Append row in sheet3` | `sheetName` | Your IP high risk tab name |
| `Append row in sheet2` | `documentId` | Your Google Sheet ID |
| `Append row in sheet2` | `sheetName` | Your IP safe tab name |
| `Send Slack Alert` | `channelId` | Your Slack channel ID |
| `Send Slack Alert2` | `channelId` | Your Slack channel ID |
| `Check IP - AbuseIPDB1` | `Key` header | Your AbuseIPDB API key |

> **Finding your Google Sheet ID:** It's the long string in your Sheet URL between `/d/` and `/edit`. Example: `https://docs.google.com/spreadsheets/d/THIS_IS_YOUR_ID/edit`

> **Finding your Slack channel ID:** Right-click your channel in Slack > View channel details > Copy channel ID at the bottom.

### Step 4: Activate the workflow

Toggle the workflow to **Active** in n8n. Your webhook endpoint is now live at:

```
https://your-n8n-instance.com/webhook/url-scan
```


## 🚀 How to Use It

Send a POST request to your webhook URL with a JSON body containing either a URL or an IP address:

**Check a URL:**
```bash
curl -X POST https://perversive-emelina-promoderation.ngrok-free.dev/webhook-test/url-scan -H "Content-Type: application/json" -d '{"query": "1.177.63.24"}'
```

**Check an IP:**
```bash
curl -X POST https://perversive-emelina-promoderation.ngrok-free.dev/webhook-test/url-scan -H "Content-Type: application/json" -d '{"query": "1.177.63.24"}'
```

**PowerShell (Windows SOC environments):**
```powershell
Invoke-RestMethod -Uri "https://your-n8n-instance.com/webhook/url-scan" `
  -Method POST `
  -ContentType "application/json" `
  -Body '{"url": "http://suspicious-domain.com"}'
```

**Python:**
```python
import requests

response = requests.post(
    "https://your-n8n-instance.com/webhook/url-scan",
    json={"url": "http://suspicious-domain.com"}
)
print(response.text)
```

The workflow accepts `url`, `query`, or `ip` as the key — all three work.

## 📣 Sample Slack Alert

When a high risk URL is detected, this hits your Slack channel:

```
🚨 HIGH RISK URL DETECTED 🚨

🔗 URL: http://110.37.0.37:35668/bin.sh
📊 Risk Score: 60/100

🛡️ Threat Intelligence
Source          | Result
VirusTotal      | 21 malicious, 2 suspicious
URLScan Verdict | Malicious
URLScan Score   | 7/100

📄 Reports
🦠 VirusTotal Report
🔍 URLScan Report
📸 Screenshot

🔎 Scan Details
🆔 Scan ID: 1771614286778
🕐 Timestamp: 2026-02-20T19:04:46.778Z
```

For high risk IPs, the alert includes: IP address, abuse confidence score, ISP, usage type, total community reports, and last reported date.

## 📊 Google Sheets Schema

### URL Sheets (High Risk + Safe)

| Column | Description |
|---|---|
| Timestamp | ISO timestamp of the scan |
| URL | The indicator that was scanned |
| Risk Score | 0-100 composite score |
| VT Detections | Number of malicious VT engine detections |
| VT Detection Ratio | Format: `malicious/total engines` |
| URLScan | URLScan verdict (Malicious / Clean / Unavailable) |
| Verdict | HIGH or LOW |
| Screenshot URL | Direct link to URLScan page screenshot |
| Last Analysis Date | Timestamp repeated for sorting |
| URLScan Report | Full URLScan report URL |
| VT Result Report | VirusTotal GUI link for the scanned URL |

> The Safe sheet logs a subset of these columns — Risk Score, VT Detections, and Detection Ratio only. Enough to confirm it was checked without bloating the log.

### IP Sheets (High Risk + Safe)

| Column | Description |
|---|---|
| Risk | HIGH or LOW |
| ipAddress | The IP that was checked |
| abuseScore | AbuseIPDB confidence score (0-100) |
| usageType | ISP classification (e.g., Data Center, Residential) |
| country | Two-letter country code |
| isp | Internet Service Provider name |
| Domain and Hostname | Associated domain and hostnames |
| totalReports | Number of community abuse reports |
| isWhitelisted | Whether AbuseIPDB has whitelisted this IP |


## 🔧 Customization

### Adjust the risk threshold

The `Risk Score > 50?` IF node uses `>= 50` as the HIGH risk cutoff. To make it more or less sensitive:

- **More aggressive (lower threshold):** Change `50` to `40` — catches more potential threats, more false positives
- **More conservative (higher threshold):** Change `50` to `65` — fewer alerts, higher confidence per alert

### Adjust wait times

Both `Wait for URLScan` and `Wait for Virustotal Scan` are set to **30 seconds**. If you're seeing incomplete results, increase to 45-60 seconds. If your volume is high and you want faster responses, you can try 20 seconds — but results may occasionally be incomplete for slower scans.

### Modify the high-rep vendor list

In the `Calculate Risk Score` code node, find the `reliableVendors` array and add or remove vendors based on your threat intelligence preferences:

```javascript
const reliableVendors = [
    'Kaspersky', 'BitDefender', 'ESET', 'Sophos',
    'Emsisoft', 'G-Data', 'Webroot', 'Netcraft',
    'Forcepoint ThreatSeeker', 'Google Safebrowsing'
];
```

### Change the score floor values

In the `applyHighRepOverride` function, the floors are currently `50` (2+ vendors) and `60` (4+ vendors). Adjust to match your organization's risk tolerance.

## 🚧 Known Limitations & Roadmap

### Current limitations

**IP path is single-source.** The IP pipeline currently relies only on AbuseIPDB. One source means one blind spot. An IP can have a low AbuseIPDB score while simultaneously being flagged as a known botnet node on AlienVault OTX and classified as a mass scanner on GreyNoise.

**No retry logic on VT submission.** The URLScan Get node has retry logic (5 retries, 5s intervals). The VirusTotal path does not. If VT returns an incomplete result, the workflow proceeds without retrying.

**No deduplication.** The same indicator submitted twice will be logged twice. There is no check against existing Sheet entries.

### Planned improvements

- 🔄 **Multi-source IP intelligence** — Adding GreyNoise, AlienVault OTX, IPQualityScore, Shodan, IPinfo.io, and ThreatFox to the IP pipeline with a hybrid scoring algorithm matching the URL path
- 🔁 **VT retry logic** — Mirror the URLScan retry approach on the VirusTotal Get Report node
- 🗃️ **Deduplication check** — Query existing Sheet entries before running a full scan
- 📊 **Risk factor transparency in Slack** — Surface the `risk_factors` field (which sources flagged it and why) directly in the Slack alert
- 🔗 **VT report link in URL Slack alert** — The IP alert includes a VT GUI link; the URL alert does not yet


## 🤝 Contributing

Found a bug? Have a better scoring approach? Added a new threat intel source?

1. Fork this repo
2. Create a feature branch (`git checkout -b feature/add-greynoise`)
3. Make your changes to the workflow JSON
4. Test against clean, suspicious, and known malicious indicators
5. Submit a PR with a description of what changed and why

Tag me on LinkedIn when you fork it — I'll reshare the best improvements with the community.

## 📄 License

MIT License — use it, fork it, deploy it, improve it. Attribution appreciated.

## 🙋 Questions?

If you get stuck on setup, open an issue with:
- Your n8n version
- Which node is failing
- The error message from the n8n execution log

The execution log in n8n (click any failed run) is your best friend for debugging. Check it before opening an issue — it usually tells you exactly what went wrong.

---

*Built by a SOC analyst who got tired of opening six browser tabs for every alert. If this saves your team time, consider giving it a ⭐*