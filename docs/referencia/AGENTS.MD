AGENTS.MD - Especificaciones Técnicas del Proyecto "Turnero SaaS"
1. Visión General
Sistema de gestión de turnos (Turnero) bajo modelo SaaS Multi-tenant. El sistema permite a comercios (enfocado inicialmente en peluquerías) gestionar agendas de múltiples profesionales y ofrecer un widget embebible para reservas de clientes. Incluye integración con n8n para automatización de WhatsApp.

2. Stack Tecnológico
Framework: Next.js 14+ (App Router).

Lenguaje: TypeScript.

Base de Datos: PostgreSQL (vía Supabase).

Seguridad: Row Level Security (RLS) en base de datos.

UI Library: Shadcn UI + Tailwind CSS.

Iconografía: Lucide React.

Validación: Zod + React Hook Form.

3. Arquitectura y Seguridad
Multi-tenancy: Aislamiento total por shop_id. Ninguna peluquería puede acceder a datos de otra.

CORS: Las API Routes deben validar el dominio del cliente registrado en la tabla shops.

Autenticación:

Dashboard: Supabase Auth.

n8n / API Externa: secret_api_key por comercio en los headers.

Widget Público: public_key para lectura de disponibilidad y creación de turnos pendiente.

4. Esquema de Base de Datos (SQL)
CREATE TYPE appointment_status AS ENUM ('pendiente', 'confirmado', 'cancelado', 'completado', 'no_asistio');

CREATE TABLE shops (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  name TEXT NOT NULL,
  slug TEXT UNIQUE NOT NULL,
  domain TEXT,
  api_key_n8n UUID DEFAULT uuid_generate_v4(),
  public_key UUID DEFAULT uuid_generate_v4(),
  theme JSONB DEFAULT '{"primary_color": "#2563eb", "dark_mode": true}'::jsonb,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE professionals (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  shop_id UUID REFERENCES shops(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  buffer_time_minutes INTEGER DEFAULT 10,
  is_active BOOLEAN DEFAULT true
);

CREATE TABLE services (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  shop_id UUID REFERENCES shops(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  duration_minutes INTEGER NOT NULL,
  price DECIMAL(10,2)
);

CREATE TABLE schedules (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  professional_id UUID REFERENCES professionals(id) ON DELETE CASCADE,
  day_of_week INTEGER CHECK (day_of_week BETWEEN 0 AND 6),
  start_time TIME NOT NULL,
  end_time TIME NOT NULL
);

CREATE TABLE exceptions (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  professional_id UUID REFERENCES professionals(id) ON DELETE CASCADE,
  specific_date DATE NOT NULL,
  start_time TIME, 
  end_time TIME,
  is_blocked BOOLEAN DEFAULT true
);

CREATE TABLE appointments (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  shop_id UUID REFERENCES shops(id) ON DELETE CASCADE,
  professional_id UUID REFERENCES professionals(id) ON DELETE CASCADE,
  service_id UUID REFERENCES services(id) ON DELETE SET NULL,
  start_time TIMESTAMP WITH TIME ZONE NOT NULL,
  end_time TIMESTAMP WITH TIME ZONE NOT NULL,
  customer_name TEXT NOT NULL,
  customer_phone TEXT NOT NULL,
  customer_email TEXT,
  status appointment_status DEFAULT 'pendiente'
);


5. Reglas de Negocio (Lógica del Turnero)
Cálculo de Slots:

Un slot disponible debe durar: service.duration_minutes + professional.buffer_time_minutes.

No se pueden solapar turnos existentes (excepto los cancelado).

Horizonte de Reserva: Solo se muestran turnos para los próximos 30 días.

Antelación: No se permiten reservas para una hora que ya pasó o que está a menos de 1 hora del momento actual.

Cancelaciones: El cliente puede cancelar su turno mediante un link único, lo cual libera el hueco inmediatamente en la base de datos.

Excepciones: Las entradas en la tabla exceptions tienen prioridad sobre los horarios base en schedules.

REQUERIMIENTOS FUNCIONALES (RF)
Gestión de Comercio (Admin)
RF1 - Multi-tenancy: Registro independiente de comercios. Cada uno con su propio subdominio/slug.

RF2 - Gestión de Profesionales: Alta/Baja de profesionales por comercio. Cada profesional tiene su propia agenda.

RF3 - Gestión de Servicios: Definición de servicios con nombre, precio y duración (minutos).

RF4 - Configuración de Disponibilidad: Horario semanal base (ej. Lunes a Viernes 09:00-18:00).

RF5 - Excepciones y Bloqueos: Interfaz para que el admin haga clic en un horario y lo bloquee (Manual override).

RF6 - Dashboard Visual: Vista mensual del calendario con desglose diario en lista de horarios.

Reserva de Clientes (Público)
RF7 - Widget Embebible: Botón flotante que abre un modal responsive.

RF8 - Flujo de Reserva: Selección de Servicio -> Profesional -> Fecha -> Horario -> Datos personales.

RF9 - Validación Anti-Spam: Implementación obligatoria de CAPTCHA en el formulario.

RF10 - Cancelación: Link único enviado al cliente para cancelar el turno sin login, liberando el slot.


REQUERIMIENTOS NO FUNCIONALES (RNF)
RNF1 - Seguridad de Datos: Aislamiento estricto vía PostgreSQL Row Level Security (RLS).

RNF2 - Escalabilidad: Backend Serverless (Next.js) preparado para picos de tráfico en reservas.

RNF3 - Performance: El cálculo de disponibilidad debe ocurrir en <200ms para evitar latencia en el modal.

RNF4 - UI/UX: Interfaz limpia basada en Shadcn UI. Paleta: Azul (#2563eb), Celeste (#38bdf8), Blanco y Negro.

RNF5 - Modo Dark: Soporte total para temas claro/oscuro.

6. Diseño UI/UX (Shadcn UI)
Paleta de Colores:

Principal: Azul (#2563eb).

Acento: Celeste (#38bdf8).

Neutros: Blanco (#ffffff) y Negro (#000000).

Modo Dark: Soportado nativamente mediante clases dark: de Tailwind.

Dashboard Admin:

Vista mensual (Componente Calendar).

Al seleccionar día: Lista de horarios con badges de estado.

Bloqueo manual: Al clickear un horario libre, opción de "Bloquear".

Widget Cliente:

Activado mediante botón flotante.

Abre un modal (Dialog) o panel lateral (Sheet).

Flujo: Servicio -> Profesional -> Día -> Hora -> Formulario + Captcha.

7. API & Integración n8n
GET /api/v1/availability: Devuelve array de strings (ej: ["09:00", "09:40"]).

POST /api/v1/appointments/external: Permite a n8n crear turnos sin captcha (Usa api_key_n8n).

Webhooks de Salida: Al insertar un registro en appointments, realizar un POST a la URL de n8n configurada en shops.config.

9. Backlog de Implementación Priorizado
Setup de Next.js + Supabase Auth + Shadcn UI.

Implementación del algoritmo de cálculo de slots (Logic util).

Dashboard: CRUD de Profesionales y Servicios.

Dashboard: Vista de Agenda con bloqueo manual de horarios.

Widget: Modal de reserva responsive para sitios externos.

API Routes para integración con n8n.

ESPECIFICACIONES DE API PARA N8N
Headers requeridos: Authorization: Bearer <shop_api_key_n8n>.

Payload para Reserva: { professional_id, service_id, start_time, customer_data }.

Seguridad: El endpoint debe saltarse el Captcha si la API Key es válida.

10. FASES DE DESARROLLO (STEP-BY-STEP)
Fase 1: Entorno y Conectividad
Inicializar Next.js 14 (App Router) + Tailwind + Shadcn UI.

Configurar cliente de Supabase y validar conexión.

Crear Middleware para manejar la detección de shop_id basado en el dominio o slug.

Fase 2: El Motor de Disponibilidad (Core)
Desarrollar utilidad getAvailableSlots(date, serviceId, professionalId).

Implementar lógica de cruce: Horario Base vs Appointments existentes vs Exceptions.

Testear que el buffer_time se sume correctamente al final de cada turno.

Fase 3: Dashboard Administrativo
Implementar Auth con Supabase para dueños de comercios.

Crear formularios de configuración de Servicios y Profesionales.

Crear vista de calendario mensual. Al clickear un día, mostrar lista de horarios y permitir el "Toggle" de bloqueo (insertar en exceptions).

Fase 4: Widget Público Embebible
Crear una página widget/[shop_slug] optimizada para ser cargada en un modal/iframe.

Implementar formulario de reserva por pasos con validación de Captcha.

Implementar página de confirmación y link de cancelación.

Fase 5: Integración n8n y Webhooks
Crear API Route POST /api/v1/appointments/external (Protegida por API Key).

Configurar disparadores de Webhooks en Next.js cada vez que una fila en appointments cambie de estado.



PSEUDOCODIGO 
El algoritmo para calcular los huecos (slots) disponibles es el motor de tu aplicación. Como estamos en fase de diseño, vamos a definir la lógica matemática y los pasos secuenciales que tu backend deberá ejecutar cada vez que un cliente elija una fecha.

1. El Concepto del "Bloque Total"

Para que el calendario sea ordenado, no calculamos solo la duración del servicio, sino el bloque completo que ese turno ocupará en la agenda del profesional.

$$BloqueTotal = DuraciónServicio + BufferTime$$

Ejemplo: Si el corte dura 30 min y el peluquero tiene un buffer de 10 min, cada turno ocupa 40 minutos reales en el sistema.

2. Pasos del Algoritmo de Disponibilidad

Cuando el sistema recibe una petición para una fecha y un profesional_id, debe seguir este orden:

Paso 1: Definir el Marco de Tiempo (La Jornada)

Identificar qué día de la semana es la fecha solicitada (0-6).

Consultar la tabla schedules para obtener la hora_inicio y hora_fin de ese profesional.

Validación de Excepciones: Consultar la tabla exceptions. Si existe un registro para ese día con is_blocked = true y sin horas definidas, el resultado es "Día no laborable".

Paso 2: Obtener Ocupaciones Existentes

Traer todos los registros de la tabla appointments para ese profesional en esa fecha que no estén en estado cancelado.

Extraer los rangos de tiempo: [start_time, end_time].

Paso 3: Generar la "Grilla" de Huecos Potenciales

Empezando desde la hora_inicio de la jornada, el sistema genera turnos candidatos sumando el BloqueTotal.

Inicio: 09:00 hs.

Slot 1: 09:00 a 09:40.

Slot 2: 09:40 a 10:20.

...y así hasta llegar a la hora_fin.

Paso 4: El Filtro de Colisiones (Cruce de Datos)

Para cada "Hueco Potencial", el sistema debe verificar:

Superposición con Turnos: ¿El hueco choca con algún turno del Paso 2?

Superposición con Excepciones: ¿El hueco choca con un bloqueo horario manual (ej: almuerzo)?

Margen de Tiempo Real: Si la fecha es hoy, eliminar los huecos cuya hora de inicio ya pasó o están dentro del margen de antelación mínima (ej: no mostrar turnos para dentro de 15 minutos).

Paso 5: Formatear la Respuesta

El resultado final es un array simple de strings que el frontend (Shadcn UI) usará para dibujar los botones:

["09:00", "09:40", "11:00", "11:40", ...]

3. Manejo de la "Ventana de 30 Días"

Esta regla de negocio se aplica en dos niveles:

Frontend: El componente <Calendar /> de Shadcn debe tener la propiedad disabled para cualquier fecha mayor a hoy + 30 días.

API: El endpoint de disponibilidad debe rechazar cualquier petición fuera de ese rango por seguridad.

4. Reglas para el Caso de Uso de n8n

Cuando n8n consulta la API, el flujo es idéntico, pero con una consideración extra:

Sin Captcha: Como la petición viene de un bot de WhatsApp, n8n usará el api_key_n8n para saltarse la validación de spam que sí tiene el widget web.

Reserva Forzada: n8n debe enviar el end_time ya calculado (start_time + duración) para que la base de datos se mantenga íntegra.

Estado del Diseño: Finalizado

Hemos cubierto:

Modelo de Datos (SQL listo).

Arquitectura (SaaS, Next.js, Supabase, n8n).

UI/UX (Shadcn, Dark Mode, Lista de horarios).

Lógica Algorítmica (Cálculo de slots y buffers).