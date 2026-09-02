# Instrucciones para Claude Code: Migración de n8n a Nativo

Hola Claude Code, tu objetivo es finalizar la migración de los flujos de automatización de WhatsApp que actualmente se encuentran en n8n hacia nuestra solución nativa en Next.js (dentro de la carpeta `turnero-saas`).

## Contexto
El usuario quiere reemplazar por completo los webhooks y flujos de n8n por endpoints en Next.js utilizando YCloud para WhatsApp y Gemini para IA. Tenemos dos planes detallados en la raíz del proyecto que contienen la arquitectura, interfaces y el código de base.

## Archivos a leer obligatoriamente antes de empezar:
1. `plan-migracion-n8n-native.md` (Contiene los 3 flujos principales, interfaces y la máquina de estados).
2. `plan-whatsapp-ycloud.md` (Contiene la lógica de envío de notificaciones outbound, utilidades y el orquestador).

## Pasos de Ejecución Requeridos:

### 1. Interfaces y Tipos (Fase 0)
- Escribe `turnero-saas/src/types/ycloud.ts` unificando los tipos de ambos planes.
- Escribe `turnero-saas/src/types/whatsapp-session.ts` con las interfaces de sesión y máquina de estados del plan nativo.

### 2. Capa de Servicios y Utilidades (lib/whatsapp)
- Implementa `turnero-saas/src/lib/whatsapp/config.ts` (Variables de entorno).
- Implementa `turnero-saas/src/lib/whatsapp/signatureVerifier.ts`.
- Implementa `turnero-saas/src/lib/whatsapp/whatsappUtils.ts`.
- Implementa `turnero-saas/src/lib/whatsapp/ycloudClient.ts` y `ycloudService.ts` (o únelos de forma coherente según ambos planes).
- Implementa `turnero-saas/src/lib/whatsapp/shopContext.ts`.
- Implementa `turnero-saas/src/lib/whatsapp/sessionManager.ts`.
- Implementa `turnero-saas/src/lib/whatsapp/aiInterpreter.ts`.
- Implementa `turnero-saas/src/lib/whatsapp/stateMachine.ts`.
- Implementa `turnero-saas/src/lib/whatsapp/notificationService.ts`.

### 3. Endpoints (API Routes)
Crea las rutas en Next.js adaptando el código de los planes:
- Inbound Simple: `turnero-saas/src/app/api/v1/whatsapp/inbound/route.ts`
- Bot IA Conversacional: `turnero-saas/src/app/api/v1/whatsapp/inbound/ai/route.ts`
- Outbound (Dashboard/Webhooks internos): `turnero-saas/src/app/api/v1/whatsapp/outbound/route.ts`

### 4. Integración y Limpieza
- Asegúrate de actualizar el archivo de webhooks o creación de turnos existente en `turnero-saas/src/app/api/...` para usar `sendAppointmentConfirmation()` como se describe en `plan-whatsapp-ycloud.md`.
- Verifica que el código compile sin errores de TypeScript y formatea los archivos creados.

---
**IMPORTANTE:** 
- Al crear los archivos, asegúrate de mantener el path dentro de `turnero-saas/`.
- Sigue exactamente la lógica y estructuras propuestas en los archivos `.md`.
- Si necesitas crear tablas en supabase, infórmalo para que el usuario corra las funciones RPC o migraciones indicadas en los planes.
- Procede paso a paso, explicando qué archivos creaste o modificaste. Comienza leyendo los archivos `.md` indicados.
