# 🚀 n8n Docker Setup & Webhook Workflow Automation

A complete guide to setting up **n8n in Docker** with **persistent storage**, **webhook configuration**, and an example **AI receptionist / lead automation workflow**.

---

## 🧭 Table of Contents
1. [Introduction](#introduction)
2. [Prerequisites](#prerequisites)
3. [Install Docker & Docker Compose](#install-docker--docker-compose)
4. [Setup n8n in Docker](#setup-n8n-in-docker)
5. [Persistent Storage](#persistent-storage)
6. [Environment Variables](#environment-variables)
7. [Webhook Configuration](#webhook-configuration)
8. [Testing Your Setup](#testing-your-setup)
9. [Securing with Reverse Proxy (Optional)](#securing-with-reverse-proxy-optional)
10. [Custom Workflow: AI Receptionist Automation](#custom-workflow-ai-receptionist-automation)
11. [Workflow Steps Explained](#workflow-steps-explained)
12. [Node Configuration Examples](#node-configuration-examples)
13. [Integration Examples (Slack, Email, Sheets)](#integration-examples-slack-email-sheets)
14. [Troubleshooting](#troubleshooting)

---

## 1. Introduction
This guide helps you deploy **n8n**, an automation tool similar to Zapier, using Docker.  
You'll also learn how to create a production workflow for **AI receptionist and lead management automation**.

---

## 2. Prerequisites
- A server or local machine with:
  - Docker installed
  - Docker Compose installed
  - Node and npm (optional, for local testing)
- Basic understanding of webhooks

---

## 3. Install Docker & Docker Compose

**Ubuntu Example:**

```bash
sudo apt update && sudo apt install -y docker.io docker-compose
sudo systemctl enable docker
sudo systemctl start docker
```

Check installation:
```bash
docker --version
docker-compose --version
```

---

## 4. Setup n8n in Docker

Create a folder for n8n setup:
```bash
mkdir n8n-docker && cd n8n-docker
```

Create `docker-compose.yml`:

```yaml
version: '3.7'

services:
  n8n:
    image: n8nio/n8n:latest
    restart: unless-stopped
    ports:
      - "5678:5678"
    env_file:
      - .env
    volumes:
      - ./n8n_data:/home/node/.n8n
```

---

## 5. Persistent Storage

All n8n credentials and workflow data will be stored in the `./n8n_data` folder.

Ensure correct permissions:
```bash
sudo chmod -R 777 n8n_data
```

---

## 6. Environment Variables

Create a `.env` file in the same directory:

```bash
# Basic n8n setup
GENERIC_TIMEZONE=Asia/Kolkata
WEBHOOK_TUNNEL_URL=https://your-domain.com/
N8N_PORT=5678
N8N_PROTOCOL=http

# Optional authentication
N8N_BASIC_AUTH_ACTIVE=true
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=securepassword

# Data folder
DATA_FOLDER=/home/node/.n8n

# Executions
EXECUTIONS_DATA_SAVE_ON_ERROR=all
EXECUTIONS_DATA_SAVE_ON_SUCCESS=none
EXECUTIONS_DATA_SAVE_MANUAL_EXECUTIONS=true
```

Start n8n:
```bash
docker-compose up -d
```

Check logs:
```bash
docker-compose logs -f
```

---

## 7. Webhook Configuration

By default, n8n exposes webhooks on:
```
http://localhost:5678/webhook/
http://localhost:5678/webhook-test/
```

If you’re using a public server, replace `localhost` with your domain.

You can expose the webhook externally using **ngrok** or a reverse proxy.

Example with ngrok:
```bash
ngrok http 5678
```

Copy the public URL and set it in `.env` under `WEBHOOK_TUNNEL_URL`.

---

## 8. Testing Your Setup

Go to `http://localhost:5678` → Create a new workflow → Add a **Webhook Node**.

Select:
- HTTP Method: `POST`
- Path: `/lead-intake`

Click **Execute Workflow**, then send a test request:
```bash
curl -X POST http://localhost:5678/webhook-test/lead-intake -H "Content-Type: application/json" -d '{"lead":"John Doe"}'
```

You should see the data appear in n8n.

---

## 9. Securing with Reverse Proxy (Optional)

You can secure your n8n instance using **Traefik** or **Nginx** for HTTPS with Let’s Encrypt.

Example (Traefik):
```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.n8n.rule=Host(`your-domain.com`)"
  - "traefik.http.routers.n8n.entrypoints=websecure"
  - "traefik.http.routers.n8n.tls.certresolver=myresolver"
```

---

## 10. Custom Workflow: AI Receptionist Automation

### Workflow Logic

```
Webhook → Check if picked up
    ├─ NO → Wait 1 day → Log for email follow-up
    └─ YES → Detect human/voicemail → Update stats
              ├─ VOICEMAIL → Log voicemail
              └─ HUMAN → Check appointment
                         ├─ BOOKED → Log + Slack notify + Stop emails
                         └─ NOT BOOKED → Categorize interest
                                        ├─ REJECTED → Get reason + Log + Stop emails
                                        └─ FUTURE → Log + Start email sequence
```

---

## 11. Workflow Steps Explained

### 🧩 Step 1: Webhook Trigger
- **Node:** Webhook
- **Method:** POST
- **Path:** `/ai-receptionist`
- **Sample JSON Payload:**
```json
{
  "call_id": "1234",
  "picked_up": true,
  "type": "human",
  "appointment": false
}
```

---

### 🧠 Step 2: Check if Picked Up
Use **IF node**:
- Condition: `picked_up` equals `true`  
If `false`, connect to “Wait” node.

---

### ⏳ Step 3: Wait 1 Day (if not picked)
- **Node:** Wait
- **Duration:** 1 day  
- Then → **Log follow-up** (Google Sheet or Database)

---

### 🗣 Step 4: Detect Human or Voicemail
- **Node:** IF
- **Condition:** `type` equals `human` or `voicemail`

---

### 📞 Step 5: If Human → Check Appointment
- **Node:** IF
- **Condition:** `appointment` equals `true`

If **true** → send Slack notification + stop email sequence.  
If **false** → continue to interest categorization.

---

### 💬 Step 6: Categorize Interest
- **Node:** Switch / IF
- Branch based on `interest`: `rejected` or `future`

---

### 🚫 Step 7: Rejected
- Get reason from API or webhook.
- Log into Google Sheet.
- Stop emails.

---

### ⏰ Step 8: Future
- Log lead info.
- Add to email sequence via SMTP / Gmail / Mailgun.

---

## 12. Node Configuration Examples

| Step | Node Type | Key Config | Output |
|------|------------|-------------|---------|
| 1 | Webhook | Path `/ai-receptionist` | Call data |
| 2 | IF | picked_up = true | YES/NO |
| 3 | Wait | 1 day | Delay follow-up |
| 4 | IF | type = human | human/voicemail |
| 5 | IF | appointment = true | Booked / Not Booked |
| 6 | Switch | interest = rejected/future | Categorize |
| 7 | Google Sheet | Append row | Logs |
| 8 | Slack | Send message | Notify |
| 9 | Email | Send via SMTP | Follow-up |

---

## 13. Integration Examples (Slack, Email, Sheets)

### Slack Notification
- Node: **Slack**
- Action: “Send Message”
- Channel: `#bookings`
- Message: `🎉 New appointment booked by {{ $json.name }}`

### Email Sequence
- Node: **Email**
- SMTP Configuration:
  - Host: `smtp.gmail.com`
  - Port: 587
  - Auth: OAuth2 or App Password

### Google Sheets
- Node: **Google Sheets**
- Action: “Append Row”
- Columns: Name, Call ID, Status, Notes

---

## 14. Troubleshooting

| Issue | Fix |
|-------|-----|
| Webhook not triggering | Check webhook URL and tunnel/public domain |
| 401 Unauthorized | Verify `N8N_BASIC_AUTH_*` values |
| Missing credentials | Ensure volume is persistent (`./n8n_data`) |
| HTTPS errors | Setup reverse proxy or Traefik |
| Workflow not executing | Ensure “Active” is toggled on |

---

**Author:** Ardhendu Abhishek Meher  
**Maintained for:** AI Receptionist & Automation Workflows  
**Theme Color:** #088178  
**License:** MIT

---
