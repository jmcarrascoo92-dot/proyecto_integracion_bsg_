# Cambio de proveedor: Gemini 3.1 Flash-Lite

## Qué cambió

El proyecto reemplaza `langchain-openai` y `ChatOpenAI` por `langchain-google-genai` y `ChatGoogleGenerativeAI`. La arquitectura MCP, las tools SQL, la memoria de corto plazo, Streamlit y Claude Desktop no cambian.

## Variables de entorno

```env
GOOGLE_API_KEY=tu_clave
GEMINI_MODEL=gemini-3.1-flash-lite-preview
```

## Por qué el modelo queda configurable

El nombre de Gemini 3.1 Flash-Lite se usa como preview en este laboratorio. Los identificadores de modelos preview, los límites de uso y las regiones habilitadas pueden cambiar. `GEMINI_MODEL` permite actualizar el valor sin editar código.

## Impacto didáctico

Este cambio permite mostrar que LangChain funciona como capa de abstracción del modelo: el agente continúa usando las mismas tools MCP y la misma memoria; únicamente cambia la implementación del chat model y sus credenciales.
