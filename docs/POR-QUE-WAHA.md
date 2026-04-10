# ❌ Por qué Evolution API no está funcionando

Después de probar múltiples versiones de Evolution API (latest, v1.7.4, v2.0.10), todas tienen problemas críticos:

1. **v2.2.3 (latest)**: No genera QR code con Baileys
2. **v1.7.4**: Error de base de datos (`Cannot read properties of undefined (reading 'db')`)
3. **v2.0.10**: Rechaza todos los database providers incluso cuando DATABASE_ENABLED=false

## ✅ Solución Definitiva: WAHA

WAHA es una alternativa más estable y mejor mantenida que Evolution API. Vamos a usarlo en su lugar.

### Instalación (YA CONFIGURADO)

Ya configuré el archivo `docker-compose-waha.yml`. Solo necesitas:

```bash
cd /Users/bisiojuan/Desktop/joacoJuan
export PATH="/Applications/Docker.app/Contents/Resources/bin:$PATH"
docker compose -f docker-compose-waha.yml up -d
```

### Cómo obtener el QR Code con WAHA

1. **Iniciar sesión:**
```bash
curl -X POST http://localhost:3000/api/sessions/start \
  -H "X-Api-Key: SaaS_Instance_Key_2026_Secure_Token_12345" \
  -H "Content-Type: application/json" \
  -d '{"name": "default"}'
```

2. **Ver QR en el navegador:**
```
http://localhost:3000/api/screenshot/session/default/qr
```

O descargarlo con:
```bash
curl "http://localhost:3000/api/screenshot/session/default/qr" \
  -H "X-Api-Key: SaaS_Instance_Key_2026_Secure_Token_12345" \
  --output qr-code.png

open qr-code.png
```

3. **Escanear con WhatsApp:**
   - WhatsApp → Settings → Linked Devices → Link a Device
   - Escanea el QR

### Actualizar el workflow de n8n para WAHA

En tu workflow de n8n, cambia el nodo "Send WhatsApp":

**URL:** `http://localhost:3000/api/sendText`  
**Headers:**
- `X-Api-Key`: `SaaS_Instance_Key_2026_Secure_Token_12345`
- `Content-Type`: `application/json`

**Body:**
```json
{
  "session": "default",
  "chatId": "+5491112345678@c.us",
  "text": "Tu mensaje aquí"
}
```

Nota: `chatId` debe tener formato `{phone}@c.us`

### ¿Quieres que te lo configure automáticamente?

Si me das permiso, puedo:
1. Detener Evolution API
2. Iniciar WAHA
3. Crear la sesión
4. Mostrarte el QR en tu navegador

¿Procedo?
