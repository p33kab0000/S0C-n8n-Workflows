## **📝 Post-Import Setup Notes**

Include these instructions when sharing the workflow (for your article/GitHub):

---

### **⚙️ Setup Instructions**

After importing this workflow, you'll need to configure a few things:

### **1. AbuseIPDB API Key**

- Go to [abuseipdb.com](https://www.abuseipdb.com/) → Create free account
- Navigate to **API** → **Create Key**
- In n8n, open the **"Check IP - AbuseIPDB"** node
- Replace `YOUR_ABUSEIPDB_API_KEY` with your actual key

> 💡 Pro Tip: Use n8n's Credentials Manager instead of hardcoding. Go to Credentials → Add Credential → Header Auth
> 

---

### **2. Slack Integration**

- Create a Slack channel called `#suspicious_ip_alert`
- In n8n, go to **Credentials → Add Credential → Slack API**
- Follow the OAuth setup to connect your workspace
- Open the **"Send Slack Alert"** node and select your channel

---

### **3. Data Table Setup**

- In n8n, go to **Data Tables** (left sidebar)
- Create a new table called **"IP Reputation Log"**
- Add these columns:

| Column Name | Type |
| --- | --- |
| `Risk` | String |
| `ipAddress` | String |
| `abuseScore` | Number |
| `country` | String |
| `isp` | String |
| `usageType` | String |
| `totalReports` | Number |
| `isWhitelisted` | String |
- Update both **Data Table** nodes with your new table ID

---

### **4. Activate & Test**

1. Toggle the workflow **Active** (top-right)
2. Test with PowerShell:

powershell

`Invoke-RestMethod -Uri "https://YOUR-N8N-URL/webhook/ip-check" -Method POST -ContentType "application/json" -Body '{"ip": "185.220.101.34"}'`

1. Or with Bash/cURL:

bash

`curl -X POST https://YOUR-N8N-URL/webhook/ip-check \
  -H "Content-Type: application/json" \
  -d '{"ip": "185.220.101.34"}'
```


### **🧪 Test IPs**

| IP Address | Expected Result | Why |
|------------|-----------------|-----|
| `185.220.101.34` | 🔴 High Risk (~100 score) | Known Tor exit node |
| `8.8.8.8` | 🟢 Low Risk (0 score) | Google Public DNS |
| `1.1.1.1` | 🟢 Low Risk (0 score) | Cloudflare DNS |


### **🔧 Customization Ideas**

| Modification | How |
|--------------|-----|
| Change risk threshold | Edit the **IF** node → Change `50` to your preferred value |
| Add more threat intel | Add VirusTotal, GreyNoise nodes after AbuseIPDB |
| Different alerting | Replace Slack with Microsoft Teams, Discord, or Email |
| Auto-block IPs | Add a firewall API node after high-risk detection |


### **📊 What This Workflow Does**
```
📥 INPUT:  POST request with {"ip": "x.x.x.x"}
     ↓
🔍 LOOKUP: Query AbuseIPDB for threat intelligence
     ↓
⚖️ DECIDE: Is abuse score > 50?
     ↓
   YES → 🚨 Slack Alert + 📝 Log as "High Risk"
   NO  → 📝 Log as "Low Risk"
     ↓
📤 OUTPUT: Return full API response`

---

## **✅ Summary: What Was Sanitized**

| Original | Sanitized |
| --- | --- |
| `040fb7c0b0630c14...` (API key) | `YOUR_ABUSEIPDB_API_KEY` |
| `C0A6YEB58QN` (Slack channel ID) | `YOUR_SLACK_CHANNEL_ID` |
| `ykhuGJlBwPmMQ5lZ` (Slack credential ID) | `YOUR_SLACK_CREDENTIAL_ID` |
| `cL7zcYjC45DYXSKl` (Data table ID) | `YOUR_DATA_TABLE_ID` |
| `/projects/rFAsFQqsLss5x3aK/...` (URL) | `YOUR_DATA_TABLE_URL` |
| Ngrok URL in sticky note | Removed entirely |
| Instance ID | Removed |
