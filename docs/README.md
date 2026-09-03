# Documentación — Turnero SaaS

Índice de la documentación del proyecto. El código de la aplicación vive en
[`turnero-saas/`](../turnero-saas) (submódulo, repo propio).

## 📁 Estructura

| Carpeta | Contenido |
| --- | --- |
| [`setup/`](setup/) | Guías de configuración **vigentes** (YCloud, Gemini, Supabase, notificaciones) |
| [`planes/`](planes/) | Planes de arquitectura y migración |
| [`prompts/`](prompts/) | Prompts de trabajo para agentes de código |
| [`seguridad/`](seguridad/) | Auditorías y vulnerabilidades pendientes |
| [`referencia/`](referencia/) | Especificaciones técnicas del proyecto |
| [`deprecated/`](deprecated/) | Documentación histórica (Evolution API, WAHA, n8n) — solo referencia |

---

## ⚙️ Setup (vigente)

| Documento | Para qué sirve |
| --- | --- |
| [YCLOUD_SETUP.md](setup/YCLOUD_SETUP.md) | Alta y configuración de la cuenta YCloud |
| [YCLOUD_WEBHOOK_SETUP.md](setup/YCLOUD_WEBHOOK_SETUP.md) | Webhook de mensajes entrantes + firma HMAC |
| [PLANTILLA_WHATSAPP.md](setup/PLANTILLA_WHATSAPP.md) | Plantilla a crear en Meta Business Manager |
| [GEMINI_SETUP.md](setup/GEMINI_SETUP.md) | API key de Gemini para el intérprete de mensajes |
| [GUIA-NOTIFICACIONES.md](setup/GUIA-NOTIFICACIONES.md) | Elegir canal de confirmación: WhatsApp (YCloud) o Email (Resend) |
| [ULTIMO-PASO-SUPABASE.md](setup/ULTIMO-PASO-SUPABASE.md) | Ejecución de SQL en el dashboard de Supabase |

## 🗺️ Planes

| Documento | Estado |
| --- | --- |
| [plan-migracion-n8n-native.md](planes/plan-migracion-n8n-native.md) | Migración n8n → endpoints nativos Next.js (en curso) |
| [plan-whatsapp-ycloud.md](planes/plan-whatsapp-ycloud.md) | Notificaciones outbound, utilidades y orquestador |

## 🤖 Prompts

| Documento | Objetivo |
| --- | --- |
| [prompt-para-claude-code.md](prompts/prompt-para-claude-code.md) | Ejecutar la migración n8n → nativo |
| [prompt-debug-claude-code.md](prompts/prompt-debug-claude-code.md) | Auditar y depurar los 3 endpoints de WhatsApp |
| [rediseño.md](prompts/rediseño.md) | Refactor visual "Midnight Glass" del dashboard |

## 🔒 Seguridad

- [errores-seguridad.md](seguridad/errores-seguridad.md) — vulnerabilidades RLS detectadas, **pendientes de resolver**
- [revision-2026-09-02.md](seguridad/revision-2026-09-02.md) — revisión manual (críticos/altos/medios) y seguimiento de las correcciones aplicadas en el commit `fa375f3`

## 📘 Referencia

- [AGENTS.MD](referencia/AGENTS.MD) — especificaciones técnicas del sistema

## 🗄️ Deprecated

Documentación del stack anterior (Evolution API + WAHA + n8n), reemplazado por
YCloud + endpoints nativos. Se conserva solo como referencia histórica:
QUICK-START, TROUBLESHOOTING, SOLUCIONES-QR, POR-QUE-WAHA, evolution-api-setup,
n8n-setup-guide, COMO-IMPORTAR-N8N, N8N_CONNECTIONS, N8N_MANUAL_CONFIG(_V2).

Los workflows JSON de n8n están en [`../workflows/deprecated_n8n/`](../workflows/deprecated_n8n).
