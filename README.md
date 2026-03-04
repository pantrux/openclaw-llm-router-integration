# Claw LLM Router Integration

El **Claw LLM Router** es un plugin de OpenClaw diseñado para enrutar dinámicamente las solicitudes de los usuarios hacia diferentes modelos de lenguaje (LLMs) dependiendo de la complejidad del prompt. Esto permite reducir costos de API de manera drástica y acelerar el tiempo de respuesta en tareas simples, reservando los modelos avanzados únicamente para tareas que realmente lo requieren.

## 🎯 Objetivo

Optimizar los costos y tiempos de respuesta mediante un enrutamiento automático e inteligente de los prompts, transparente para el usuario final. El usuario siempre interactúa con el modelo virtual `claw-llm-router/auto`.

## 🚀 Beneficios

- **Reducción de Costos:** Disminuye hasta un 80% los costos de API derivando preguntas y tareas sencillas a modelos económicos.
- **Baja Latencia:** Tareas rápidas son procesadas por modelos ligeros casi de manera instantánea (ej: Gemini 3 Flash).
- **Clasificación Local:** El análisis de los prompts ocurre 100% de manera local mediante un motor de reglas (15 dimensiones evaluadas: presencia de código, longitud, verbos imperativos, heurísticas de razonamiento, etc.). No hay llamadas previas a APIs para clasificar.
- **Fallback Automático:** Si un modelo falla, el sistema intenta la tarea automáticamente con el modelo del Tier superior.

## 🧠 Lógica de Enrutamiento y Tiers

El plugin corre un servidor proxy local (por defecto en el puerto `8401`) que recibe un request en formato OpenAI. Dependiendo de su análisis, asigna un nivel de complejidad (Tier):

1. **SIMPLE:** Búsquedas factuales, traducciones, definiciones cortas, saludos.
2. **MEDIUM:** Snippets de código simples, análisis ligero, resúmenes.
3. **COMPLEX:** Tareas agenticas complejas, análisis multi-archivo, arquitectura.
4. **REASONING:** Demostraciones lógicas y matemáticas complejas, derivaciones profundas paso a paso.

## ⚙️ Proveedores y Modelos (Configuración Actual)

La integración actual de Skynet utiliza el ecosistema de **Google Antigravity**. Se ha configurado el archivo `router-config.json` para mapear los Tiers de la siguiente forma:

| Tier | Modelo Asignado |
|------|-----------------|
| **SIMPLE** | `google-antigravity/gemini-3-flash` |
| **MEDIUM** | `google-antigravity/gemini-3-flash` |
| **COMPLEX** | `google-antigravity/gemini-3.1-pro-high` |
| **REASONING**| `google-antigravity/claude-opus-4-6-thinking` |

## 🔧 Parámetros Técnicos

### 1. openclaw.json
Se ajustan varios puntos críticos para permitir el uso del router:
- **`agents.defaults.model.primary`**: `claw-llm-router/auto`
- **`gateway.http.endpoints.chatCompletions.enabled`**: `true` (Habilita que el proxy se conecte y redirija las peticiones por el Gateway).
- **Base URL Dummy:** Para que los modelos de `google-antigravity` pasen por el Gateway usando credenciales existentes (OAuth/Tokens), en `models.providers.google-antigravity` la variable `baseUrl` debe estar como `"gateway"`. Esto activa el provider fallback de gateway interno.

### 2. Prevención de Bucles (Recursion Hook)
Al usar credenciales tipo OAuth (las cuales el router no puede usar directamente) y siendo el router el modelo principal de OpenClaw, este deriva la solicitud *de vuelta* al Gateway de OpenClaw. 

Para que el Gateway no intente volver a usar el router primario y se arme un bucle infinito, el plugin utiliza un Hook en memoria (memoria temporal indexada por el prompt) que intercepta el evento `before_model_resolve` de OpenClaw. De esta forma, reescribe la petición justo a tiempo e invoca al modelo final (`gemini-3.1-pro-high`, etc.).