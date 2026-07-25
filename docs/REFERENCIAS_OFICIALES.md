# Referencias oficiales consultadas

- Google Gemini API — documentación y modelos: https://ai.google.dev/gemini-api/docs
- Google AI Studio: https://aistudio.google.com/
- LangChain — integración Google Gemini: https://python.langchain.com/docs/integrations/chat/google_generative_ai/
- LangChain — Short-term memory: https://docs.langchain.com/oss/python/langchain/short-term-memory
- Model Context Protocol — Arquitectura: https://modelcontextprotocol.io/specification/2025-11-25/architecture/index
- Model Context Protocol — Tools: https://modelcontextprotocol.io/specification/2025-11-25/server/tools

El laboratorio usa `langchain-google-genai` y toma el modelo desde `GEMINI_MODEL`. Los modelos en preview, su identificador, disponibilidad regional y sus cuotas pueden cambiar; por eso el nombre se mantiene configurable en `.env`.
