# 🎯 Último Paso: Configurar Supabase

## Opción 1: SQL Editor (MÁS FÁCIL - 2 MINUTOS)

1. **Ve a Supabase:**
   - Abre: https://supabase.com/dashboard
   - Proyecto: **TurneroAutomatico**

2. **Abre el SQL Editor:**
   - En el menú lateral, busca **"SQL Editor"**
   - Haz clic en **"New query"**

3. **Copia y pega este SQL:**
   ```sql
   UPDATE shops
   SET 
     webhook_url = 'https://bisiojuan.app.n8n.cloud/webhook/appointment-created',
     webhook_enabled = true
   WHERE slug = 'joaquin';
   
   SELECT name, webhook_url, webhook_enabled FROM shops WHERE slug = 'joaquin';
   ```

4. **Ejecuta la query:**
   - Haz clic en **"Run"** (o presiona Ctrl/Cmd + Enter)

5. **Verifica el resultado:**
   - Deberías ver una fila que muestra:
     - `name`: Joaquin
     - `webhook_url`: https://bisiojuan.app.n8n.cloud/webhook/appointment-created
     - `webhook_enabled`: true

---

## Opción 2: Table Editor (ALTERNATIVA)

1. Ve a Supabase Dashboard
2. Busca la tabla **"shops"**
3. Encuentra la fila donde `slug = 'joaquin'`
4. Edita estos campos:
   - `webhook_url`: `https://bisiojuan.app.n8n.cloud/webhook/appointment-created`
   - `webhook_enabled`: `true` (checkbox marcado)
5. Guarda los cambios

---

## ✅ Verificación Final

Después de configurar Supabase, vamos a probar el sistema completo:

### Prueba 1: Test Manual del Webhook

Ejecuta este comando en tu terminal:

```bash
curl -X POST 'https://bisiojuan.app.n8n.cloud/webhook/appointment-created' \
  -H 'Content-Type: application/json' \
  -d '{
    "event": "appointment.created",
    "customer_name": "Test Usuario",
    "customer_phone": "+5493584022597",
    "service_name": "Corte de Cabello",
    "professional_name": "Juan Pérez",
    "start_time_formatted": "2026-01-23 15:00:00"
  }'
```

**¿Qué debería pasar?**
- ✅ Recibes un WhatsApp en tu teléfono con la confirmación
- ✅ El mensaje tiene el formato bonito que configuramos

### Prueba 2: Crear una Reserva Real

1. Abre tu app: `http://localhost:3000/widget/joaquin`
2. Completa una reserva con tu número de teléfono
3. Verifica que llegue el WhatsApp automáticamente

---

## 🔍 Cómo verificar que todo funciona

### Ver logs de webhooks en Supabase:

```sql
SELECT 
  created_at,
  status,
  payload->>'customer_name' as customer,
  payload->>'customer_phone' as phone,
  error_message
FROM webhook_logs
ORDER BY created_at DESC
LIMIT 10;
```

### Ver ejecuciones en n8n:

1. Ve a n8n
2. Haz clic en **"Executions"** en la barra lateral
3. Verás todas las ejecuciones del workflow

---

## 📋 Resumen del Sistema Completo

| Componente | Estado | URL/Config |
|------------|--------|------------|
| **WAHA** | ✅ Conectado | http://localhost:3000 |
| **WhatsApp** | ✅ Vinculado | +5493584022597 |
| **n8n Workflow** | ⏳ Pendiente activar | https://bisiojuan.app.n8n.cloud |
| **Webhook URL** | ✅ Configurado | https://bisiojuan.app.n8n.cloud/webhook/appointment-created |
| **Supabase** | ⏳ Pendiente configurar | webhook_url + webhook_enabled |

---

## 🆘 ¿Necesitas ayuda?

Si algo no funciona:
1. Verifica que WAHA esté corriendo: `curl http://localhost:3000/api/sessions`
2. Verifica que n8n workflow esté activo (toggle verde en n8n)
3. Revisa los logs en Supabase: `SELECT * FROM webhook_logs ORDER BY created_at DESC LIMIT 5;`

---

**¿Ya configuraste Supabase? Avísame cuando termines y hacemos una prueba completa.** 🚀
