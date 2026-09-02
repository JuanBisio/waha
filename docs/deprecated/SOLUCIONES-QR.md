# Soluciones para el Problema del QR Code

## ❌ Problema Identificado

Evolution API con WHATSAPP-BAILEYS no está generando el QR code. Esto es un problema conocido con la integración de Baileys en algunas versiones.

## ✅ 3 Soluciones Disponibles

### Opción 1️⃣: WAHA (MÁS FÁCIL - RECOMENDADO)

**WAHA** es una alternativa más simple y estable. Funciona mejor que Evolution API.

#### Pasos:

1. **Detener Evolution API:**
```bash
cd /Users/bisiojuan/Desktop/joacoJuan
export PATH="/Applications/Docker.app/Contents/Resources/bin:$PATH"
docker compose down
```

2. **Iniciar WAHA:**
```bash
docker compose -f docker-compose-waha.yml up -d
```

3. **Crear sesión WhatsApp:**
```bash
curl -X POST http://localhost:3000/api/sessions/start \
  -H "X-Api-Key: SaaS_Instance_Key_2026_Secure_Token_12345" \
  -H "Content-Type: application/json" \
  -d '{"name": "default"}'
```

4. **Obtener QR Code:**
```bash
curl -X GET "http://localhost:3000/api/sessions/default/qr" \
  -H "X-Api-Key: SaaS_Instance_Key_2026_Secure_Token_12345"
```

El QR estará en base64. Puedes verlo en el navegador:
```
http://localhost:3000/api/sessions/default/qr
```

5. **Actualizar el workflow de n8n:**
   - Cambiar la URL de: `http://localhost:8080/message/sendText/SaaS_Instance`
   - A: `http://localhost:3000/api/sendText`
   - Cambiar header de `apikey` a `X-Api-Key`
   - Cambiar el body a:
     ```json
     {
       "session": "default",
       "chatId": "{phone}@c.us",
       "text": "{message}"
     }
     ```

---

### Opción 2️⃣: CallMeBot API (SIN DOCKER - MUY SIMPLE)

**CallMeBot** es un servicio gratuito que no requiere Docker ni QR codes complicados.

#### Pasos:

1. **Agregar el contacto de CallMeBot a tu WhatsApp:**
   - Guarda este número: `+34 644 46 96 74`
   - Envíale un mensaje que diga: `I allow callmebot to send me messages`
   - Espera la respuesta con tu API key

2. **Actualizar el workflow de n8n:**
   - URL: `https://api.callmebot.com/whatsapp.php`
   - Parámetros:
     ```
     phone: {tu_número_sin_+}
     text: {mensaje}
     apikey: {tu_api_key_de_callmebot}
     ```

**Ventajas:**
- ✅ Sin Docker
- ✅ Sin QR codes
- ✅ Configuración en 2 minutos
- ✅ Gratis

**Desventaja:**
- ⚠️ Solo puedes enviar a tu propio número (no a clientes)

---

### Opción 3️⃣: Reintentar Evolution API con otra versión

Intentar con una versión anterior más estable:

```bash
export PATH="/Applications/Docker.app/Contents/Resources/bin:$PATH"
docker compose down
```

Edita `docker-compose.yml` y cambia:
```yaml
evolution-api:
  image: atendai/evolution-api:v1.7.4  # Versión estable anterior
```

Luego:
```bash
docker compose up -d
```

---

## 🎯 Mi Recomendación

Para **producción con clientes reales**: Usa **Opción 1 (WAHA)**

Para **pruebas rápidas solo contigo**: Usa **Opción 2 (CallMeBot)**

---

## ¿Qué opción prefieres?

Responde con el número (1, 2 o 3) y te ayudo a configurarla paso a paso.
