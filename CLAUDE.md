# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Turnero SaaS** — Sistema multi-tenant de reserva de turnos con automatización de WhatsApp.

- **Stack:** Next.js 16 + React 19 + TypeScript + Supabase (PostgreSQL) + Tailwind CSS 4
- **WhatsApp:** YCloud API (reemplazó Evolution API + n8n)
- **IA:** Google Gemini para interpretación de mensajes entrantes
- **CAPTCHA:** Cloudflare Turnstile en el widget público

## Comandos

```bash
cd turnero-saas

npm run dev       # Servidor de desarrollo
npm run build     # Build de producción
npm run lint      # ESLint
```

## Variables de entorno

Ver `turnero-saas/.env.local.example`. Variables requeridas:
- `NEXT_PUBLIC_SUPABASE_URL` / `NEXT_PUBLIC_SUPABASE_ANON_KEY` / `SUPABASE_SERVICE_ROLE_KEY`
- `YCLOUD_API_KEY` / `YCLOUD_DEFAULT_SENDER` / `YCLOUD_WEBHOOK_SECRET`
- `WEBHOOK_MASTER_SECRET` / `CANCELLATION_SECRET`
- `NEXT_PUBLIC_APP_URL` / `NEXT_PUBLIC_TURNSTILE_SITE_KEY`
- `GEMINI_API_KEY`

## Arquitectura

### Multi-tenancy

Cada negocio es un `shop` identificado por `shop_slug`. El middleware (`src/middleware.ts`) detecta el slug en la URL y lo pasa en el header `x-shop-slug`. Todas las queries a Supabase deben filtrar por `shop_id` para respetar el aislamiento RLS.

### Rutas principales

```
/dashboard/[shop_slug]/{agenda,profesionales,servicios,configuracion,inbox}
/widget/[shop_slug]                    ← Widget público de reserva (5 pasos)
/widget/[shop_slug]/cancelar/[token]   ← Formulario de cancelación

/api/public/availability               ← Sin auth (widget)
/api/public/appointments               ← Sin auth (widget)
/api/v1/availability                   ← Con auth
/api/v1/appointments/cancel
/api/v1/admin/appointments/external    ← HMAC protegido
/api/v1/admin/webhooks/test
/api/v1/webhooks/trigger
/api/v1/whatsapp/inbound               ← Webhook de YCloud (mensajes entrantes)
/api/v1/whatsapp/inbound/ai            ← Bot IA con máquina de estados
/api/v1/whatsapp/outbound              ← Envío desde dashboard
```

### Motor de disponibilidad (`src/lib/availability/`)

Módulo central que calcula slots disponibles considerando:
- Horarios de profesionales (`schedules`)
- Excepciones (días bloqueados)
- Turnos ya reservados (detección de colisiones)

Entrypoint: `getAvailableSlots.ts`

### WhatsApp con YCloud (`src/lib/whatsapp/`)

Flujo de mensajes entrantes:
1. `signatureVerifier.ts` — Verifica HMAC-SHA256 del webhook
2. `sessionManager.ts` — Lee/escribe estado de conversación en Supabase (`whatsapp_sessions`)
3. `aiInterpreter.ts` — Llama a Gemini para interpretar intención del mensaje
4. `stateMachine.ts` — Avanza el estado de reserva/cancelación
5. `ycloudService.ts` / `messageFormatter.ts` — Construye y envía respuesta

Estados de sesión (`src/types/whatsapp-session.ts`): `IDLE → AWAITING_SERVICE → AWAITING_PROFESSIONAL → AWAITING_DATE → AWAITING_TIME`

### Widget de reserva (`src/components/widget/`)

Estado manejado con reducer (`bookingReducer.ts`) y contexto (`BookingProvider.tsx`). Flujo en 5 pasos: servicio → profesional → fecha → hora → datos del cliente.

### Base de datos

Supabase con RLS habilitado. Migraciones en `supabase/migrations/` (68 archivos). Las tablas principales: `shops`, `professionals`, `schedules`, `services`, `appointments`, `exceptions`, `webhook_logs`, `whatsapp_sessions`.

**Atención:** Existe un archivo `errores-seguridad.md` en la raíz con vulnerabilidades RLS críticas identificadas que requieren atención. Algunas políticas RLS fueron deshabilitadas en migración de debug (`9000_disable_rls_debug.sql`) y no se ha confirmado su restauración completa.

## Estado del proyecto

- Fases 1–5 completadas (arquitectura, disponibilidad, dashboard, widget, webhooks)
- Migración en curso: flujos n8n → implementaciones nativas Next.js en `/api/v1/whatsapp/`
- Los workflows de n8n anteriores están en `workflows/deprecated_n8n/` solo como referencia

## Alias de paths

TypeScript usa `@/*` → `./src/*` (configurado en `tsconfig.json`).
