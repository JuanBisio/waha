# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Turnero SaaS** — Sistema multi-tenant de reserva de turnos con automatización de WhatsApp.

- **Stack:** Next.js 16 + React 19 + TypeScript + Supabase (PostgreSQL) + Tailwind CSS 4
- **WhatsApp:** YCloud API (reemplazó Evolution API + n8n) — solo notificaciones, sin bot conversacional
- **CAPTCHA:** Cloudflare Turnstile en el widget público

## Estructura del repositorio

```
Gendo/
├── turnero-saas/          ← Aplicación Next.js (submódulo git, repo propio)
├── docs/                  ← Documentación (ver docs/README.md para el índice)
│   ├── setup/             ← Guías de configuración vigentes
│   ├── planes/            ← Planes de arquitectura y migración
│   ├── prompts/           ← Prompts de trabajo para agentes
│   ├── seguridad/         ← Auditorías pendientes
│   ├── referencia/        ← Especificaciones técnicas
│   └── deprecated/        ← Docs históricos (Evolution API, WAHA, n8n)
├── workflows/deprecated_n8n/  ← Workflows JSON de n8n (solo referencia)
├── assets/                ← Imágenes y recursos estáticos
└── docker-compose.yml
```

**Importante:** `turnero-saas/` es un submódulo apuntando a `JuanBisio/Turnero-Saas`. Los cambios en la app se commitean dentro de ese repo, no en la raíz.

## Comandos

```bash
cd turnero-saas

npm run dev       # Servidor de desarrollo
npm run build     # Build de producción
npm run lint      # ESLint
npm test          # Vitest (tests unitarios en src/**/*.test.ts)
```

## Variables de entorno

Ver `turnero-saas/.env.local.example`. Variables requeridas:
- `NEXT_PUBLIC_SUPABASE_URL` / `NEXT_PUBLIC_SUPABASE_ANON_KEY` / `SUPABASE_SERVICE_ROLE_KEY`
- `YCLOUD_API_KEY` / `YCLOUD_DEFAULT_SENDER` / `YCLOUD_WEBHOOK_SECRET`
- `WEBHOOK_MASTER_SECRET` / `CANCELLATION_SECRET`
- `NEXT_PUBLIC_APP_URL` / `NEXT_PUBLIC_TURNSTILE_SITE_KEY`

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
/api/v1/whatsapp/inbound                    ← Webhook de YCloud, single-tenant legacy
/api/v1/whatsapp/inbound/[shop_slug]        ← Webhook de YCloud, multi-tenant (por shop)
/api/v1/whatsapp/outbound                   ← Envío desde dashboard
```

### Motor de disponibilidad (`src/lib/availability/`)

Módulo central que calcula slots disponibles considerando:
- Horarios de profesionales (`schedules`)
- Excepciones (días bloqueados)
- Turnos ya reservados (detección de colisiones)

Entrypoint: `getAvailableSlots.ts`

### WhatsApp con YCloud (`src/lib/whatsapp/`)

No hay bot conversacional — se sacó (no funcionaba de forma confiable y era mucho lío
configurar por negocio). El flujo es solo notificaciones salientes:

- `signatureVerifier.ts` — Verifica HMAC-SHA256 del webhook (falla cerrado sin secret)
- `ycloudService.ts` / `ycloudClient.ts` — Cliente HTTP con reintentos hacia la API de YCloud
- `notificationService.ts` — Arma y envía la confirmación de turno (template aprobado por Meta)

Los mensajes entrantes (`/api/v1/whatsapp/inbound[/[shop_slug]]`) solo se validan y se
guardan en el Inbox del dashboard vía `handle_inbound_message` — no generan ninguna
respuesta automática.

### Widget de reserva (`src/components/widget/`)

Estado manejado con reducer (`bookingReducer.ts`) y contexto (`BookingProvider.tsx`). Flujo en 5 pasos: servicio → profesional → fecha → hora → datos del cliente.

### Base de datos

Supabase con RLS habilitado. Migraciones en `supabase/migrations/` (68 archivos). Las tablas principales: `shops`, `professionals`, `schedules`, `services`, `appointments`, `exceptions`, `webhook_logs`, `whatsapp_sessions`.

**Atención:** `docs/seguridad/errores-seguridad.md` documenta vulnerabilidades RLS críticas identificadas que requieren atención. Algunas políticas RLS fueron deshabilitadas en migración de debug (`9000_disable_rls_debug.sql`) y no se ha confirmado su restauración completa.

## Estado del proyecto

- Fases 1–5 completadas (arquitectura, disponibilidad, dashboard, widget, webhooks)
- Migración en curso: flujos n8n → implementaciones nativas Next.js en `/api/v1/whatsapp/`
- Los workflows de n8n anteriores están en `workflows/deprecated_n8n/` solo como referencia

## Alias de paths

TypeScript usa `@/*` → `./src/*` (configurado en `tsconfig.json`).

## Seguimiento de tareas en Notion

Las tareas pendientes de Turnero se trackean en el Notion personal del usuario: Habit Tracker → base "Tareas" → tablero filtrado por Contexto = "Turnero". Convención: Área = Trabajo, Contexto = relación a esa página, Prioridad (Alta/Media/Baja), Estado (Pendiente/En curso/Hecha/Archivada).

Regla: cada vez que hagas un push o merge en este repo, revisá si ese cambio resuelve alguna tarea pendiente en ese tablero de Notion y marcala como Hecha, avisándome qué tareas se actualizaron y por qué. Esto aplica solo cuando el push/merge lo hacés vos (Claude Code) dentro de una sesión con el conector de Notion disponible — no cubre pushes hechos manualmente fuera de una sesión con vos.
