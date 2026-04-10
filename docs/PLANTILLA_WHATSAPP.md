# Configuración de Plantilla de WhatsApp (Meta Business Manager)

Para que el sistema de notificaciones funcione, debes crear **exactamente** esta plantilla en tu consola de Meta.

## Datos Principales

| Campo | Valor |
|-------|-------|
| **Nombre de la plantilla** | `confirmacion_turno` |
| **Categoría** | `Utilidad` (Utility) |
| **Idioma** | `Español (Argentina)` (`es_AR`) |

## Contenido del Mensaje

Copia y pega este texto en el cuerpo del mensaje:

```text
Hola {{1}}, te confirmamos tu turno para el día {{2}} a las {{3}} hs.

📍 Servicio: {{4}}
👤 Profesional: {{5}}

¡Te esperamos! Si necesitas cancelar, por favor avísanos con anticipación.
```

## Ejemplo de Variables (Para la vista previa en Facebook)

Facebook te pedirá ejemplos para aprobar la plantilla. Puedes usar estos:

- **{{1}}** (Nombre): `Juan`
- **{{2}}** (Fecha): `28/01/2026`
- **{{3}}** (Hora): `14:30`
- **{{4}}** (Servicio): `Corte de Cabello`
- **{{5}}** (Profesional): `Andrés`

---

> [!NOTE]
> Una vez creada, la revisión de Facebook suele demorar entre 1 a 5 minutos. Cuando el estado cambie a **Activo (Active)**, ya podrás enviar mensajes desde n8n.
