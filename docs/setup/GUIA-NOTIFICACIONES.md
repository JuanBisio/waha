# Guía: Configurar notificaciones de turno (Email o WhatsApp)

Cada negocio puede elegir cómo les llega la confirmación a sus clientes al reservar un turno:
**por WhatsApp** (via YCloud) o **por Email** (via Resend).

La elección se hace desde el dashboard → **Configuración → Canal de notificaciones**.

---

## Modelo de número compartido (default)

Por defecto, **todos los negocios nuevos envían sus confirmaciones y recordatorios desde
un único número de WhatsApp central de la plataforma** — no hace falta que cada negocio
compre y verifique su propio número. El mensaje ya identifica de qué negocio es el turno
(el template incluye el nombre del negocio), así que el cliente sabe perfectamente quién
le está escribiendo aunque el número sea compartido.

- Las credenciales de ese número compartido son las que están cargadas en Vercel como
  `YCLOUD_API_KEY` / `YCLOUD_DEFAULT_SENDER` (ver tabla de variables más abajo).
- Un negocio puede optar por conectar **su propio número** desde
  `/dashboard/[slug]/configuracion` → sección **WhatsApp con YCloud** — esas credenciales,
  si están cargadas, pisan a las del número compartido para ese shop puntual. Ver
  "Agregar un cliente nuevo con WhatsApp (multi-tenant)" más abajo para cuándo tiene
  sentido hacerlo.
- **Limitación importante:** el número compartido es **solo para salida** (confirmaciones
  y recordatorios). YCloud no puede repartir automáticamente los mensajes **entrantes**
  (respuestas de clientes) entre varios negocios que comparten un mismo número/WABA — solo
  puede enrutarlos a un único webhook. Por eso el **Inbox** del dashboard (respuestas de
  clientes) solo funciona para negocios que configuraron su propio número, no para los que
  usan el número compartido.

---

## Opción A — Email (Resend)

### Cuándo usarlo
- El negocio no tiene número de WhatsApp Business
- El negocio prefiere que las confirmaciones lleguen por correo electrónico
- Flujo más sencillo: sin cuentas de Meta, sin templates aprobados

### Pasos

#### 1. Crear cuenta en Resend

1. Ir a [resend.com](https://resend.com) → **Sign up** (plan gratuito: 3.000 emails/mes)
2. En el panel, ir a **API Keys** → **Create API Key**
3. Darle un nombre (ej. `turnero-prod`) y copiar la clave generada (`re_xxxxxxxxxxxxx`)

#### 2. Agregar variables de entorno en Vercel

Ir a **Vercel → tu proyecto → Settings → Environment Variables** y agregar:

| Variable | Valor | Obligatoria |
|---|---|---|
| `RESEND_API_KEY` | `re_xxxxxxxxxxxxx` | Sí |
| `EMAIL_FROM` | `Turnero <info@tudominio.com>` | No (usa `onboarding@resend.dev` por defecto) |
| `EMAIL_TO_OVERRIDE` | `tuemail@gmail.com` | Solo para pruebas |

> **Nota sobre `EMAIL_FROM`:** Por defecto se usa `onboarding@resend.dev`, que funciona pero
> puede terminar en spam. Para usar tu propio dominio debés verificarlo en Resend (Settings → Domains).
> 
> **Nota sobre `EMAIL_TO_OVERRIDE`:** Si esta variable está definida, TODOS los emails de confirmación
> se envían a esa dirección en lugar de al cliente. Útil para probar sin enviarle correos reales a nadie.
> Eliminala cuando quieras activar el envío real.

#### 3. Redeploy

Después de agregar las variables, hacer un **Redeploy** para que tomen efecto:
Vercel → Deployments → el último deployment → botón **Redeploy**.

#### 4. Configurar el canal en el dashboard

1. Ir a `/dashboard/[tu-slug]/configuracion`
2. En **Canal de notificaciones**, hacer clic en **Email**
3. Opcionalmente, ingresar un **Email de respuesta del negocio** (si el cliente responde el correo, llegará ahí)
4. Clic en **Guardar canal**

#### 5. Verificar

1. Ir al widget de reserva: `/widget/[tu-slug]`
2. Completar una reserva hasta el paso 5 — el campo de contacto debe pedir **Email** (no teléfono)
3. Revisar en Vercel Logs que aparezca: `[Email] Confirmación enviada a cliente@email.com`

---

## Opción B — WhatsApp (YCloud)

### Cuándo usarlo
- El negocio tiene un número de WhatsApp Business activo
- Se quiere que la confirmación llegue al WhatsApp del cliente

### Requisitos previos

- Cuenta activa en [ycloud.com](https://ycloud.com)
- Un número de WhatsApp Business verificado en YCloud
- Template de mensaje aprobado por Meta (ver paso 3)

### Pasos

#### 1. Crear cuenta en YCloud

1. Ir a [ycloud.com](https://ycloud.com) → **Sign up**
2. Conectar el número de WhatsApp Business siguiendo el wizard de YCloud
3. Esperar verificación del número (puede demorar 24–48 hs si es nuevo)

#### 2. Obtener credenciales de YCloud

En el panel de YCloud:

- **API Key:** Settings → API → copiar la clave (`yc_xxxxxxxxxxxxx`)
- **Número de WhatsApp:** el número conectado en formato internacional (ej. `+5493581234567`)
- **Webhook Secret:** se configura al crear el webhook (ver paso 4)

#### 3. Aprobar el template de mensaje

El bot usa el template **`aviso_turno_cliente_v1`** con 7 parámetros.

En YCloud → **WhatsApp → Templates** → crear nuevo template:

```
Nombre: aviso_turno_cliente_v1
Categoría: Utility
Idioma: Español

Cuerpo del mensaje:
Hola {{1}}, tu turno en {{2}} está confirmado.
📅 Fecha: {{3}}
🕐 Hora: {{4}}
✂️ Servicio: {{5}}
💈 Profesional: {{6}}
📍 Lugar: {{7}}

¡Te esperamos!
```

> Meta tarda entre 24 hs y 72 hs en aprobar el template. Solo se puede enviar mensajes
> con templates aprobados (estado: **Approved**).

#### 4. Configurar el webhook en YCloud

En YCloud → **Webhooks** → **Create Webhook**:

- **URL:** `https://turnero-saas.vercel.app/api/v1/whatsapp/inbound/[tu-slug]`
- **Secret:** inventar un string aleatorio y copiarlo (se usará en el paso siguiente)
- **Events:** seleccionar `whatsapp.inbound_message.received`

#### 5. Agregar variables de entorno en Vercel

| Variable | Valor | Obligatoria |
|---|---|---|
| `YCLOUD_API_KEY` | `yc_xxxxxxxxxxxxx` | Sí (fallback global) |
| `YCLOUD_DEFAULT_SENDER` | `+5493581234567` | Sí (fallback global) |
| `YCLOUD_WEBHOOK_SECRET` | el secret del paso 4 | Sí (fallback global) |

> Las variables de Vercel son el **fallback global**. Cada shop puede sobreescribirlas
> con sus propias credenciales desde el dashboard (ver paso 6).

Hacer **Redeploy** después de agregar las variables.

#### 6. Configurar credenciales en el dashboard

1. Ir a `/dashboard/[tu-slug]/configuracion`
2. En **Canal de notificaciones**, hacer clic en **WhatsApp** → **Guardar canal**
3. Bajar a la sección **WhatsApp con YCloud**:
   - **Número de WhatsApp del negocio:** el número conectado (ej. `+5493581234567`)
   - **API Key de YCloud:** la clave del paso 2
   - **Webhook Secret:** el secret del paso 4
4. Clic en **Guardar configuración de WhatsApp**

#### 7. Verificar

1. Ir al widget: `/widget/[tu-slug]`
2. Completar una reserva — el paso 5 debe pedir **número de teléfono** (no email)
3. Revisar en Vercel Logs que aparezca: `[Notifications] WhatsApp enviado OK → turno xxxx`
4. Confirmar que el cliente recibe el mensaje en WhatsApp

---

> **Nota:** hubo un bot conversacional (el cliente reservaba/cancelaba charlando por
> WhatsApp) que se sacó del todo — no funcionaba de forma confiable y era mucho lío
> configurar por negocio. Los mensajes entrantes solo quedan guardados en el Inbox del
> dashboard; el único WhatsApp saliente es la notificación de turno de este flujo.

---

## Agregar un cliente nuevo con WhatsApp (multi-tenant)

**Esto es opcional.** Por defecto, un negocio nuevo no necesita nada de esto: ya manda
notificaciones desde el número compartido de la plataforma (ver sección de arriba). Seguí
estos pasos solo si el negocio puntualmente quiere **su propio número** — por ejemplo,
porque necesita el Inbox de respuestas, quiere volumen/marca propia, o no quiere compartir
la reputación de su WABA con otros negocios del SaaS. Meta liga el remitente y los
templates aprobados a un número/WABA puntual, así que un número propio implica su propia
aprobación de template (paso 3). Pasos, en orden:

#### 1. Conseguir la línea

- Tiene que ser un número que pueda recibir el código de verificación (SMS o llamada) y
  que **no esté ya activo en WhatsApp normal o Business** (si lo está, hay que borrar la
  cuenta de WhatsApp de ese número primero, desde el celular donde esté activo).
- Puede ser una SIM física nueva o un número virtual — pero **verificá con YCloud/Meta
  que acepten números VoIP** antes de comprarlo; algunos proveedores VoIP fallan la
  verificación de WhatsApp Business.
- No hace falta que sea un número argentino si el negocio opera en Argentina: el prefijo
  internacional (`+54...`) es lo que se guarda en `whatsapp_number`, no depende de dónde
  esté físicamente la SIM.

#### 2. Conectar el número en YCloud

1. En el panel de YCloud (la cuenta "J2Studio" que ya usás) → sección WhatsApp → **agregar
   un nuevo número/canal** (no crear una cuenta de YCloud nueva, a menos que quieras
   facturarle cada cliente por separado — confirmá esto en el panel de Billing de YCloud,
   no es algo que dependa del código).
2. Completar el wizard de verificación (código por SMS o llamada).
3. Settings → API → confirmá si el número nuevo comparte la misma API Key de la cuenta o
   genera una propia (varía según plan de YCloud) — la necesitás para el paso 5.

#### 3. Aprobar el template de mensaje para ese número

Igual que en la Opción B, paso 3: el template `aviso_turno_cliente_v1` (u otro que uses)
se aprueba **por WABA**, así que aunque el texto sea idéntico al de `perro-estudio`, este
número nuevo necesita su propia aprobación de Meta (24–72 hs).

#### 4. Crear el shop en la app

- El dueño del negocio se registra en `/signup` con el código de invitación
  (`INVITE_CODE` en Vercel, o `ColdBizTech` si no está seteado) y elige un `slug` único
  (ej. `otra-peluqueria`).
- Si preferís cargarlo vos directamente en vez de que el cliente se registre, es la misma
  pantalla — solo necesitás el código de invitación.

#### 5. Configurar el canal en el dashboard del cliente nuevo

En `/dashboard/otra-peluqueria/configuracion`:

1. **Canal de notificaciones** → WhatsApp → Guardar canal.
2. Sección WhatsApp con YCloud:
   - **Número de WhatsApp:** el número del paso 1, formato internacional.
   - **API Key de YCloud:** la del paso 2 (propia o compartida según lo que haya salido).
   - **Webhook Secret:** opcional ahora que no hay bot — solo hace falta si querés que
     las respuestas del cliente aparezcan en el Inbox del dashboard (ver paso 6). Si no
     te interesa eso, dejalo vacío y solo funcionan las notificaciones salientes.

#### 6. (Opcional) Webhook para el Inbox

Si querés que las respuestas de los clientes de este negocio queden guardadas en
`/dashboard/otra-peluqueria/inbox`:

1. YCloud → Webhooks → **Add Endpoint**.
2. URL: `https://turnero-saas.vercel.app/api/v1/whatsapp/inbound/otra-peluqueria`
3. Secret: inventá un string aleatorio (ej. `openssl rand -hex 32`), pegalo ahí.
4. Ese mismo secret va en el campo **Webhook Secret** del paso 5.
5. Events: `whatsapp.inbound_message.received` (es lo único que usa este endpoint;
   no hay procesamiento de estados de entrega).

#### 7. Probar

Igual que la Opción B, paso 7 — reservar desde `/widget/otra-peluqueria` y confirmar que
llega el WhatsApp.

---

## Recordatorios de turno (3hs antes)

Además de la confirmación al reservar, cada turno `confirmado` recibe un recordatorio por
WhatsApp ~3 horas antes de su `start_time`. Es un flujo separado de la confirmación:

- Usa su propio template de Meta (mensaje iniciado por el negocio, fuera de la ventana de
  24hs de una conversación — Meta exige un template aparte, no se puede reusar
  `aviso_turno_cliente_v1` con otro texto).
- Lo dispara un endpoint propio (`GET /api/cron/reminders`), llamado cada 15 minutos por
  un workflow de GitHub Actions (`.github/workflows/appointment-reminders.yml`) — no por
  Vercel Cron, porque el proyecto está en plan Hobby y ese plan solo permite cron 1 vez
  por día.

### 1. Aprobar el template en Meta

En YCloud → **WhatsApp → Templates** → crear nuevo template:

```
Nombre: recordatorio_turno_cliente_v1
Categoría: Utility
Idioma: Español (es_AR)

Cuerpo del mensaje:
Hola {{1}}, te recordamos tu turno en {{2}} dentro de 3 horas.
📅 Fecha: {{3}}
🕐 Hora: {{4}}
✂️ Servicio: {{5}}
💈 Profesional: {{6}}
📍 Lugar: {{7}}

¡Te esperamos!
```

Igual que con el template de confirmación, Meta tarda entre 24hs y 72hs en aprobarlo.

### 2. Variables de entorno en Vercel

| Variable | Valor | Obligatoria |
|---|---|---|
| `CRON_SECRET` | un string aleatorio (ej. `openssl rand -hex 32`) | Sí |
| `YCLOUD_REMINDER_TEMPLATE_NAME` | `recordatorio_turno_cliente_v1` | Solo una vez que Meta lo apruebe — mientras tanto el endpoint simplemente loguea el turno como no enviado, sin errores |

Redeploy después de agregarlas.

### 3. Configurar el disparador en GitHub Actions

En el repo `turnero-saas` (GitHub) → **Settings → Secrets and variables → Actions**:

- **Secrets → New repository secret:** `CRON_SECRET` = el mismo valor que en Vercel.
- **Variables → New repository variable:** `CRON_TARGET_URL` = la URL base de producción
  (ej. `https://turnero-saas.vercel.app`, sin `/` al final).

El workflow corre solo cada 15 minutos, pero también se puede disparar a mano desde
**Actions → Appointment Reminders → Run workflow** para probarlo sin esperar.

### 4. Verificar

1. Insertar (o esperar) un turno `confirmado` con `start_time` ~3hs en el futuro.
2. Disparar el workflow manualmente, o esperar a que corra.
3. Revisar en Supabase → `notification_logs` que aparezca una fila con
   `notification_type = 'reminder'` y `status = 'success'`, y que
   `appointments.reminder_sent_at` haya quedado seteado.
4. Correr el workflow una segunda vez y confirmar que **no** se manda un segundo mensaje
   para el mismo turno (el claim atómico ya lo marcó como enviado).

---

## Resumen de variables de entorno

| Variable | Descripción | Canal |
|---|---|---|
| `RESEND_API_KEY` | Clave de Resend para envío de emails | Email |
| `EMAIL_FROM` | Dirección remitente (ej. `Turnero <info@tudominio.com>`) | Email |
| `EMAIL_TO_OVERRIDE` | Redirige todos los emails a esta dirección (solo pruebas) | Email |
| `YCLOUD_API_KEY` | Credenciales del número de WhatsApp **compartido** (default para todo shop sin credenciales propias) | WhatsApp |
| `YCLOUD_DEFAULT_SENDER` | Número de WhatsApp **compartido** (default para todo shop sin número propio) | WhatsApp |
| `YCLOUD_WEBHOOK_SECRET` | Secret para validar webhooks de YCloud del número compartido | WhatsApp |
| `YCLOUD_REMINDER_TEMPLATE_NAME` | Nombre del template aprobado en Meta para recordatorios (`recordatorio_turno_cliente_v1`) | WhatsApp |
| `CRON_SECRET` | Token que autentica al workflow de GitHub Actions contra `/api/cron/reminders` | WhatsApp |
| `NEXT_PUBLIC_SUPABASE_URL` | URL del proyecto Supabase | Ambos |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Clave anon de Supabase | Ambos |
| `SUPABASE_SERVICE_ROLE_KEY` | Clave service role de Supabase | Ambos |
| `NEXT_PUBLIC_TURNSTILE_SITE_KEY` | Site key de Cloudflare Turnstile (CAPTCHA del widget) | Ambos |
| `WEBHOOK_MASTER_SECRET` | Secret para autenticar el endpoint de webhooks internos | Ambos |
