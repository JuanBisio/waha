# n8n Workflow Setup Guide

## Prerequisites

- n8n instance running at: `https://bisiojuan.app.n8n.cloud`
- Evolution API running on `localhost:8080` (see evolution-api-setup.md)

## Step 1: Import the Workflow

1. **Access n8n Dashboard:**
   - Open: https://bisiojuan.app.n8n.cloud
   - Log in with your credentials

2. **Import Workflow:**
   - Click on **"Workflows"** in the sidebar
   - Click **"Add workflow"** → **"Import from File"**
   - Select the file: `n8n-whatsapp-workflow.json`
   - Or copy-paste the JSON content directly

3. **Activate the Workflow:**
   - After importing, click **"Activate"** toggle in the top right
   - The workflow should show as "Active"

## Step 2: Get the Webhook URL

1. **Open the Webhook Node:**
   - Click on the **"Webhook Trigger"** node in the workflow
   - Copy the **Production URL** (should look like):
     ```
     https://bisiojuan.app.n8n.cloud/webhook/appointment-created
     ```

2. **Save this URL** - you'll need it for Step 3

## Step 3: Configure Supabase Shop

Now we need to tell Supabase where to send the webhook notifications.

**Option A: Via SQL (Recommended)**

Run this SQL query in Supabase SQL Editor, replacing `YOUR_WEBHOOK_URL`:

```sql
UPDATE shops
SET 
  webhook_url = 'YOUR_WEBHOOK_URL',
  webhook_enabled = true
WHERE slug = 'joaquin';  -- Or your shop slug
```

**Option B: Via Supabase MCP (if you prefer)**

Use the Antigravity agent to run:
```sql
UPDATE shops
SET webhook_url = 'YOUR_N8N_WEBHOOK_URL', webhook_enabled = true
WHERE slug = 'YOUR_SHOP_SLUG';
```

## Step 4: Update Evolution API URL (if needed)

If Evolution API is NOT running on localhost (e.g., running on a server), update the workflow:

1. Open the **"Send WhatsApp"** node
2. Update the URL from:
   ```
   http://localhost:8080/message/sendText/SaaS_Instance
   ```
   To your actual Evolution API URL:
   ```
   http://YOUR_SERVER_IP:8080/message/sendText/SaaS_Instance
   ```

3. **Save** the workflow

## Step 5: Test the Workflow

### Manual Test (Recommended First)

1. In n8n, click **"Test workflow"**
2. Click on **"Webhook Trigger"** node
3. Click **"Listen for test event"**
4. Open a new terminal/Postman and send a test webhook:

```bash
curl -X POST 'YOUR_WEBHOOK_URL' \
  -H 'Content-Type: application/json' \
  -d '{
    "event": "appointment.created",
    "customer_name": "Test User",
    "customer_phone": "+5491112345678",
    "service_name": "Corte de Cabello",
    "professional_name": "Juan Pérez",
    "start_time_formatted": "2026-01-21 15:00:00"
  }'
```

**Replace:**
- `YOUR_WEBHOOK_URL` with your actual n8n webhook URL
- `+5491112345678` with YOUR actual WhatsApp number

5. Check your phone - you should receive a WhatsApp message!

### End-to-End Test

1. Go to your booking widget: `http://localhost:3000/widget/joaquin` (or your shop slug)
2. Complete the booking flow with your actual phone number
3. Verify you receive the WhatsApp confirmation

## Workflow Overview

The workflow has 5 nodes:

1. **Webhook Trigger**: Receives data from Supabase
2. **Extract Data**: Pulls out relevant fields (phone, name, service, etc.)
3. **Format Message**: Cleans phone number and creates WhatsApp message template
4. **Send WhatsApp**: Calls Evolution API to send the message
5. **Check Success**: Verifies the message was sent successfully

### Message Template

The default message is:
```
¡Hola [Name]! 👋

Tu reserva ha sido confirmada exitosamente.

📅 Fecha: [DD/MM/YYYY]
⏰ Hora: [HH:MM]
💈 Servicio: [Service Name]
👤 Profesional: [Professional Name]

¡Nos vemos pronto!
```

You can customize this in the **"Format Message"** node.

## Troubleshooting

### Webhook not triggering

1. Check that `webhook_enabled = true` in the shops table:
   ```sql
   SELECT webhook_url, webhook_enabled FROM shops WHERE slug = 'joaquin';
   ```

2. Check webhook logs in Supabase:
   ```sql
   SELECT * FROM webhook_logs ORDER BY created_at DESC LIMIT 10;
   ```

### WhatsApp message not sending

1. **Verify Evolution API is running:**
   ```bash
   curl http://localhost:8080/instance/connectionState/SaaS_Instance \
     -H "apikey: SaaS_Instance_Key_2026_Secure_Token_12345"
   ```
   Should return `status: "open"`

2. **Check n8n execution logs:**
   - Go to **"Executions"** in n8n sidebar
   - Find the latest execution
   - Check if there are any errors in the **"Send WhatsApp"** node

3. **Verify phone number format:**
   - Must be in international format: `+5491112345678`
   - No spaces, no dashes, no parentheses

### Common Errors

| Error | Solution |
|-------|----------|
| `Instance not found` | Make sure Evolution API is running and instance `SaaS_Instance` is connected |
| `Invalid phone number` | Ensure phone is in international format with + prefix |
| `Webhook timeout` | Check that n8n URL is accessible from Supabase (not localhost) |
| `401 Unauthorized` | Verify the API key in the workflow matches Evolution API |

## Monitoring

### Check Recent Executions

In n8n:
1. Click **"Executions"** in the sidebar
2. View success/failure status
3. Click on any execution to see detailed logs

### Check Webhook Logs in Supabase

```sql
SELECT 
  created_at,
  status,
  payload->>'customer_name' as customer_name,
  payload->>'customer_phone' as customer_phone,
  error_message
FROM webhook_logs
ORDER BY created_at DESC
LIMIT 20;
```

## Next Steps

Once everything is working:
- ✅ Customize the WhatsApp message template
- ✅ Add error notifications (e.g., send email if WhatsApp fails)
- ✅ Monitor execution logs regularly
- ✅ Consider adding a delay node if you want to send reminders before appointments
