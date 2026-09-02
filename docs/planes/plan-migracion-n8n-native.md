# Plan de Migración: 3 Flujos n8n → Next.js Nativo
> Proyecto `turnero-saas` · Supabase + YCloud · Versión 1.0

---

## Resumen de los 3 flujos a migrar

| # | Nombre actual en n8n | Función | Nuevo endpoint |
|---|---|---|---|
| 1 | `whatsapp-ai-incoming` (flujo largo) | Bot conversacional con IA: recibe mensaje, interpreta con Gemini, máquina de estados, reserva/cancela turnos | `POST /api/v1/whatsapp/inbound/ai` |
| 2 | `whatsapp-inbound` (flujo corto) | Recibe webhook de YCloud, valida firma, guarda mensaje en DB vía RPC | `POST /api/v1/whatsapp/inbound` |
| 3 | `whatsapp-outbound` (dashboard) | Recibe mensaje del dashboard, lo envía a YCloud, loguea en DB | `POST /api/v1/whatsapp/outbound` |

> **Orden de implementación recomendado:** Flujo 2 → Flujo 3 → Flujo 1.
> El flujo 1 (bot IA) depende de la infraestructura que establecen el 2 y el 3.

---

## Arquitectura de archivos a crear

```
src/
├── types/
│   ├── ycloud.ts                        # Tipos de mensajes YCloud (inbound/outbound)
│   └── whatsapp-session.ts              # Tipos de sesión y estado de la máquina
│
├── lib/
│   └── whatsapp/
│       ├── config.ts                    # Variables de entorno y constantes
│       ├── signatureVerifier.ts         # Verificación HMAC de firma YCloud
│       ├── ycloudClient.ts              # Cliente HTTP para enviar mensajes
│       ├── sessionManager.ts            # get/update/reset session via Supabase RPC
│       ├── shopContext.ts               # get_shop_context via Supabase RPC
│       ├── aiInterpreter.ts             # Llamada a Gemini + parseo de JSON
│       ├── stateMachine.ts              # Lógica de estados (la más compleja)
│       └── messageFormatter.ts          # Armado de respuestas interactive/text
│
└── app/api/v1/whatsapp/
    ├── inbound/
    │   └── route.ts                     # Flujo 2: recibir + guardar en DB
    ├── inbound/ai/
    │   └── route.ts                     # Flujo 1: bot conversacional completo
    └── outbound/
        └── route.ts                     # Flujo 3: enviar desde dashboard
```

---

## Variables de entorno requeridas

```bash
# .env.local
YCLOUD_API_KEY="d0eafac203bf8fe636a0294232f3930e"
YCLOUD_WEBHOOK_SECRET="tu_webhook_secret_de_ycloud"   # para verificar firma
YCLOUD_DEFAULT_SENDER="5493586548065"                  # número verificado

SUPABASE_URL="https://nyoodvdlrrkgibjbxmgj.supabase.co"
SUPABASE_ANON_KEY="eyJhbGci..."                        # la anon key existente

GEMINI_API_KEY="tu_gemini_api_key"                     # para el AI Interpreter
```

---

## Fase 0 – Tipos y contratos

### `src/types/ycloud.ts`

```typescript
// ── Inbound (YCloud → nuestro servidor) ──────────────────────────────────────

export interface YCloudInboundEvent {
  id: string;
  type: "whatsapp.inbound_message.received";
  apiVersion: "v2";
  createTime: string;
  whatsappInboundMessage: YCloudInboundMessage;
}

export interface YCloudInboundMessage {
  id: string;
  wamid: string;
  wabaId: string;
  from: string;                              // "+5493584014857"
  to: string;
  sendTime: string;
  type: "text" | "interactive" | "image" | "audio" | "document";
  text?: { body: string };
  interactive?: {
    type: "list_reply" | "button_reply";
    list_reply?: { id: string; title: string };
    button_reply?: { id: string; title: string };
  };
  customerProfile?: { name: string };
}

// ── Outbound (nuestro servidor → YCloud) ─────────────────────────────────────

export interface YCloudTextMessage {
  messaging_product: "whatsapp";
  to: string;
  from: string;
  type: "text";
  text: { body: string };
}

export interface YCloudInteractiveMessage {
  messaging_product: "whatsapp";
  to: string;
  from: string;
  type: "interactive";
  interactive: YCloudInteractivePayload;
}

export type YCloudInteractivePayload =
  | YCloudListMessage
  | YCloudButtonMessage;

export interface YCloudListMessage {
  type: "list";
  header: { type: "text"; text: string };
  body: { text: string };
  footer: { text: string };
  action: {
    button: string;
    sections: Array<{
      title: string;
      rows: Array<{ id: string; title: string; description: string }>;
    }>;
  };
}

export interface YCloudButtonMessage {
  type: "button";
  body: { text: string };
  action: {
    buttons: Array<{ type: "reply"; reply: { id: string; title: string } }>;
  };
}

export type YCloudOutboundMessage = YCloudTextMessage | YCloudInteractiveMessage;
```

### `src/types/whatsapp-session.ts`

```typescript
export type SessionState =
  | "IDLE"
  | "AWAITING_SERVICE"
  | "AWAITING_PROFESSIONAL"
  | "AWAITING_DATE"
  | "AWAITING_TIME"
  | "AWAITING_CANCEL_SELECTION";

export type SessionIntent = "BOOK" | "CANCEL" | "GREET" | "INFO" | "OTHER" | null;

export interface WhatsappSession {
  current_state: SessionState;
  intent: SessionIntent;
  service: string | null;
  professional: string | null;
  preferred_date: string | null;        // "YYYY-MM-DD"
  preferred_time: string | null;        // "HH:MM:SS"
}

export interface ShopContext {
  shop_name: string;
  services: string[];                   // ya spliteados del string "s1,s2,s3"
  professionals: string[];
  sender_phone: string;                 // shop_sender para YCloud
}

export interface AIInterpreterResult {
  intent: "BOOK" | "CANCEL" | "INFO" | "GREET" | "OTHER";
  extracted: {
    service: string | null;
    professional: string | null;
    date: string | null;               // "YYYY-MM-DD"
    time: string | null;               // "HH:MM:SS"
  };
  naturalResponse: string;
}

export interface StateMachineInput {
  userMessage: string;
  userName: string;
  sender: string;
  shopContext: ShopContext;
  session: WhatsappSession;
  aiResult: AIInterpreterResult;
  todayDate: string;                   // "YYYY-MM-DD"
}

export interface StateMachineOutput {
  routeTo: 0 | 1 | 2 | 3;
  // 0 = respuesta directa | 1 = fetch horarios | 2 = confirmar reserva | 3 = cancelar
  response: YCloudOutboundMessage | null;
  updatedSession: WhatsappSession;
  context: {
    sender: string;
    userName: string;
    shopName: string;
    service: string | null;
    professional: string | null;
    preferredDate: string | null;
    preferredTime: string | null;
  };
}
```

---

## Flujo 2 – Inbound simple (`/api/v1/whatsapp/inbound`)

**Qué hace:** Recibir evento de YCloud → verificar firma HMAC → guardar en DB.

### `src/lib/whatsapp/signatureVerifier.ts`

```typescript
import { createHmac } from "crypto";

/**
 * Verifica la firma HMAC-SHA256 del webhook de YCloud.
 * Header: "ycloud-signature: t=TIMESTAMP,s=SIGNATURE"
 */
export function verifyYCloudSignature(
  rawBody: string,
  signatureHeader: string,
  secret: string
): boolean {
  try {
    const parts = Object.fromEntries(
      signatureHeader.split(",").map(p => p.split("="))
    );
    const timestamp = parts["t"];
    const receivedSig = parts["s"];

    const payload = `${timestamp}.${rawBody}`;
    const expectedSig = createHmac("sha256", secret)
      .update(payload)
      .digest("hex");

    // Comparación de tiempo constante para evitar timing attacks
    return receivedSig === expectedSig;
  } catch {
    return false;
  }
}
```

### `src/app/api/v1/whatsapp/inbound/route.ts`

```typescript
import { NextRequest, NextResponse } from "next/server";
import { verifyYCloudSignature } from "@/lib/whatsapp/signatureVerifier";
import { supabase } from "@/lib/supabaseClient";
import { YCloudInboundEvent } from "@/types/ycloud";

export async function POST(request: NextRequest): Promise<NextResponse> {
  const rawBody = await request.text();
  const signatureHeader = request.headers.get("ycloud-signature") ?? "";

  // 1. Verificar firma
  const isValid = verifyYCloudSignature(
    rawBody,
    signatureHeader,
    process.env.YCLOUD_WEBHOOK_SECRET!
  );
  if (!isValid) {
    console.warn("[Inbound] Firma inválida rechazada");
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }

  // 2. Parsear evento
  const event: YCloudInboundEvent = JSON.parse(rawBody);
  if (!event.whatsappInboundMessage) {
    return NextResponse.json({ ok: true }); // no es un mensaje, ignorar
  }

  const msg = event.whatsappInboundMessage;

  // 3. Guardar en DB vía RPC
  const { error } = await supabase.rpc("handle_inbound_message", {
    p_phone: msg.from,
    p_name: msg.customerProfile?.name ?? "Unknown",
    p_content: msg.text?.body ?? extractInteractiveText(msg),
    p_y_id: msg.id,
  });

  if (error) {
    console.error("[Inbound] Error al guardar en DB:", error.message);
    return NextResponse.json({ error: "DB Error" }, { status: 500 });
  }

  return NextResponse.json({ ok: true });
}

function extractInteractiveText(msg: YCloudInboundEvent["whatsappInboundMessage"]): string {
  if (!msg.interactive) return "";
  return (
    msg.interactive.list_reply?.id ??
    msg.interactive.button_reply?.id ??
    ""
  );
}
```

---

## Flujo 3 – Outbound desde dashboard (`/api/v1/whatsapp/outbound`)

**Qué hace:** Dashboard llama a este endpoint → envía mensaje a YCloud → loguea en DB.

### `src/lib/whatsapp/ycloudClient.ts`

```typescript
import { YCloudOutboundMessage } from "@/types/ycloud";

const YCLOUD_BASE = "https://api.ycloud.com/v2";

export async function sendWhatsAppMessage(
  payload: YCloudOutboundMessage
): Promise<{ id: string; status: string }> {
  const response = await fetch(`${YCLOUD_BASE}/whatsapp/messages`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "X-API-Key": process.env.YCLOUD_API_KEY!,
    },
    body: JSON.stringify(payload),
    signal: AbortSignal.timeout(10_000),
  });

  if (!response.ok) {
    const err = await response.text();
    throw new Error(`[YCloud] HTTP ${response.status}: ${err}`);
  }

  return response.json();
}
```

### `src/app/api/v1/whatsapp/outbound/route.ts`

```typescript
import { NextRequest, NextResponse } from "next/server";
import { sendWhatsAppMessage } from "@/lib/whatsapp/ycloudClient";
import { supabase } from "@/lib/supabaseClient";

interface OutboundRequest {
  contact_id: string;
  recipient: string;   // número destino
  message: string;
}

export async function POST(request: NextRequest): Promise<NextResponse> {
  const body: OutboundRequest = await request.json();

  // 1. Enviar mensaje a YCloud
  let yCloudResponse: { id: string; status: string };
  try {
    yCloudResponse = await sendWhatsAppMessage({
      messaging_product: "whatsapp",
      to: body.recipient,
      from: process.env.YCLOUD_DEFAULT_SENDER!,
      type: "text",
      text: { body: body.message },
    });
  } catch (error) {
    console.error("[Outbound] Error enviando a YCloud:", error);
    return NextResponse.json({ error: "YCloud send failed" }, { status: 502 });
  }

  // 2. Loguear en DB vía RPC (no-bloqueante para la respuesta)
  void supabase.rpc("handle_outbound_message", {
    p_contact_id: body.contact_id,
    p_content: body.message,
    p_y_id: yCloudResponse.id,
  }).then(({ error }) => {
    if (error) console.error("[Outbound] Error logueando en DB:", error.message);
  });

  return NextResponse.json({ ok: true, messageId: yCloudResponse.id });
}
```

---

## Flujo 1 – Bot IA conversacional (`/api/v1/whatsapp/inbound/ai`)

Este es el flujo más complejo. Se desglosa en módulos independientes.

### `src/lib/whatsapp/shopContext.ts`

```typescript
import { supabase } from "@/lib/supabaseClient";
import { ShopContext } from "@/types/whatsapp-session";

export async function getShopContext(
  slug: string,
  phone: string
): Promise<ShopContext> {
  const { data, error } = await supabase.rpc("get_shop_context", {
    p_slug: slug,
    p_phone: phone,
  });

  if (error || !data) throw new Error(`[ShopContext] ${error?.message}`);
  const ctx = Array.isArray(data) ? data[0] : data;

  return {
    shop_name: ctx.shop_name ?? "Peluquería",
    services: (ctx.services ?? "").split(",").map((s: string) => s.trim()).filter(Boolean),
    professionals: (ctx.professionals ?? "").split(",").map((s: string) => s.trim()).filter(Boolean),
    sender_phone: process.env.YCLOUD_DEFAULT_SENDER!,
  };
}
```

### `src/lib/whatsapp/sessionManager.ts`

```typescript
import { supabase } from "@/lib/supabaseClient";
import { WhatsappSession } from "@/types/whatsapp-session";

export async function getOrCreateSession(phone: string): Promise<WhatsappSession> {
  const { data, error } = await supabase.rpc("get_or_create_session", { p_phone: phone });
  if (error) throw new Error(`[Session] get: ${error.message}`);
  const raw = Array.isArray(data) ? data[0] : data;
  return {
    current_state: raw?.current_state ?? "IDLE",
    intent: raw?.intent || null,
    service: raw?.service || null,
    professional: raw?.professional || null,
    preferred_date: raw?.preferred_date || null,
    preferred_time: raw?.preferred_time || null,
  };
}

export async function updateSession(
  phone: string,
  session: WhatsappSession
): Promise<void> {
  const { error } = await supabase.rpc("update_session", {
    p_phone: phone,
    p_current_state: session.current_state,
    p_intent: session.intent ?? "",
    p_service: session.service ?? "",
    p_professional: session.professional ?? "",
    p_preferred_date: session.preferred_date ?? "",
    p_preferred_time: session.preferred_time ?? "",
  });
  if (error) console.error("[Session] update:", error.message);
}

export async function resetSession(phone: string): Promise<void> {
  const { error } = await supabase.rpc("reset_session", { p_phone: phone });
  if (error) console.error("[Session] reset:", error.message);
}
```

### `src/lib/whatsapp/aiInterpreter.ts`

> Reemplaza el nodo "AI Interpreter" + "Google Gemini Chat Model" de n8n.

```typescript
import { AIInterpreterResult } from "@/types/whatsapp-session";

const SYSTEM_PROMPT = `Sos un INTÉRPRETE de mensajes para un bot de turnos. Analizá el mensaje y extraé información.

RESPONDÉ SOLO JSON (sin markdown):
{
  "intent": "BOOK" | "CANCEL" | "INFO" | "GREET" | "OTHER",
  "extracted": {
    "service": "nombre exacto del servicio o null",
    "professional": "nombre exacto del profesional o null",
    "date": "YYYY-MM-DD o null",
    "time": "HH:MM:SS o null"
  },
  "naturalResponse": "respuesta amigable corta"
}

REGLAS CRÍTICAS:
1. Si el usuario dice "cancelar" o "quiero cancelar", intent=CANCEL.
2. Si dice "reservar" u "otro turno" o "nueva reserva", intent=BOOK (empezar de nuevo).
3. Si el mensaje es una selección de lista exacta, extrae ese valor.
4. naturalResponse: máximo 1 emoji.

SOLO JSON.`;

interface InterpreterInput {
  userMessage: string;
  shopName: string;
  services: string[];
  professionals: string[];
  currentState: string;
  currentIntent: string | null;
  sessionService: string | null;
  sessionProfessional: string | null;
  sessionDate: string | null;
  todayDate: string;
}

export async function interpretMessage(
  input: InterpreterInput
): Promise<AIInterpreterResult> {
  const userPrompt = `MENSAJE DEL USUARIO: ${input.userMessage}

CONTEXTO:
- Peluquería: ${input.shopName}
- Servicios: ${input.services.join(", ")}
- Profesionales: ${input.professionals.join(", ")}
- Fecha hoy: ${input.todayDate}

ESTADO ACTUAL:
- Estado: ${input.currentState}
- Intención: ${input.currentIntent ?? "Ninguna"}
- Servicio elegido: ${input.sessionService ?? "Ninguno"}
- Profesional elegido: ${input.sessionProfessional ?? "Ninguno"}
- Fecha elegida: ${input.sessionDate ?? "Ninguna"}`;

  const response = await fetch("https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "X-goog-api-key": process.env.GEMINI_API_KEY!,
    },
    body: JSON.stringify({
      system_instruction: { parts: [{ text: SYSTEM_PROMPT }] },
      contents: [{ role: "user", parts: [{ text: userPrompt }] }],
      generationConfig: { temperature: 0.3, maxOutputTokens: 500 },
    }),
    signal: AbortSignal.timeout(15_000),
  });

  if (!response.ok) throw new Error(`[AI] Gemini HTTP ${response.status}`);

  const data = await response.json();
  const rawText = data.candidates?.[0]?.content?.parts?.[0]?.text ?? "{}";

  try {
    const match = rawText.match(/\{[\s\S]*\}/);
    return match ? JSON.parse(match[0]) : defaultResult();
  } catch {
    return defaultResult();
  }
}

function defaultResult(): AIInterpreterResult {
  return { intent: "OTHER", extracted: { service: null, professional: null, date: null, time: null }, naturalResponse: "" };
}
```

### `src/lib/whatsapp/stateMachine.ts`

> Traducción directa y tipada del nodo "State Machine" de n8n.

```typescript
import { StateMachineInput, StateMachineOutput, WhatsappSession } from "@/types/whatsapp-session";
import { YCloudOutboundMessage } from "@/types/ycloud";

export function runStateMachine(input: StateMachineInput): StateMachineOutput {
  const { userMessage, userName, sender, shopContext, session, aiResult, todayDate } = input;
  const { services, professionals, shop_name: shopName } = shopContext;
  const msgLower = userMessage.toLowerCase();

  let intent = session.intent;
  let service = session.service;
  let professional = session.professional;
  let preferredDate = session.preferred_date;
  let preferredTime = session.preferred_time;
  let currentState = session.current_state;

  // ── Prioridades de intent ─────────────────────────────────────────────────
  if (msgLower.includes("cancelar") || aiResult.intent === "CANCEL") {
    intent = "CANCEL";
    service = null; professional = null; preferredDate = null; preferredTime = null;

  } else if (
    msgLower.includes("reservar") || msgLower.includes("otro turno") ||
    msgLower.includes("nueva") || (aiResult.intent === "BOOK" && currentState === "IDLE")
  ) {
    intent = "BOOK";
    service = null; professional = null; preferredDate = null; preferredTime = null;

  } else if (aiResult.intent === "GREET" && currentState === "IDLE") {
    intent = "GREET";

  } else if (aiResult.intent === "BOOK" || currentState.startsWith("AWAITING_")) {
    intent = "BOOK";

    // Aplicar extracciones del AI
    if (aiResult.extracted.service)       service = aiResult.extracted.service;
    if (aiResult.extracted.professional)  professional = aiResult.extracted.professional;
    if (aiResult.extracted.date)          preferredDate = aiResult.extracted.date;
    if (aiResult.extracted.time)          preferredTime = aiResult.extracted.time;

    // Matching directo de listas (más confiable que el AI para selecciones)
    for (const s of services) {
      if (userMessage === s || msgLower === s.toLowerCase()) { service = s; break; }
    }
    for (const p of professionals) {
      if (userMessage === p || msgLower === p.toLowerCase()) { professional = p; break; }
    }
    if (/^\d{4}-\d{2}-\d{2}$/.test(userMessage)) preferredDate = userMessage;
    if (/^\d{1,2}:\d{2}(:\d{2})?$/.test(userMessage)) {
      preferredTime = userMessage.length === 5 ? `${userMessage}:00` : userMessage;
    }
  }

  // ── Determinar respuesta y ruta ───────────────────────────────────────────
  let response: YCloudOutboundMessage | null = null;
  let routeTo: 0 | 1 | 2 | 3 = 0;

  const updatedSession: WhatsappSession = {
    current_state: currentState,
    intent, service, professional,
    preferred_date: preferredDate,
    preferred_time: preferredTime,
  };

  if (intent === "CANCEL") {
    updatedSession.current_state = "AWAITING_CANCEL_SELECTION";
    routeTo = 3;

  } else if (intent === "GREET") {
    response = text(sender, aiResult.naturalResponse || `¡Hola! 👋 Bienvenido a *${shopName}*. ¿Querés reservar o cancelar un turno?`);
    updatedSession.current_state = "IDLE";
    updatedSession.intent = null;

  } else if (intent === "BOOK") {
    if (!service) {
      response = list(sender, "💇 Nuestros Servicios", aiResult.naturalResponse || "¿Qué servicio te gustaría? 😊", shopName, "Ver Servicios", "Servicios",
        services.slice(0, 10).map(s => ({ id: s, title: s.slice(0, 24), description: "" }))
      );
      updatedSession.current_state = "AWAITING_SERVICE";

    } else if (!professional) {
      response = list(sender, "💈 Nuestros Profesionales", aiResult.naturalResponse || `Elegiste *${service}*. ¿Con quién querés?`, shopName, "Ver Profesionales", "Profesionales",
        professionals.slice(0, 10).map(p => ({ id: p, title: p.slice(0, 24), description: "" }))
      );
      updatedSession.current_state = "AWAITING_PROFESSIONAL";

    } else if (!preferredDate) {
      response = list(sender, "📅 Elegí una Fecha", aiResult.naturalResponse || `*${service}* con *${professional}*. ¿Qué día?`, shopName, "Ver Fechas", "Próximos días",
        buildDateOptions(todayDate)
      );
      updatedSession.current_state = "AWAITING_DATE";

    } else if (!preferredTime) {
      updatedSession.current_state = "AWAITING_TIME";
      routeTo = 1; // → fetch horarios disponibles

    } else {
      routeTo = 2; // → confirmar reserva
    }

  } else {
    response = text(sender, aiResult.naturalResponse || `¡Hola! 👋 Puedo ayudarte a:\n\n• *Reservar* un turno\n• *Cancelar* un turno\n\n¿Qué te gustaría hacer?`);
    updatedSession.current_state = "IDLE";
  }

  return {
    routeTo,
    response,
    updatedSession,
    context: { sender, userName, shopName, service, professional, preferredDate, preferredTime },
  };
}

// ── Helpers ───────────────────────────────────────────────────────────────────

function text(to: string, body: string): YCloudOutboundMessage {
  return { messaging_product: "whatsapp", to, from: process.env.YCLOUD_DEFAULT_SENDER!, type: "text", text: { body } };
}

function list(
  to: string, header: string, body: string, footer: string,
  button: string, sectionTitle: string,
  rows: Array<{ id: string; title: string; description: string }>
): YCloudOutboundMessage {
  return {
    messaging_product: "whatsapp", to,
    from: process.env.YCLOUD_DEFAULT_SENDER!,
    type: "interactive",
    interactive: {
      type: "list",
      header: { type: "text", text: header },
      body: { text: body },
      footer: { text: footer },
      action: { button, sections: [{ title: sectionTitle, rows }] },
    },
  };
}

function buildDateOptions(todayDate: string) {
  const days = ["Dom", "Lun", "Mar", "Mié", "Jue", "Vie", "Sáb"];
  const months = ["Ene", "Feb", "Mar", "Abr", "May", "Jun", "Jul", "Ago", "Sep", "Oct", "Nov", "Dic"];
  const pad = (n: number) => String(n).padStart(2, "0");
  const today = new Date(todayDate);
  return Array.from({ length: 7 }, (_, i) => {
    const d = new Date(today);
    d.setDate(d.getDate() + i);
    const iso = `${d.getFullYear()}-${pad(d.getMonth() + 1)}-${pad(d.getDate())}`;
    const label = i === 0 ? "Hoy" : i === 1 ? "Mañana" : days[d.getDay()];
    return { id: iso, title: `${label} ${d.getDate()} ${months[d.getMonth()]}`, description: "" };
  });
}
```

### `src/app/api/v1/whatsapp/inbound/ai/route.ts`

> Orquestador del flujo completo. Une todos los módulos anteriores.

```typescript
import { NextRequest, NextResponse } from "next/server";
import { verifyYCloudSignature } from "@/lib/whatsapp/signatureVerifier";
import { getShopContext } from "@/lib/whatsapp/shopContext";
import { getOrCreateSession, updateSession, resetSession } from "@/lib/whatsapp/sessionManager";
import { interpretMessage } from "@/lib/whatsapp/aiInterpreter";
import { runStateMachine } from "@/lib/whatsapp/stateMachine";
import { sendWhatsAppMessage } from "@/lib/whatsapp/ycloudClient";
import { supabase } from "@/lib/supabaseClient";
import { YCloudInboundEvent } from "@/types/ycloud";

export async function POST(request: NextRequest): Promise<NextResponse> {
  const rawBody = await request.text();

  // 1. Verificar firma
  const sig = request.headers.get("ycloud-signature") ?? "";
  if (!verifyYCloudSignature(rawBody, sig, process.env.YCLOUD_WEBHOOK_SECRET!)) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }

  const event: YCloudInboundEvent = JSON.parse(rawBody);
  if (!event.whatsappInboundMessage) return NextResponse.json({ ok: true });

  const msg = event.whatsappInboundMessage;
  const sender = msg.from;
  const userName = msg.customerProfile?.name ?? "Cliente";

  // Extraer texto del mensaje (text o interactive)
  let userMessage = msg.text?.body ?? "";
  if (!userMessage && msg.interactive) {
    userMessage =
      msg.interactive.list_reply?.id ??
      msg.interactive.button_reply?.id ??
      "Hola";
  }
  userMessage = userMessage || "Hola";

  try {
    // 2. Obtener contexto del negocio y sesión en paralelo
    const [shopContext, session] = await Promise.all([
      getShopContext("demo", sender),
      getOrCreateSession(sender),
    ]);

    const today = new Date();
    const pad = (n: number) => String(n).padStart(2, "0");
    const todayDate = `${today.getFullYear()}-${pad(today.getMonth() + 1)}-${pad(today.getDate())}`;

    // 3. Interpretar con IA
    const aiResult = await interpretMessage({
      userMessage, shopName: shopContext.shop_name,
      services: shopContext.services, professionals: shopContext.professionals,
      currentState: session.current_state, currentIntent: session.intent,
      sessionService: session.service, sessionProfessional: session.professional,
      sessionDate: session.preferred_date, todayDate,
    });

    // 4. Correr máquina de estados
    const smOutput = runStateMachine({
      userMessage, userName, sender, shopContext, session, aiResult, todayDate,
    });

    // 5. Guardar sesión actualizada
    await updateSession(sender, smOutput.updatedSession);

    // 6. Rutear según la acción determinada
    let finalResponse = smOutput.response;

    if (smOutput.routeTo === 1) {
      // Fetch de horarios disponibles
      finalResponse = await fetchTimesResponse(smOutput.context);

    } else if (smOutput.routeTo === 2) {
      // Confirmar reserva
      finalResponse = await bookAndRespond(smOutput.context, sender);
      await resetSession(sender);

    } else if (smOutput.routeTo === 3) {
      // Cancelar: mostrar lista de turnos del usuario
      finalResponse = await fetchAppointmentsResponse(sender, shopContext.shop_name);
    }

    // 7. Enviar respuesta a YCloud
    if (finalResponse) {
      await sendWhatsAppMessage(finalResponse);
    }

    return NextResponse.json({ ok: true });

  } catch (error) {
    console.error("[AI Inbound] Error:", error);
    // Intentar enviar mensaje de error amigable
    try {
      await sendWhatsAppMessage({
        messaging_product: "whatsapp",
        to: sender,
        from: process.env.YCLOUD_DEFAULT_SENDER!,
        type: "text",
        text: { body: "😕 Tuve un problema técnico. Por favor, intentá de nuevo en unos segundos." },
      });
    } catch { /* silenciar */ }
    return NextResponse.json({ error: "Internal error" }, { status: 500 });
  }
}

// ── Helpers de ruteo ──────────────────────────────────────────────────────────

async function fetchTimesResponse(ctx: ReturnType<typeof runStateMachine>["context"]) {
  const { data } = await supabase.rpc("get_available_slots", {
    p_date: ctx.preferredDate,
    p_professional_name: ctx.professional,
    p_shop_slug: "demo",
    p_service_name: ctx.service,
  });

  const slots: { start_time?: string; time?: string; slot?: string }[] = Array.isArray(data) ? data : [];

  if (slots.length === 0) {
    return buildText(ctx.sender, "😕 No hay horarios disponibles para esa fecha. ¿Probamos otro día?");
  }

  const rows = slots.slice(0, 10).map((slot, i) => {
    const raw = slot.start_time ?? slot.time ?? slot.slot ?? "";
    const time = typeof raw === "string" && raw.includes("T")
      ? raw.substring(11, 16)
      : String(raw).substring(0, 5) || `Horario ${i + 1}`;
    return { id: `${time}:00`, title: time, description: "" };
  });

  const [y, m, d] = (ctx.preferredDate ?? "").split("-");
  const formattedDate = `${d}/${m}`;

  return buildList(ctx.sender, "🕐 Horarios Disponibles",
    `📅 *${formattedDate}* con *${ctx.professional}*\n\n¿A qué hora te viene bien?`,
    ctx.shopName, "Ver Horarios", "Horarios", rows
  );
}

async function bookAndRespond(ctx: ReturnType<typeof runStateMachine>["context"], sender: string) {
  const { data } = await supabase.rpc("book_appointment_by_name", {
    p_customer_name: ctx.userName,
    p_customer_phone: sender,
    p_date: ctx.preferredDate,
    p_time: ctx.preferredTime,
    p_professional_name: ctx.professional,
  });

  const result = Array.isArray(data) ? data[0] : data;

  if (result?.success) {
    const [y, m, d] = (ctx.preferredDate ?? "").split("-");
    const formattedDate = `${d}/${m}/${y}`;
    const formattedTime = (ctx.preferredTime ?? "").substring(0, 5);
    return buildText(ctx.sender,
      `✅ *¡Reserva Confirmada!*\n\n💇 *Servicio:* ${ctx.service}\n💈 *Profesional:* ${ctx.professional}\n📅 *Fecha:* ${formattedDate} a las ${formattedTime}\n\n📍 *Lugar:* ${ctx.shopName}\n\n¡Te esperamos! 🎉\n\n_Escribí "reservar" para hacer otra reserva._`
    );
  }

  const errorMsg = result?.error ?? "";
  const isDuplicate = errorMsg.includes("idx_unique_active_appointment") || errorMsg.includes("duplicate key");
  return buildText(ctx.sender,
    isDuplicate
      ? `⚠️ Ya hay un turno en ese horario con ${ctx.professional}.\n\n¿Querés elegir otro horario o día?`
      : `❌ ${errorMsg || "Hubo un problema."}\n\n¿Probamos otro horario?`
  );
}

async function fetchAppointmentsResponse(sender: string, shopName: string) {
  const { data } = await supabase.rpc("get_client_appointments", { p_phone: sender });
  const appointments = Array.isArray(data) ? data : [];

  if (appointments.length === 0) {
    return buildText(sender, "No tenés turnos pendientes. 😊 ¿Querés reservar uno?");
  }

  const rows = appointments.slice(0, 10).map((apt: Record<string, unknown>) => {
    const date = String(apt.appointment_date ?? (apt.start_time ? String(apt.start_time).substring(0, 10) : ""));
    const time = String(apt.appointment_time ?? (apt.start_time ? String(apt.start_time).substring(11, 16) : ""));
    const [y, m, d] = date.split("-");
    const shortDate = d && m ? `${d}/${m}` : date;
    return {
      id: String(apt.appointment_id ?? apt.id),
      title: `${shortDate} ${time}`.slice(0, 24),
      description: String(apt.service_name ?? "Servicio").slice(0, 72),
    };
  });

  return buildList(sender, "📋 Tus Turnos", "Seleccioná el turno que querés cancelar:", shopName, "Ver Turnos", "Turnos", rows);
}

// ── Constructores de mensajes ──────────────────────────────────────────────────

function buildText(to: string, body: string) {
  return { messaging_product: "whatsapp" as const, to, from: process.env.YCLOUD_DEFAULT_SENDER!, type: "text" as const, text: { body } };
}

function buildList(to: string, header: string, body: string, footer: string, button: string, sectionTitle: string, rows: Array<{ id: string; title: string; description: string }>) {
  return {
    messaging_product: "whatsapp" as const, to,
    from: process.env.YCLOUD_DEFAULT_SENDER!,
    type: "interactive" as const,
    interactive: {
      type: "list" as const,
      header: { type: "text" as const, text: header },
      body: { text: body },
      footer: { text: footer },
      action: { button, sections: [{ title: sectionTitle, rows }] },
    },
  };
}
```

---

## Fase de Pruebas

### Checklist por flujo

```
FLUJO 2 – Inbound simple
[ ] Firma válida → 200 + registro en DB
[ ] Firma inválida → 401 sin registro
[ ] Mensaje tipo "text" → p_content = texto
[ ] Mensaje tipo "interactive list_reply" → p_content = id de la opción
[ ] Evento sin whatsappInboundMessage → 200 silencioso

FLUJO 3 – Outbound
[ ] POST con { contact_id, recipient, message } → mensaje llega al WhatsApp
[ ] Si YCloud falla → 502 con error descriptivo
[ ] El log en DB se guarda aunque la respuesta ya fue enviada

FLUJO 1 – Bot IA
[ ] "Hola" → saludo con menú
[ ] "Quiero reservar" → lista de servicios
[ ] Selección de servicio → lista de profesionales
[ ] Selección de profesional → lista de fechas
[ ] Selección de fecha → lista de horarios (RPC get_available_slots)
[ ] Selección de horario → confirmación (RPC book_appointment_by_name)
[ ] "Cancelar" → lista de turnos (RPC get_client_appointments)
[ ] Mensaje roto/basura → respuesta genérica, no excepción
[ ] Gemini timeout → respuesta de error amigable al usuario
```

### Curl de prueba – Flujo 3

```bash
curl -X POST http://localhost:3000/api/v1/whatsapp/outbound \
  -H "Content-Type: application/json" \
  -d '{
    "contact_id": "uuid-del-contacto",
    "recipient": "5491145678901",
    "message": "Hola, este es un mensaje de prueba desde el dashboard."
  }'
```

---

## Plan de transición desde n8n

```
Día 1:   Deploy de Flujo 2 y Flujo 3 en producción
         → Mantener n8n activo en paralelo
         → Comparar logs de DB: n8n vs nativo

Día 2:   Si logs son consistentes, activar Flujo 1 en paralelo
         → Redirigir 10% del tráfico al endpoint nativo (feature flag o A/B)

Día 3:   100% del tráfico al endpoint nativo
         → Desactivar workflows en n8n (no eliminar aún)

Día 7:   Si no hubo incidentes, eliminar workflows de n8n
         → Eliminar variables de entorno de n8n del servidor
```

---

## Tabla de RPCs de Supabase utilizadas

| RPC | Flujo | Parámetros |
|---|---|---|
| `handle_inbound_message` | 2, 1 | `p_phone`, `p_name`, `p_content`, `p_y_id` |
| `handle_outbound_message` | 3 | `p_contact_id`, `p_content`, `p_y_id` |
| `get_shop_context` | 1 | `p_slug`, `p_phone` |
| `get_or_create_session` | 1 | `p_phone` |
| `update_session` | 1 | `p_phone`, `p_current_state`, `p_intent`, `p_service`, `p_professional`, `p_preferred_date`, `p_preferred_time` |
| `reset_session` | 1 | `p_phone` |
| `get_available_slots` | 1 | `p_date`, `p_professional_name`, `p_shop_slug`, `p_service_name` |
| `book_appointment_by_name` | 1 | `p_customer_name`, `p_customer_phone`, `p_date`, `p_time`, `p_professional_name` |
| `get_client_appointments` | 1 | `p_phone` |

---

*Plan v1.0 · Migración n8n → Next.js nativo · `turnero-saas`*
