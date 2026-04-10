# Guía Visual de Conexiones n8n

Para que el Agente de IA funcione, tenés que conectar los "cables" de esta manera exacta.

```mermaid
graph LR
    %% Nodos Principales (Flujo de Mensaje)
    WEBHOOK[("⚡ YCloud Webhook")] --> EXTRACT[("{ } Extract Message")]
    EXTRACT -->|Main Input| AGENT("🤖 AI Agent")
    AGENT -->|Main Output| SEND("🌐 Send YCloud Reply")

    %% Nodos de Soporte (Conectan al AI Agent)
    GEMINI("🧠 Google Gemini Chat") -.->|Model Input| AGENT
    MEMORY("🧠 Window Buffer Memory") -.->|Memory Input| AGENT
    
    %% Herramientas (Tools)
    CHECK("🛠️ Check Availability") -.->|Tool Input| AGENT
    BOOK("🛠️ Book Appointment") -.->|Tool Input| AGENT

    %% Estilos
    style AGENT fill:#ff9900,stroke:#333,stroke-width:2px
    style GEMINI fill:#4285f4,color:white
    style CHECK fill:#0f9d58,color:white
    style BOOK fill:#0f9d58,color:white
```

## Instrucciones Paso a Paso

1.  **Flujo Principal (Línea Continua):**
    *   El mensaje entra por el Webhook, pasa por Extract, entra al Agente y sale hacia "Send Reply". Esto es lo normal.

2.  **Las Entradas Especiales del AI Agent (Líneas Punteadas):**
    *   El nodo **AI Agent** tiene 3 entradas especiales (a veces hay que hacer zoom para ver los puntitos grises en el borde izquierdo o inferior del nodo).
    *   **Model**: Arrastrá desde el puntito gris de **Gemini** hasta la entrada "Model" del Agente.
    *   **Memory**: Arrastrá desde **Window Buffer** hasta la entrada "Memory".
    *   **Tool**: Arrastrá desde **Check Availability** hasta la entrada "Tool".
    *   **Tool (2)**: Arrastrá también desde **Book Appointment** hasta la misma entrada "Tool" (se pueden conectar varios a la misma entrada).
