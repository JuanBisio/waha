# Guía: Configurar notificaciones de turno (Email o WhatsApp)

Cada negocio puede elegir cómo les llega la confirmación a sus clientes al reservar un turno:
**por WhatsApp** (via YCloud) o **por Email** (via Resend).

La elección se hace desde el dashboard → **Configuración → Canal de notificaciones**.

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
| `WHATSAPP_BOT_ENABLED` | `true` | Solo si querés activar el bot conversacional |

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

## Bot conversacional (opcional)

El bot permite que los clientes reserven, consulten y cancelen turnos directamente desde WhatsApp, sin necesidad de usar el widget.

Para activarlo:

1. Asegurarse de tener `GEMINI_API_KEY` configurada en Vercel (obtener en [aistudio.google.com](https://aistudio.google.com))
2. Agregar en Vercel: `WHATSAPP_BOT_ENABLED=true`
3. Hacer Redeploy

Para desactivarlo (el bot no responde nada, solo acusa recibo):

- Eliminar o poner `WHATSAPP_BOT_ENABLED=false` en Vercel y redeploy

---

## Resumen de variables de entorno

| Variable | Descripción | Canal |
|---|---|---|
| `RESEND_API_KEY` | Clave de Resend para envío de emails | Email |
| `EMAIL_FROM` | Dirección remitente (ej. `Turnero <info@tudominio.com>`) | Email |
| `EMAIL_TO_OVERRIDE` | Redirige todos los emails a esta dirección (solo pruebas) | Email |
| `YCLOUD_API_KEY` | Clave global de YCloud | WhatsApp |
| `YCLOUD_DEFAULT_SENDER` | Número WhatsApp global del sender | WhatsApp |
| `YCLOUD_WEBHOOK_SECRET` | Secret global para validar webhooks de YCloud | WhatsApp |
| `WHATSAPP_BOT_ENABLED` | `true` para activar el bot conversacional | WhatsApp |
| `GEMINI_API_KEY` | Clave de Google Gemini (requerida por el bot) | WhatsApp bot |
| `NEXT_PUBLIC_SUPABASE_URL` | URL del proyecto Supabase | Ambos |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Clave anon de Supabase | Ambos |
| `SUPABASE_SERVICE_ROLE_KEY` | Clave service role de Supabase | Ambos |
| `NEXT_PUBLIC_TURNSTILE_SITE_KEY` | Site key de Cloudflare Turnstile (CAPTCHA del widget) | Ambos |
| `WEBHOOK_MASTER_SECRET` | Secret para autenticar el endpoint de webhooks internos | Ambos |
