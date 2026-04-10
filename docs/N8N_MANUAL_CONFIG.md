# Guía Definitiva de Configuración Manual n8n

Acá tenés los valores exactos para copiar y pegar en cada uno de los 4 nodos que arrastraste.

## 1. Nodo: Simple Memory
(El que en tu foto aparece como "Simple Memory")
*   **Configuración**: No toques nada. Solo conectalo a la entrada `Memory` del Agente.

## 2. Nodo: Google Gemini Chat Model
*   **Credential**: Seleccioná tu `Google Gemini account`.
*   **Model**:
    *   Si podés elegir de la lista: `gemini-1.5-flash` o `gemini-pro`.
    *   Si tenés que escribir: `gemini-flash-latest`.
*   Conectalo a la entrada `Model` del Agente.

## 3. Nodo: AI Agent
(El cerebro central)

*   **Prompt (User Message)**:
    *   Hacé clic en el engranaje/expression.
    *   Pegá: `{{ $json.message }}`
*   **System Message** (Buscá abajo "Add Option" -> "System Message"):
    *   Pegá este texto:
    ```text
    Eres el asistente de turnos de 'ColdBiz Dev'.
    TU OBJETIVO: Ayudar al cliente a reservar un turno.
    
    REGLAS:
    1. Si piden horarios, usa 'checkAvailability' con la fecha (YYYY-MM-DD).
    2. Si dicen "mañana", calculala (Hoy es: {{ $now.format('yyyy-MM-dd') }}).
    3. Para reservar, usa 'bookAppointment'. Pide Nombre y Servicio.
    4. Sé breve.
    ```

## 4. Nodo: Tool - "Check Availability"
(Arrastrá un "HTTP Request Tool" y llamalo así)

*   **Name**: `checkAvailability`
*   **Description**: `Checks available slots. Input: JSON { "date": "YYYY-MM-DD" }`
*   **Method**: `POST`
*   **URL**: `https://nyoodvdlrrkgibjbxmgj.supabase.co/rest/v1/rpc/get_available_slots`
*   **Authentication**: `Header Auth` (o None y ponés headers manuales).
*   **Headers**:
    *   `apikey`: `TU_ANON_KEY_DE_SUPABASE`
    *   `Authorization`: `Bearer TU_ANON_KEY_DE_SUPABASE`
*   **Body Content Type**: `JSON`
*   **JSON parameters**:
    ```json
    { "p_date": "{{ JSON.parse($json.arguments).date }}" }
    ```

## 5. Nodo: Tool - "Book Appointment"
(Otro "HTTP Request Tool")

*   **Name**: `bookAppointment`
*   **Description**: `Books appointment. Input: JSON { "name": "Juan", "date": "2026-02-01", "time": "10:00:00", "phone": "54..." }`
*   **Method**: `POST`
*   **URL**: `https://nyoodvdlrrkgibjbxmgj.supabase.co/rest/v1/appointments`
*   **Headers**: (Mismos de arriba: `apikey` y `Authorization`)
    *   Extra Header -> `Prefer`: `return=minimal`
*   **Body Content Type**: `JSON`
*   **JSON parameters**:
    ```json
    {
      "customer_name": "{{ JSON.parse($json.arguments).name }}",
      "customer_phone": "{{ JSON.parse($json.arguments).phone }}",
      "appointment_date": "{{ JSON.parse($json.arguments).date }}",
      "appointment_time": "{{ JSON.parse($json.arguments).time }}",
      "start_time": "{{ JSON.parse($json.arguments).date }}T{{ JSON.parse($json.arguments).time }}",
      "shop_id": "TU_SHOP_ID", 
      "service_id": "TU_SERVICE_ID",
      "professional_id": "TU_PROFESSIONAL_ID"
    }
    ```
    *(Tip: Si no tenés IDs a mano, borrá las lineas de shop, service y professional por ahora para testear).*
