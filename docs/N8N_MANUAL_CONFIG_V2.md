# Guía Definitiva v2 (Con Soporte para Profesionales)

Seguimos usando los mismos 4 nodos, pero actualizamos lo que va adentro para que la IA entienda nombres.

## 1. Nodo: AI Agent (Actualizar System Message)

*   **System Message**: Borrá el anterior y pegá este nuevo que sabe de profesionales:
    ```text
    Eres el asistente virtual de '{{ $json.shop_name }}'.
    TU OBJETIVO: Gestionar turnos.
    
    PROFESIONALES: {{ $json.professionals }}
    
    REGLAS FECHAS (CRÍTICO):
    1. HOY ES: {{ $now.format('yyyy-MM-dd') }}
    2. Si dicen "el sábado" o "el 31", ASUME la fecha futura más cercana. NO preguntes qué mes/año.
    3. CALCULA SIEMPRE el formato YYYY-MM-DD mentalmente antes de llamar a la herramienta.
    
    REGLAS GENERALES:
    1. Pregunta siempre con qué profesional quieren (dales la lista).
    2. Usa 'checkAvailability' y 'bookAppointment' con la fecha ya calculada.
    ```

## 2. Nodo: "Check Availability" (Actualizar JSON)

*   **URL**: (La misma) `.../rpc/get_available_slots`
*   **JSON parameters** (Actualizar):
    *   Ahora le pasamos también el nombre del peluquero.
    ```json
    { 
      "p_date": "{{ $json.date }}",
      "p_professional_name": "{{ $json.professional }}"
    }
    ```
    *(Nota: Si la IA no manda nombre, mandará "undefined" o null, y nuestra base de datos buscará general).*

## 3. Nodo: "Book Appointment" (CAMBIO GRANDE)

*   **URL** (Cambió, ahora usamos la función inteligente): 
    `https://nyoodvdlrrkgibjbxmgj.supabase.co/rest/v1/rpc/book_appointment_by_name`
    *(Fijate que al final dice `/rpc/book_appointment_by_name` en vez de `/appointments`)*.

*   **Headers**: Iguales (`apikey` y `Authorization`).
*   **JSON parameters** (Nuevo formato):
    ```json
    {
      "p_customer_name": "{{ $json.name }}",
      "p_customer_phone": "{{ $json.phone }}",
      "p_date": "{{ $json.date }}",
      "p_time": "{{ $json.time }}",
      "p_professional_name": "{{ $json.professional }}"
    }
    ```

---
**¿Y el Historial de Chat?**
Para guardar el chat, la forma más simple es agregar un nodo "Supabase" DESPUÉS del Agente, que haga un INSERT en la tabla `chat_history`. Pero primero hagamos andar la reserva con profesionales.
