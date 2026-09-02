# Instrucciones para Claude Code: Depuración de Flujos de WhatsApp

Hola Claude Code, actualmente tenemos problemas con los 3 endpoints de WhatsApp que nos impiden que los flujos (inbound, inbound/ai y outbound) funcionen correctamente en el proyecto `turnero-saas`. 

Tu objetivo es **auditar, depurar y explicar** por qué no están funcionando y proponer las soluciones exactas.

## Tareas a realizar:

### 1. Auditoría de Endpoints (Rutas API)
Revisa minuciosamente el código de los siguientes archivos:
- `turnero-saas/src/app/api/v1/whatsapp/inbound/route.ts`
- `turnero-saas/src/app/api/v1/whatsapp/inbound/ai/route.ts`
- `turnero-saas/src/app/api/v1/whatsapp/outbound/route.ts`

**Puntos a verificar en las rutas:**
- ¿Se están parseando correctamente los payloads de YCloud (tipos `YCloudInboundEvent`, etc.)?
- ¿El manejador de la petición extrae correctamente el texto o interacciones del mensaje?
- ¿Existen respuestas bloqueantes que causen timeouts antes de que los servicios completen su ejecución (ej: esperar respuesta de Supabase o Gemini)?

### 2. Auditoría de Servicios Core
Revisa la implementación en `turnero-saas/src/lib/whatsapp/`:
- **Verificador de Firmas** (`signatureVerifier.ts`): ¿Se está comparando correctamente el hash HMAC con el `YCLOUD_WEBHOOK_SECRET`? (Un error común aquí devuelve un 401 Unauthorized silencioso).
- **YCloud Client** (`ycloudClient.ts` o `ycloudService.ts`): Verifica si los envíos están fallando por headers incorrectos, body estructurado mal según la V2 de YCloud, o manejo de reintentos defectuoso.
- **Base de Datos / Sesión**: Revisa `sessionManager.ts`, `shopContext.ts` y si existen llamadas RPG (`supabase.rpc`) a Supabase que podrían estar fallando silenciosamente si las funciones no están definidas en la BD o están fallando por permisos de RLS.
- **Inteligencia Artificial**: Revisa `aiInterpreter.ts`. ¿Gemini está respondiendo en el formato JSON esperado? Si responde texto plano o falla, ¿se maneja bien el default?

### 3. Plan de Reparación
Una vez que hayas analizado la infraestructura de código, dales respuesta a estas preguntas en un resumen claro para mí:

1. **Diagnóstico Inbound Simple:** ¿Por qué falla el guardado de los mensajes entrantes?
2. **Diagnóstico Inbound AI:** ¿En qué etapa se ahoga el bot? (¿Firma, Extracción, Supabase RPC, IA, o Máquina de Estados?)
3. **Diagnóstico Outbound:** ¿Cuál es el motivo de fallo en el envío desde el sistema hacia YCloud?
4. **Pasos accionables:** ¿Qué archivos debemos modificar o qué configuración de entorno / SQL / YCloud falta por ajustar?

---
**IMPORTANTE:**
- Inicia el proceso leyendo los archivos mencionados (usa `cat` o equivalente).
- Si encuentras código que envuelve funciones asíncronas sin `await` o usa `void` inadecuadamente generando contextos perdidos, destácalo.
- Explícame la causa raíz de los problemas de la forma más clara y concisa posible antes de hacer cualquier cambio en el código. ¡Espero tu diagnóstico!
