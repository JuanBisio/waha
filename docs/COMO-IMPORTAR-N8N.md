# 📋 Pasos para Importar el Workflow a n8n

## 1. Accede a n8n

Abre tu navegador y ve a: **https://bisiojuan.app.n8n.cloud**

## 2. Importa el Workflow

1. Haz clic en **"Workflows"** en la barra lateral
2. Haz clic en **"Add workflow"** → **"Import from File"**
3. Selecciona el archivo: `n8n-whatsapp-workflow.json` 
   (está en `/Users/bisiojuan/Desktop/joacoJuan/`)
4. O copia y pega el contenido JSON directamente

## 3. Activa el Workflow

1. Una vez importado, haz clic en **"Activate"** (toggle en la esquina superior derecha)
2. El workflow debe mostrar estado: **"Active"**

## 4. Copia la URL del Webhook

1. Haz clic en el nodo **"Webhook Trigger"**
2. Copia la **Production URL** (algo como):
   ```
   https://bisiojuan.app.n8n.cloud/webhook/appointment-created
   ```

## 5. Configura Supabase

Ahora necesitas configurar la tabla `shops` en Supabase con la URL del webhook.

### Opción A: SQL Editor en Supabase

1. Ve a https://supabase.com/dashboard
2. Proyecto: **TurneroAutomatico**
3. SQL Editor
4. Ejecuta esta query (reemplaza `TU_WEBHOOK_URL`):

```sql
UPDATE shops
SET 
  webhook_url = 'TU_WEBHOOK_URL',
  webhook_enabled = true
WHERE slug = 'joaquin';
```

**Ejemplo:**
```sql
UPDATE shops
SET 
  webhook_url = 'https://bisiojuan.app.n8n.cloud/webhook/appointment-created',
  webhook_enabled = true
WHERE slug = 'joaquin';
```

### Opción B: Via Terminal

```bash
curl -X POST https://vfeaoejqmvoeebzthxnw.supabase.co/rest/v1/shops \
  -H "apikey: TU_SUPABASE_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "webhook_url": "TU_WEBHOOK_URL",
    "webhook_enabled": true
  }'
```

## 6. Prueba el Sistema Completo

### Prueba 1: Test Manual del Webhook

```bash
curl -X POST 'TU_WEBHOOK_URL' \
  -H 'Content-Type: application/json' \
  -d '{
    "event": "appointment.created",
    "customer_name": "Test User",
    "customer_phone": "+5493584022597",
    "service_name": "Corte de Cabello",
    "professional_name": "Juan Pérez",
    "start_time_formatted": "2026-01-23 15:00:00"
  }'
```

**Deberías recibir un WhatsApp** en tu teléfono en unos segundos.

### Prueba 2: Crear Reserva Real

1. Ve a: `http://localhost:3000/widget/joaquin`
2. Completa el flujo de reserva con un teléfono real
3. Verifica:
   - ✅ Webhook se dispara en Supabase
   - ✅ n8n recibe la notificación
   - ✅ WhatsApp se envía correctamente

## 7. Verificar Logs

### En n8n:
1. Ve a **"Executions"** en la barra lateral
2. Busca la última ejecución
3. Revisa que todos los nodos estén en verde ✅

### En Supabase:
```sql
SELECT * FROM webhook_logs 
ORDER BY created_at DESC 
LIMIT 10;
```

---

## ✅ Resumen de URLs y Configuración

| Item | Valor |
|------|-------|
| **WAHA URL** | `http://localhost:3000` |
| **WAHA API Key** | `SaaS_Instance_Key_2026_Secure_Token_12345` |
| **WAHA Session** | `default` |
| **n8n Dashboard** | `https://bisiojuan.app.n8n.cloud` |
| **Workflow File** | `/Users/bisiojuan/Desktop/joacoJuan/n8n-whatsapp-workflow.json` |
| **Supabase Project** | `TurneroAutomatico` |

---

## 🆘 Troubleshooting

**Webhook no se dispara:**
```sql
-- Verificar configuración
SELECT webhook_url, webhook_enabled FROM shops WHERE slug = 'joaquin';
```

**WhatsApp no se envía:**
```bash
# Verificar WAHA está corriendo
curl -H "X-Api-Key: SaaS_Instance_Key_2026_Secure_Token_12345" \
  http://localhost:3000/api/sessions

# Debería mostrar: "status": "WORKING"
```

**Formato de teléfono incorrecto:**
- El teléfono debe estar en formato: `5493584022597@c.us`
- Sin espacios, sin guiones, sin + 
- Debe terminar con `@c.us`

---

🎉 **¡Listo!** Una vez que completes estos pasos, el sistema enviará confirmaciones automáticas por WhatsApp cada vez que alguien reserve un turno.
