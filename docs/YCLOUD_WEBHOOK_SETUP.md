# Configuración del Webhook en YCloud

Para que YCloud envíe los mensajes entrantes a tu bot de n8n, necesitás configurar un Webhook.

## Paso 1: Obtener la URL del Webhook de n8n

1.  Abrí n8n y cargá el workflow **"WhatsApp AI Booking Agent"**.
2.  Hacé clic en el nodo **"YCloud Webhook"**.
3.  Copiá la URL que aparece arriba (algo como):
    ```
    https://TU_INSTANCIA_N8N.app.n8n.cloud/webhook/whatsapp-ai-incoming
    ```
    (o si es local: `http://localhost:5678/webhook/whatsapp-ai-incoming`)

## Paso 2: Configurar en YCloud

1.  **Entrá a tu panel de YCloud**: [https://www.ycloud.com/console/](https://www.ycloud.com/console/)

2.  Andá a **WhatsApp** > **Configurations** > **Webhooks**.

3.  En el campo **Webhook URL**, pegá la URL de n8n que copiaste.

4.  En **Events**, asegurate de tener habilitado:
    - `whatsapp.inbound_message.received` (Mensajes entrantes)

5.  Guardá los cambios.

## Paso 3: Activar el Workflow

1.  Volvé a n8n.
2.  **Activá el workflow** con el switch de arriba a la derecha.
3.  Probá enviando un mensaje desde tu WhatsApp personal al número `+5493586548065`.

Si todo está bien, deberías recibir una respuesta del bot.

## Troubleshooting

- **No llega nada a n8n**: Verificá que la URL sea correcta y que n8n esté en modo producción (no "Test").
- **Error 500**: Revisá los logs del workflow en n8n para ver qué nodo falla.
- **El bot no responde**: Confirmá que el número de envío (`from`) esté verificado en YCloud.
