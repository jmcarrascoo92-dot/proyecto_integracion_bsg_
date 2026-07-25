# Agente Analista de E-commerce con MCP

## Problema
Este agente está diseñado para un **analista comercial o de negocio** que necesita consultar y analizar datos reales de ventas, comportamiento de clientes y logística de un e-commerce sin necesidad de escribir código SQL.
Resuelve la necesidad de obtener respuestas verificables y métricas exactas basadas en la base de datos de órdenes (30.000 registros). 
Límites: El agente es de **solo lectura**, no puede modificar datos, aprobar órdenes ni ejecutar instrucciones ambiguas que comprometan el sistema. No inventa cifras ni datos que no estén respaldados por el dataset subyacente.

## Arquitectura
```text
Usuario -> Streamlit -> Agente LangChain + Gemini -> MCP (mcp_datos.py) -> SQLite (ecommerce_orders.db)
```
- **Streamlit**: Interfaz web (Frontend) que gestiona el chat, la memoria de sesión temporal (`session_id`) y visualiza las respuestas y trazas de las tools.
- **Agente LangChain + Gemini**: Capa de inteligencia que procesa el lenguaje natural, selecciona las tools adecuadas del MCP y compone la respuesta final con contexto y memoria.
- **MCP de dominio**: Expone herramientas funcionales seguras de lectura mediante el SDK FastMCP.
- **Datos**: Base de datos SQLite generada a partir del dataset real en formato CSV.

## Tools MCP

| Tool | Propósito | Entrada | Salida | Riesgo |
| --- | --- | --- | --- | --- |
| `buscar_clientes` | Encuentra clientes por ID, ubicación, segmento o membresía. | `texto` (str), `limite` (int, opcional) | Lista de clientes con sus IDs y métricas básicas (JSON). | Lectura: Bajo |
| `resumen_cliente` | Resume compras, gasto, utilidad y ticket de un cliente. | `customer_id` (str) | KPIs de rentabilidad y actividad del cliente (JSON). | Lectura: Bajo |
| `perfil_compras_cliente` | Muestra categorías y subcategorías preferidas de un cliente. | `customer_id` (str), `limite` (int, opcional) | Categorías con métricas de compras y unidades (JSON). | Lectura: Bajo |
| `experiencia_cliente` | Evalúa devoluciones, rating, días de entrega y estados. | `customer_id` (str) | Métricas de satisfacción y logística (JSON). | Lectura: Bajo |
| `ventas_por_dimension` | Ranking de ventas y utilidad por país, categoría, segmento, etc. | `dimension` (str), `year` (int, opc), `limite` (int, opc) | Agrupación de ingresos y utilidades por la dimensión indicada (JSON). | Lectura: Bajo |
| `tendencia_ventas` | Resume ventas mensuales, órdenes y ticket promedio. | `year` (int, opc), `country` (str, opc) | Serie de tiempo mensual de KPIs (JSON). | Lectura: Bajo |
| `detalle_orden` | Recupera el detalle completo de una orden específica. | `order_id` (int) | Toda la información transaccional de una orden (JSON). | Lectura: Bajo |

## Memoria
Usamos un identificador único por conversación llamado `session_id` (enlazado al `thread_id` de LangGraph) y una **memoria de corto plazo** gestionada por `InMemorySaver` de LangGraph.
Mantenemos una ventana máxima de N mensajes (configurada por la variable `MEMORY_WINDOW_MESSAGES`, por defecto 8) para limitar el contexto enviado al modelo y evitar sobrepasar el límite de tokens.
**Limitaciones**: La memoria se mantiene activa únicamente mientras el proceso de Streamlit y el agente se encuentren en ejecución. Se reinicia y se pierde al apagar la aplicación o al crear una nueva conversación (nuevo `session_id`).

## Instalación local

### 1. Crear entorno virtual
En Windows PowerShell:
```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
```
En macOS/Linux:
```bash
python -m venv .venv
source .venv/bin/activate
```

### 2. Instalar dependencias
```bash
pip install -r requirements.txt
```

### 3. Configurar variables de entorno (.env)
Crear un archivo `.env` basado en `.env.example`:
```text
GOOGLE_API_KEY="tu_clave_aqui"
GEMINI_MODEL="gemini-3.1-flash-lite-preview"
DATA_MCP_URL="http://127.0.0.1:8000/mcp"
AGENT_MCP_URL="http://127.0.0.1:8001/mcp"
MEMORY_WINDOW_MESSAGES=8
```

### 4. Preparar la Base de Datos SQLite
Para convertir el CSV incluido a la base SQLite requerida:
```bash
python data/import_dataset_to_sqlite.py
```

### 5. Ejecutar MCP y Streamlit
Abre dos terminales con el entorno virtual activado.

**Terminal 1 (Servidor MCP):**
```bash
python mcp_datos.py
```

**Terminal 2 (Interfaz Streamlit):**
```bash
streamlit run app_streamlit.py
```
*(Opcionalmente, puedes ejecutar `mcp_agente.py` si deseas consumir el agente como un servidor MCP externo con Claude Desktop).*

## Despliegue
La aplicación se encuentra desplegada en **Streamlit Community Cloud**, que actúa como plataforma para la interfaz web (*frontend*). Por su parte, el servidor MCP (`mcp_datos.py`) se expone como un servicio HTTP remoto, alojado en la misma infraestructura de **Streamlit Community Cloud**.

Para hacer posible este despliegue fue necesario adaptar el código a las restricciones de ejecución de la plataforma. En particular, se implementó un mecanismo que inicia los servidores en segundo plano mediante `subprocess.Popen`, complementado con una pausa controlada utilizando `time.sleep`, con el fin de proporcionar el tiempo necesario para que los servicios se inicialicen correctamente antes de atender las solicitudes de la aplicación.

La conexión entre la app Streamlit desplegada y el MCP se configura mediante la variable de entorno `DATA_MCP_URL` alojada en la sección "Secrets" de Streamlit Community Cloud. 

## Pruebas
**Escenarios de prueba obligatorios realizados:**
1. **Consulta directa**: 
   - *Pregunta*: "Revisa el detalle de la orden 615717." 
   - *Resultado esperado*: El agente identifica y extrae los detalles precisos (comprador, monto, fecha) mediante la tool `detalle_orden`.
2. **Consulta compuesta**: 
   - *Pregunta*: "Busca clientes Premium en Germany y analiza el perfil de compras del que más gasta."
   - *Resultado esperado*: El agente usa primero `buscar_clientes` y luego, con el ID devuelto, ejecuta `perfil_compras_cliente`.
3. **Referencia con memoria**:
   - *Pregunta 1*: "Busca a un cliente llamado 'John Doe' o similar." 
   - *Pregunta 2*: "¿Cuál es su experiencia de cliente respecto a las devoluciones?"
   - *Resultado esperado*: El agente recuerda el Customer_ID de John Doe hallado en el primer paso y consulta `experiencia_cliente` sin volver a preguntar su ID.
4. **Dato inexistente**:
   - *Pregunta*: "Busca el detalle de la orden 999999999."
   - *Resultado esperado*: La tool no encuentra resultados. El agente comunica clara y amablemente que la orden no existe, sin alucinar datos.
5. **Consulta fuera de alcance**:
   - *Pregunta*: "Elimina la orden 615717" o "¿Cuál es el clima hoy?"
   - *Resultado esperado*: El agente responde educadamente que sus capacidades solo incluyen consultas de analítica de e-commerce sobre ventas y órdenes (solo lectura), negándose a ejecutar la acción.

## Enlaces
- **App Streamlit Pública**: [https://proyectointegracionbsg-jmc.streamlit.app/](https://proyectointegracionbsg-jmc.streamlit.app/)
- **Repositorio**: [https://github.com/jmcarrascoo92-dot/proyecto_integracion_bsg_](https://github.com/jmcarrascoo92-dot/proyecto_integracion_bsg_)