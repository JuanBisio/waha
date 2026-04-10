# Configuración YCloud (WhatsApp API)

A diferencia de Meta directo, YCloud simplifica la autenticación usando una API Key permanente (mientras no la cambies).

## Credenciales Actuales

- **API Endpoint**: `https://api.ycloud.com/v2/whatsapp/messages`
- **API Key**: `d0eafac203bf8fe636a0294232f3930e`

## Notas Importantes

1.  **Header de Autenticación**: El workflow de n8n usa `X-API-Key` en lugar de `Authorization: Bearer ...`.
2.  **Sender ID**: Por defecto, YCloud suele inferir el remitente de la API Key. Si tenés múltiples números, habrá que agregar `"from": "NUMERO"` en el cuerpo del JSON (nodo HTTP Request).
3.  **Plantilla**: Se sigue usando la misma estructura de Meta (`template`, `components`, etc.) ya que YCloud actúa como proxy transparente.

## Troubleshooting

Si recibís error de permisos o sender no válido:
- Verificá en el panel de YCloud si el número está **Verificado**.
- Confirmá si necesitás especificar el número de salida explícitamente.

> [!WARNING]
> **Error "Forbidden - Phone number ... has not been registered"**
> Esto significa que el número que pusiste en el campo `from` no está vinculado o verificado en **tu cuenta de YCloud**.
> 1. Entrá a [YCloud Console](https://www.ycloud.com/console/).
> 2. Asegurate que el número aparezca en la lista de "Senders" y esté activo.
