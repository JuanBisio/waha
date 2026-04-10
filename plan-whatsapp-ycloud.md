# Plan de Implementación: Automatización de WhatsApp con YCloud
> Migración del flujo n8n → solución nativa en `turnero-saas` · Versión 2.0

---

## 📋 Índice

1. [Fase 0 – Contrato de API y Tipos](#fase-0)
2. [Fase 1 – Análisis de Requisitos](#fase-1)
3. [Fase 2 – Diseño de la Solución](#fase-2)
4. [Fase 3 – Implementación](#fase-3)
5. [Fase 4 – Pruebas y Validación](#fase-4)
6. [Fase 5 – Despliegue y Transición](#fase-5)
7. [Resumen de Prioridades](#resumen)

---

## Fase 0 – Contrato de API y Tipos {#fase-0}

> **Por qué primero:** Documentar el contrato exacto de YCloud antes de codificar evita bugs de integración y sirve como documentación viva para el equipo.

### Crear `src/types/ycloud.ts`

```typescript
// Representa el cuerpo del request a YCloud
export interface YCloudTemplateRequest {
  to: string; // Número en formato internacional: "5491XXXXXXXXX"
  template_name: "confirmacion_turno";
  parameters: {
    p1_name: string;    // Nombre del cliente
    p2_service: string; // Nombre del servicio
    p3_date: string;    // Fecha formateada: "DD/MM/YYYY"
    p4_time: string;    // Hora formateada: "HH:mm"
    p5_prof: string;    // Nombre del profesional
  };
  from?: string; // Número del negocio (shop_sender). Opcional, tiene fallback.
}

// Representa la respuesta de YCloud
export interface YCloudTemplateResponse {
  id: string;
  status: "submitted" | "sent" | "failed";
  error?: {
    code: string;
    message: string;
  };
}

// Payload interno que viaja entre capas de nuestra app
export interface AppointmentNotificationPayload {
  clientName: string;
  clientPhone: string;
  serviceName: string;
  professionalName: string;
  datetime: string;         // ISO 8601: "2025-01-15T14:30:00"
  shopSender?: string;      // Opcional: override del número remitente
  appointmentId: string;    // Para logging y auditoría
}
```

---

## Fase 1 – Análisis de Requisitos {#fase-1}

El flujo actual depende de un webhook externo en n8n. El objetivo es reemplazar esa dependencia con lógica nativa en el backend de Next.js, ganando control total, mejor observabilidad y menos puntos de falla externos.

### Funcionalidades requeridas

| Área | Detalle |
|---|---|
| **Input** | Datos del turno: cliente, profesional, servicio, fecha/hora, teléfono |
| **Formateo de teléfono** | Limpieza a formato internacional `54...`, sin duplicar prefijos |
| **Formateo de fecha** | Separar `datetime` ISO en `DD/MM/YYYY` y `HH:mm` para el template |
| **Remitente dinámico** | Usar `shop_sender` si existe; sino, fallback a `YCLOUD_DEFAULT_SENDER` |
| **Output** | Envío exitoso del template `confirmacion_turno` vía API de YCloud |
| **Resiliencia** | Reintentos automáticos ante fallos transitorios de red o timeout |
| **No-blocking** | El fallo de WhatsApp nunca debe cancelar la confirmación del turno |
| **Auditoría** | Registro de cada intento en base de datos con status y error si aplica |
| **Seguridad** | API Key almacenada en variables de entorno, nunca hardcodeada |

### Casos borde de teléfono a contemplar (Argentina)

```
"11 4567-8901"     → "5491145678901"  ✅ número de CABA sin código país
"0115678901"       → "5491145678901"  ✅ con 0 de larga distancia
"+5491145678901"   → "5491145678901"  ✅ ya tiene código, no duplicar
"549XXXXXXXXX"     → "549XXXXXXXXX"   ✅ ya completo, no agregar 54 de nuevo
"15XXXXXXXX"       → error controlado  ⚠️  número local sin área, loggear y continuar
```

---

## Fase 2 – Diseño de la Solución {#fase-2}

### Variables de entorno

```bash
# .env.local
YCLOUD_API_KEY="yk_live_xxxxxxxxxxxx"
YCLOUD_DEFAULT_SENDER="5491XXXXXXXXX"  # Número verificado en YCloud
```

### Arquitectura de archivos

```
src/
├── types/
│   └── ycloud.ts                          # ← FASE 0: contratos e interfaces
│
└── lib/
    └── whatsapp/
        ├── whatsappUtils.ts               # Funciones puras: formatPhone, formatDateTime
        ├── ycloudService.ts               # Cliente HTTP de YCloud con reintentos
        └── notificationService.ts         # Orquestador: une utils + service + logs
```

> **Principio clave:** `route.ts` solo llama a `sendAppointmentConfirmation(payload)` y no sabe nada de YCloud, teléfonos ni templates. Eso es responsabilidad exclusiva de `notificationService.ts`.

### Flujo de datos

```
route.ts (webhook)
    │
    │  AppointmentNotificationPayload
    ▼
notificationService.ts  ──→  whatsappUtils.ts  (formatPhone, formatDateTime)
    │                   ──→  ycloudService.ts   (POST a YCloud + reintentos)
    │
    ▼
Supabase logs table  (status: success | failed | retry_pending)
```

---

## Fase 3 – Implementación {#fase-3}

### Paso 1 – Variables de entorno

Añadir al `.env.local`:

```bash
YCLOUD_API_KEY="reemplazar_con_clave_real"
YCLOUD_DEFAULT_SENDER="reemplazar_con_numero_verificado"
```

Validar en `next.config.js` (o al iniciar el servicio) que ambas variables existan:

```typescript
// src/lib/whatsapp/config.ts
export const ycloudConfig = {
  apiKey: process.env.YCLOUD_API_KEY!,
  defaultSender: process.env.YCLOUD_DEFAULT_SENDER!,
  baseUrl: "https://api.ycloud.com/v2",
};

if (!ycloudConfig.apiKey || !ycloudConfig.defaultSender) {
  throw new Error("[YCloud] Variables de entorno YCLOUD_API_KEY y YCLOUD_DEFAULT_SENDER son requeridas.");
}
```

---

### Paso 2 – Utilidades de formateo (`whatsappUtils.ts`)

```typescript
// src/lib/whatsapp/whatsappUtils.ts

/**
 * Normaliza un número de teléfono argentino a formato internacional.
 * Maneja prefijos duplicados, caracteres especiales y números locales.
 */
export function formatPhone(raw: string): string {
  // 1. Eliminar todo lo que no sea dígito
  const digits = raw.replace(/\D/g, "");

  // 2. Casos ya completos
  if (digits.startsWith("549") && digits.length >= 12) return digits;
  if (digits.startsWith("54") && digits.length >= 11) return digits;

  // 3. Número con 0 de larga distancia (ej: "0114567890")
  const withoutLeadingZero = digits.startsWith("0") ? digits.slice(1) : digits;

  // 4. Número local con 15 (ej: "1545678901") — no se puede resolver sin área
  if (withoutLeadingZero.startsWith("15")) {
    throw new Error(`[formatPhone] Número "${raw}" tiene formato local con 15. Se requiere código de área.`);
  }

  // 5. Agregar prefijo de Argentina
  return `54${withoutLeadingZero}`;
}

/**
 * Separa un datetime ISO en fecha y hora formateadas para el template.
 * Input:  "2025-01-15T14:30:00"
 * Output: { date: "15/01/2025", time: "14:30" }
 */
export function formatDateTime(isoString: string): { date: string; time: string } {
  const dt = new Date(isoString);

  if (isNaN(dt.getTime())) {
    throw new Error(`[formatDateTime] Fecha inválida: "${isoString}"`);
  }

  const date = dt.toLocaleDateString("es-AR", {
    day: "2-digit",
    month: "2-digit",
    year: "numeric",
    timeZone: "America/Argentina/Buenos_Aires",
  });

  const time = dt.toLocaleTimeString("es-AR", {
    hour: "2-digit",
    minute: "2-digit",
    hour12: false,
    timeZone: "America/Argentina/Buenos_Aires",
  });

  return { date, time };
}
```

---

### Paso 3 – Servicio YCloud con reintentos (`ycloudService.ts`)

```typescript
// src/lib/whatsapp/ycloudService.ts
import { YCloudTemplateRequest, YCloudTemplateResponse } from "@/types/ycloud";
import { ycloudConfig } from "./config";

const MAX_RETRIES = 3;
const BASE_DELAY_MS = 1000; // 1s → 2s → 4s (backoff exponencial)

/**
 * Envía un mensaje de template a YCloud.
 * Reintenta hasta MAX_RETRIES veces ante errores transitorios (5xx, timeout).
 * Lanza error en fallos definitivos (4xx) sin reintentar.
 */
export async function sendTemplateMessage(
  payload: YCloudTemplateRequest,
  attempt = 1
): Promise<YCloudTemplateResponse> {
  const url = `${ycloudConfig.baseUrl}/whatsapp/messages`;

  const body = {
    from: payload.from ?? ycloudConfig.defaultSender,
    to: payload.to,
    type: "template",
    template: {
      name: payload.template_name,
      language: { code: "es" },
      components: [
        {
          type: "body",
          parameters: Object.entries(payload.parameters).map(([, value]) => ({
            type: "text",
            text: value,
          })),
        },
      ],
    },
  };

  try {
    const response = await fetch(url, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "X-API-Key": ycloudConfig.apiKey,
      },
      body: JSON.stringify(body),
      signal: AbortSignal.timeout(10_000), // 10s timeout
    });

    // Error definitivo (400, 401, 403, 422) → no reintentar
    if (response.status >= 400 && response.status < 500) {
      const error = await response.json();
      throw new Error(`[YCloud] Error ${response.status}: ${JSON.stringify(error)}`);
    }

    // Error transitorio (500, 502, 503) → reintentar
    if (!response.ok) {
      throw new Error(`[YCloud] Error transitorio ${response.status}`);
    }

    return response.json() as Promise<YCloudTemplateResponse>;

  } catch (error) {
    const isTransient = !(error instanceof Error && error.message.includes("[YCloud] Error 4"));

    if (isTransient && attempt < MAX_RETRIES) {
      const delay = BASE_DELAY_MS * Math.pow(2, attempt - 1);
      console.warn(`[YCloud] Reintento ${attempt}/${MAX_RETRIES} en ${delay}ms...`);
      await new Promise(res => setTimeout(res, delay));
      return sendTemplateMessage(payload, attempt + 1);
    }

    throw error;
  }
}
```

---

### Paso 4 – Orquestador (`notificationService.ts`)

> Este es el componente que faltaba en el plan original. Abstrae toda la lógica y es la única interfaz pública del módulo.

```typescript
// src/lib/whatsapp/notificationService.ts
import { AppointmentNotificationPayload } from "@/types/ycloud";
import { formatPhone, formatDateTime } from "./whatsappUtils";
import { sendTemplateMessage } from "./ycloudService";
import { supabase } from "@/lib/supabaseClient"; // ajustar al path real

export type NotificationStatus = "success" | "failed" | "retry_pending";

/**
 * Punto de entrada único para enviar confirmaciones de turno por WhatsApp.
 * Orquesta formateo, envío y logging. No lanza excepciones (manejo interno).
 */
export async function sendAppointmentConfirmation(
  payload: AppointmentNotificationPayload
): Promise<void> {
  let status: NotificationStatus = "failed";
  let errorMessage: string | null = null;

  try {
    const phone = formatPhone(payload.clientPhone);
    const { date, time } = formatDateTime(payload.datetime);

    await sendTemplateMessage({
      to: phone,
      template_name: "confirmacion_turno",
      from: payload.shopSender,
      parameters: {
        p1_name: payload.clientName,
        p2_service: payload.serviceName,
        p3_date: date,
        p4_time: time,
        p5_prof: payload.professionalName,
      },
    });

    status = "success";
    console.info(`[Notifications] WhatsApp enviado OK → turno ${payload.appointmentId}`);

  } catch (error) {
    errorMessage = error instanceof Error ? error.message : String(error);
    console.error(`[Notifications] Fallo WhatsApp → turno ${payload.appointmentId}:`, errorMessage);
  } finally {
    // Logging de auditoría en Supabase (siempre, sin importar el resultado)
    await supabase.from("notification_logs").insert({
      appointment_id: payload.appointmentId,
      channel: "whatsapp",
      status,
      error_message: errorMessage,
      sent_at: new Date().toISOString(),
    });
  }
}
```

---

### Paso 5 – Integración en el webhook (`route.ts`)

```typescript
// src/app/api/v1/webhooks/trigger/route.ts (fragmento relevante)
import { sendAppointmentConfirmation } from "@/lib/whatsapp/notificationService";

export async function POST(request: Request) {
  // ... lógica existente de creación del turno en BD ...

  const { data: appointment, error } = await supabase
    .from("appointments")
    .insert(appointmentData)
    .select()
    .single();

  if (error || !appointment) {
    return Response.json({ error: "Error al crear turno" }, { status: 500 });
  }

  // ✅ CORRECTO: no-bloqueante. Si WhatsApp falla, el turno ya está confirmado.
  // ❌ EVITAR: await sendAppointmentConfirmation(...) — bloquea y puede fallar la respuesta
  void sendAppointmentConfirmation({
    appointmentId: appointment.id,
    clientName: appointment.client_name,
    clientPhone: appointment.client_phone,
    serviceName: appointment.service_name,
    professionalName: appointment.professional_name,
    datetime: appointment.datetime,
    shopSender: appointment.shop_sender ?? undefined,
  }).catch(err => console.error("[route] Error inesperado en notificación:", err));

  return Response.json({ success: true, appointmentId: appointment.id });
}
```

---

## Fase 4 – Pruebas y Validación {#fase-4}

### Unit Tests – `whatsappUtils.test.ts`

```typescript
describe("formatPhone", () => {
  // Casos básicos
  it("número con código de área y sin prefijo", () => {
    expect(formatPhone("1145678901")).toBe("5491145678901");
  });
  it("número con 0 de larga distancia", () => {
    expect(formatPhone("0111234567890")).toBe("54111234567890");
  });

  // Casos con prefijo ya presente (no duplicar)
  it("ya tiene +54", () => {
    expect(formatPhone("+5491145678901")).toBe("5491145678901");
  });
  it("ya tiene 549 completo", () => {
    expect(formatPhone("5491145678901")).toBe("5491145678901");
  });

  // Caracteres especiales
  it("número con espacios y guiones", () => {
    expect(formatPhone("11 4567-8901")).toBe("5491145678901");
  });

  // Casos borde problemáticos
  it("número con 15 lanza error controlado", () => {
    expect(() => formatPhone("1545678901")).toThrow("formato local con 15");
  });
});

describe("formatDateTime", () => {
  it("formatea correctamente un ISO string", () => {
    expect(formatDateTime("2025-01-15T14:30:00")).toEqual({
      date: "15/01/2025",
      time: "14:30",
    });
  });
  it("lanza error con fecha inválida", () => {
    expect(() => formatDateTime("not-a-date")).toThrow("Fecha inválida");
  });
});
```

### Test de integración

```bash
# Disparar notificación real a número de desarrollo
curl -X POST http://localhost:3000/api/v1/webhooks/trigger \
  -H "Content-Type: application/json" \
  -d '{
    "appointmentId": "test-001",
    "clientName": "Juan Test",
    "clientPhone": "1145678901",
    "serviceName": "Corte de cabello",
    "professionalName": "María",
    "datetime": "2025-01-20T10:00:00"
  }'
```

### Query de auditoría en Supabase

```sql
-- Ver últimas notificaciones con errores
SELECT
  appointment_id,
  status,
  error_message,
  sent_at
FROM notification_logs
WHERE channel = 'whatsapp'
  AND created_at > now() - interval '24 hours'
ORDER BY sent_at DESC;

-- Tasa de éxito por hora
SELECT
  date_trunc('hour', sent_at) AS hora,
  COUNT(*) FILTER (WHERE status = 'success') AS exitosos,
  COUNT(*) FILTER (WHERE status = 'failed') AS fallidos,
  ROUND(
    COUNT(*) FILTER (WHERE status = 'success')::numeric / COUNT(*) * 100, 1
  ) AS tasa_exito_pct
FROM notification_logs
WHERE channel = 'whatsapp'
GROUP BY 1
ORDER BY 1 DESC;
```

---

## Fase 5 – Despliegue y Transición {#fase-5}

### Checklist previo al deploy

```
[ ] YCLOUD_API_KEY cargada en variables de entorno de producción
[ ] YCLOUD_DEFAULT_SENDER configurado con número verificado en YCloud
[ ] Tabla notification_logs creada en Supabase (ver migración abajo)
[ ] Unit tests pasando localmente
[ ] Test de integración exitoso con número de desarrollo
```

### Migración de base de datos

```sql
CREATE TABLE IF NOT EXISTS notification_logs (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  appointment_id TEXT NOT NULL,
  channel       TEXT NOT NULL DEFAULT 'whatsapp',
  status        TEXT NOT NULL CHECK (status IN ('success', 'failed', 'retry_pending')),
  error_message TEXT,
  sent_at       TIMESTAMPTZ NOT NULL,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_notification_logs_appointment ON notification_logs(appointment_id);
CREATE INDEX idx_notification_logs_status ON notification_logs(status, sent_at DESC);
```

### Activación dual (recomendada, 24–48hs)

```
Día 1-2:  n8n activo + nuevo servicio activo en paralelo
           → Comparar logs de ambos sistemas
           → Verificar que no haya mensajes duplicados al cliente

Día 3:    Desactivar webhook de n8n
           → Monitorear notification_logs por 2hs
           → Confirmar tasa de éxito > 95%

Día 4+:   Sistema nativo al 100%
           → Eliminar credenciales de n8n de variables de entorno
```

### Monitoreo post-deploy

Errores a observar en los primeros 7 días:

| Código HTTP | Causa probable | Acción |
|---|---|---|
| `401` | API Key incorrecta o expirada | Verificar `YCLOUD_API_KEY` |
| `400` | Parámetros del template mal formados | Revisar nombres de `p1..p5` |
| `403` | Número remitente no verificado | Verificar `YCLOUD_DEFAULT_SENDER` en el panel de YCloud |
| `429` | Rate limit excedido | Implementar cola con delay |
| Timeout | Conectividad con YCloud | Verificar desde el servidor de producción |

---

## Resumen de Prioridades {#resumen}

| # | Tarea | Impacto | Esfuerzo | Fase |
|---|---|---|---|---|
| 1 | Envío **no-bloqueante** (`void`) | 🔴 Crítico | Muy bajo | 3 |
| 2 | Orquestador `notificationService.ts` | 🔴 Alto | Bajo | 3 |
| 3 | Tipos explícitos `ycloud.ts` | 🟡 Medio | Muy bajo | 0 |
| 4 | Reintentos con backoff exponencial | 🟠 Alto | Medio | 3 |
| 5 | Casos borde `15XXXXXXXX` en formatPhone | 🟠 Alto | Bajo | 3 |
| 6 | Tabla de logs con query de tasa de éxito | 🟡 Medio | Bajo | 5 |

---

*Plan v2.0 · Generado para proyecto `turnero-saas`*
