# 🚀 Quick Start Guide - WhatsApp Confirmation System

## You're Ready! Here's What to Do Next:

### Step 1: Start Evolution API (5 minutes)

```bash
cd /Users/bisiojuan/Desktop/joacoJuan
docker-compose up -d
```

Wait 30-40 seconds, then verify:
```bash
docker-compose ps
```

### Step 2: Connect Your WhatsApp (2 minutes)

1. **Create instance:**
```bash
curl -X POST http://localhost:8080/instance/create \
  -H "Content-Type: application/json" \
  -H "apikey: SaaS_Instance_Key_2026_Secure_Token_12345" \
  -d '{"instanceName": "SaaS_Instance", "qrcode": true, "integration": "WHATSAPP-BAILEYS"}'
```

2. **Get QR code:**
```bash
curl -X GET http://localhost:8080/instance/connect/SaaS_Instance \
  -H "apikey: SaaS_Instance_Key_2026_Secure_Token_12345"
```

3. **Scan with your phone:**
   - Open WhatsApp → Settings → Linked Devices → Link a Device
   - Scan the QR code from the response

### Step 3: Import n8n Workflow (3 minutes)

1. Go to: https://bisiojuan.app.n8n.cloud
2. Import file: `n8n-whatsapp-workflow.json`
3. Activate the workflow
4. **Copy the webhook URL** (looks like: `https://bisiojuan.app.n8n.cloud/webhook/appointment-created`)

### Step 4: Configure Supabase (1 minute)

Replace `YOUR_WEBHOOK_URL` with the URL from Step 3:

```sql
UPDATE shops
SET webhook_url = 'YOUR_WEBHOOK_URL', webhook_enabled = true
WHERE slug = 'joaquin';
```

### Step 5: Test! (2 minutes)

**Quick Test:**
```bash
curl -X POST 'YOUR_WEBHOOK_URL' \
  -H 'Content-Type: application/json' \
  -d '{
    "customer_name": "Test",
    "customer_phone": "+5491112345678",
    "service_name": "Corte",
    "professional_name": "Juan",
    "start_time_formatted": "2026-01-21 15:00:00"
  }'
```

Replace `+5491112345678` with your number. Check your phone!

**Full Test:**
1. `npm run dev` in turnero-saas
2. Go to: http://localhost:3000/widget/joaquin
3. Book appointment with your phone number
4. Check WhatsApp!

---

## 📋 Important Info

**API Key:** `SaaS_Instance_Key_2026_Secure_Token_12345`  
**Evolution Panel:** http://localhost:8080/manager  
**Instance Name:** `SaaS_Instance`

## 📚 Full Documentation

- [evolution-api-setup.md](file:///Users/bisiojuan/Desktop/joacoJuan/evolution-api-setup.md) - Complete Evolution API guide
- [n8n-setup-guide.md](file:///Users/bisiojuan/Desktop/joacoJuan/n8n-setup-guide.md) - n8n workflow setup
- [walkthrough.md](file:///Users/bisiojuan/.gemini/antigravity/brain/9cbc6ec9-e67f-4406-bd7a-05f59b10f924/walkthrough.md) - Complete implementation details

## ❓ Need Help?

Check the troubleshooting sections in the guides above, or review the Evolution API logs:
```bash
docker-compose logs -f evolution-api
```
