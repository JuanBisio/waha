# Evolution API Setup Guide

## Quick Start

### 1. Start Evolution API

```bash
cd /Users/bisiojuan/Desktop/joacoJuan
docker-compose up -d
```

Wait 30-40 seconds for all services to initialize properly.

### 2. Verify Services are Running

```bash
docker-compose ps
```

You should see 3 containers running:
- `evolution_api` (port 8080)
- `evolution_postgres`
- `evolution_redis`

### 3. Access the Evolution API Dashboard

Open your browser and navigate to:
```
http://localhost:8080/manager
```

**API Key (AUTHENTICATION_API_KEY):**
```
SaaS_Instance_Key_2026_Secure_Token_12345
```

## Creating and Connecting WhatsApp Instance

### Step 1: Create Instance via API

Use the following curl command or access the manager UI:

```bash
curl -X POST http://localhost:8080/instance/create \
  -H "Content-Type: application/json" \
  -H "apikey: SaaS_Instance_Key_2026_Secure_Token_12345" \
  -d '{
    "instanceName": "SaaS_Instance",
    "qrcode": true,
    "integration": "WHATSAPP-BAILEYS"
  }'
```

### Step 2: Get QR Code

**Option A: Via API**
```bash
curl -X GET http://localhost:8080/instance/connect/SaaS_Instance \
  -H "apikey: SaaS_Instance_Key_2026_Secure_Token_12345"
```

**Option B: Via Manager UI**
1. Go to http://localhost:8080/manager
2. Enter API key when prompted
3. Click on "SaaS_Instance"
4. QR Code will be displayed

### Step 3: Scan QR Code

1. Open WhatsApp on your phone
2. Go to **Settings** → **Linked Devices**
3. Tap **Link a Device**
4. Scan the QR code displayed in the browser/API response
5. Wait for "Connected" status

### Step 4: Verify Connection

```bash
curl -X GET http://localhost:8080/instance/connectionState/SaaS_Instance \
  -H "apikey: SaaS_Instance_Key_2026_Secure_Token_12345"
```

Should return:
```json
{
  "instance": {
    "instanceName": "SaaS_Instance",
    "status": "open"
  }
}
```

## Testing WhatsApp Messages

### Send Test Message

```bash
curl -X POST http://localhost:8080/message/sendText/SaaS_Instance \
  -H "Content-Type: application/json" \
  -H "apikey: SaaS_Instance_Key_2026_Secure_Token_12345" \
  -d '{
    "number": "+5491112345678",
    "text": "Test message from Evolution API! 🚀"
  }'
```

**Note:** Replace `+5491112345678` with your actual WhatsApp number in international format.

## Important Configuration Details

### API Endpoints

- **Base URL:** `http://localhost:8080`
- **Manager UI:** `http://localhost:8080/manager`
- **Health Check:** `http://localhost:8080/health`
- **API Documentation:** `http://localhost:8080/docs`

### Instance Details

- **Instance Name:** `SaaS_Instance`
- **API Key:** `SaaS_Instance_Key_2026_Secure_Token_12345`

### Phone Number Format

All phone numbers MUST be in international format:
- ✅ Correct: `+5491112345678` (no spaces, no dashes)
- ❌ Wrong: `+54 9 11 1234-5678`
- ❌ Wrong: `1112345678`

## Managing the Service

### View Logs

```bash
# All services
docker-compose logs -f

# Evolution API only
docker-compose logs -f evolution-api

# Last 100 lines
docker-compose logs --tail=100 evolution-api
```

### Restart Services

```bash
# Restart all
docker-compose restart

# Restart Evolution API only
docker-compose restart evolution-api
```

### Stop Services

```bash
docker-compose down
```

### Stop and Remove Data (Fresh Start)

```bash
docker-compose down -v
```

**⚠️ Warning:** This will delete all data including connected WhatsApp sessions!

## Troubleshooting

### QR Code Not Appearing

1. Check if the instance exists:
   ```bash
   curl -X GET http://localhost:8080/instance/fetchInstances \
     -H "apikey: SaaS_Instance_Key_2026_Secure_Token_12345"
   ```

2. Recreate the instance if needed:
   ```bash
   curl -X DELETE http://localhost:8080/instance/delete/SaaS_Instance \
     -H "apikey: SaaS_Instance_Key_2026_Secure_Token_12345"
   ```
   Then create again using Step 1.

### Connection Lost

WhatsApp sessions can disconnect if:
- The phone loses internet connection
- WhatsApp Web sessions are manually logged out from the phone
- The Evolution API container restarts

**Solution:** Reconnect by getting a new QR code (Step 2).

### Messages Not Sending

1. Verify instance is connected:
   ```bash
   curl -X GET http://localhost:8080/instance/connectionState/SaaS_Instance \
     -H "apikey: SaaS_Instance_Key_2026_Secure_Token_12345"
   ```

2. Check that the phone number format is correct (international format)

3. Review Evolution API logs:
   ```bash
   docker-compose logs --tail=50 evolution-api
   ```

### Port 8080 Already in Use

If port 8080 is already in use, edit `docker-compose.yml`:

```yaml
evolution-api:
  ports:
    - "8081:8080"  # Change 8080 to 8081 or another available port
```

Then update all URLs to use the new port (e.g., `http://localhost:8081`).

## Data Persistence

All data is stored in Docker volumes:
- `postgres_data`: Database with messages, contacts, and chats
- `redis_data`: Session cache
- `evolution_instances`: WhatsApp instance configurations
- `evolution_store`: Media files and attachments

These volumes persist even when containers are stopped, so your WhatsApp connection will be maintained across restarts.

## Next Steps

Once Evolution API is running and connected:
1. ✅ Configure Supabase Database Webhook
2. ✅ Create n8n Workflow
3. ✅ Test end-to-end booking flow
