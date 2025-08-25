# 🚀 Elastic Agent & Fleet Server Setup (Vultr • Ubuntu 22.04 • Windows Agent)

This guide documents how I set up **Elastic Fleet Server** on **Ubuntu 22.04 (Vultr)**, enrolled a **Windows Server 2022** endpoint with **Elastic Agent**, fixed Fleet URL/port issues, and verified logs in Kibana.

> ⚠️ Public repo note: I redact/blur real IPs/tokens in screenshots. Use placeholders like `203.0.113.25` (public) and `10.0.0.x` (private).

---

## 📌 Environment
- **Cloud:** Vultr (with **VPC 2.0** selected for the Fleet server)
- **Fleet Server Host:** Ubuntu 22.04
- **Stack UI:** Kibana (`http://<elastic-public-ip>:5601`)
- **Endpoint:** Windows Server 2022 (already deployed)
- **Access:** SSH (Linux), RDP (Windows)

---

## 🧭 High-Level Flow
1) Deploy Ubuntu VM on Vultr (with **VPC 2.0**; network selected; hostname set)  
2) In Kibana: **Management → Fleet → Add Fleet Server (Quick Start)**  
3) Generate Fleet Server command (use **https URL** with **public IP**)  
4) SSH into Ubuntu and run the generated **Fleet Server** command  
5) Adjust cloud + host firewalls (opened ranges during setup, then tightened)  
6) Kibana: Continue the wizard, **create Agent policy**  
7) Install **Elastic Agent** on Windows Server (PowerShell as Admin)  
8) Fix enrollment issues: **Fleet URL must be `:8220`**, not `:443`; used `--insecure` for lab  
9) Confirm Fleet Server **Online**, Agent **Enrolled**, Windows logs in **Discover**

---

## 🧱 Architecture (lab)

| Component | Role | Ports (final) |
|---|---|---|
| Ubuntu 22.04 (Vultr) | Fleet Server | 8220/tcp (from trusted IPs) |
| Windows Server 2022 | Elastic Agent | Outbound to Fleet Server 8220 |
| Elasticsearch | Backend | 9200/tcp (internal) |
| Kibana | UI | 5601/tcp (from my IP) |

📸 *Diagram placeholder*  
![Fleet Architecture](./screenshots/fleet-architecture.png)

---

## ✅ Prerequisites
- Kibana reachable at `http://<elastic-public-ip>:5601`
- Elastic user with permissions (e.g., `elastic`)
- 1x Ubuntu 22.04 VM (Fleet server), **VPC 2.0 selected**
- 1x Windows Server 2022 (agent endpoint)
- Your public IP to restrict firewall rules

---

## 🚀 Steps (what I actually did)

### **Step 1 — Deploy Fleet Server VM on Vultr**
- **Deploy new server**
- **Type:** Cloud Compute CPU
- **Image:** Ubuntu 22.04
- **Networking:** Selected **VPC 2.0** (ensure network is checked)
- **Hostname:** Set a label/hostname
- Click **Deploy**

📸 *Screenshots:*  
![Deploy Ubuntu](./screenshots/vultr-deploy-ubuntu.png)  
![Select VPC 2.0 / Network](./screenshots/vultr-select-network.png)  
![Hostname Set](./screenshots/vultr-hostname.png)

---

### **Step 2 — Open Kibana and Start Fleet Quick Start**
- Go to Kibana:  
http://<elastic-public-ip>:5601

markdown
Copy
Edit
- **Management → Fleet → Add Fleet Server**
- Choose **Quick start**
- Enter:
- **Name**: e.g., `fleet-server-01`
- **URL**: `https://<fleet-server-public-ip>` (**must be https**)
- Click **Generate** to get the install command

📸 *Screenshots:*  
![Add Fleet Server (Quick Start)](./screenshots/kibana-fleet-add-server.png)  
![Fleet URL Generate](./screenshots/kibana-fleet-url-generate.png)

---

### **Step 3 — SSH to Ubuntu & Update**
```bash
ssh root@<fleet-server-public-ip>

# Update repos & packages
apt-get update && apt-get upgrade -y
📸 Screenshots:


Step 4 — Install Fleet Server (Use Command From Kibana)
From Kibana’s “Install Fleet Server on a centralized host” (Step 2), copy the Linux command and run it here.

bash
Copy
Edit
# Example placeholder — use YOUR exact command from Kibana
# (It includes ES URL, service token, CA fingerprint/flags)
sudo ./elastic-agent install \
  --fleet-server-es=https://<elasticsearch-host>:9200 \
  --fleet-server-service-token=<SERVICE_TOKEN> \
  --fleet-server-policy=<FLEET_SERVER_POLICY> \
  --url=https://<fleet-server-public-ip>:8220
# In my lab I later used --insecure until certs were sorted
📸 Screenshot:

Step 5 — Cloud & Host Firewall Adjustments
During setup I opened broader ranges, then tightened later:

Vultr firewall (cloud):

Temporary: TCP 1–65535 from my public IP (to unblock the flow)

Final (recommended): only 8220/tcp (Fleet), 22/tcp (SSH), and what you truly need — from my IP only

📸 Screenshots:


UFW on ELK / Stack host (to allow Kibana/ES as needed):

bash
Copy
Edit
# On the Elasticsearch/Kibana VM
sudo ufw allow 9200/tcp        # (I allowed this during setup)
sudo ufw allow 5601/tcp        # if you need Kibana externally (from your IP)
sudo ufw status
UFW on Fleet server host (after install):

bash
Copy
Edit
sudo ufw allow 8220/tcp        # Fleet server port
sudo ufw allow 443/tcp         # (added during troubleshooting; see Step 8)
sudo ufw status
📸 Screenshot:

🔐 Best practice: Keep both layers (Vultr + UFW) restricted to your IP. Do not leave broad port ranges open.

Step 6 — Continue in Kibana & Create Agent Policy
Back in Kibana, click Continue after Fleet Server install

Create Policy → name it (e.g., Windows-Endpoint-Policy)

Add integrations you want (e.g., Windows, System)

📸 Screenshots:

Step 7 — Install Elastic Agent on Windows Server
Log in to your Windows Server

Open PowerShell (Run as Administrator)

From Kibana → Fleet → Add agent, copy the Windows command and paste it:

powershell
Copy
Edit
# Example placeholder — use YOUR exact command from Kibana’s Add agent page
.\elastic-agent.exe install `
  --url=https://<fleet-server-public-ip>:8220 `
  --enrollment-token=<AGENT_ENROLLMENT_TOKEN> `
  --insecure
I used --insecure to bypass unsigned cert errors in this lab

📸 Screenshots:


Step 8 — Fix Fleet URL Port (8220 vs 443)
Troubleshooting I hit:

In Kibana → Fleet → Settings, I had to edit the Fleet Server host URL to include :8220 (not :443)

I also modified the install URL from :443 to :8220 so agents connect properly

Kept --insecure during lab while certs weren’t in place

📸 Screenshots:


Step 9 — Verify Enrollment & Data
Kibana → Fleet → Fleet servers: Fleet shows Healthy/Online

Kibana → Fleet → Agents: Windows agent Online

Kibana → Discover: Windows logs appearing (e.g., logs-windows.*, metrics-system.*)

📸 Screenshots:
