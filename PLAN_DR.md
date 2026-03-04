# 🚨 Plan de Validación y Disaster Recovery (DR) - Claw LLM Router

**Estado:** ⏸️ Pendiente de Ejecución
**Objetivo:** Validar la correcta clasificación de los prompts (Tiers) y asegurar que las cadenas de contingencia (Fallbacks) actúen de manera predecible y transparente ante caídas simuladas de los proveedores de LLM.

---

## 1. Topología de Tiers y Cadenas de Fallback (DR)

El sistema está configurado con las siguientes rutas de contingencia (`FALLBACK_CHAIN`):

| Tier Original | Modelo Asignado | Cadena de Rescate (Si el anterior falla) |
|---|---|---|
| **SIMPLE** | `gemini-3-flash` | -> `MEDIUM` -> `COMPLEX` |
| **MEDIUM** | `gemini-3.1-pro-low` | -> `MEDIUM_FB` (`gpt-5.2`) -> `COMPLEX` |
| **COMPLEX** | `gemini-3.1-pro-high` | -> `REASONING` |
| **REASONING** | `gemini-3.1-pro-high` | -> `REASONING_FB1` (`gpt-5.3-codex`) -> `REASONING_FB2` (`claude-opus`) |

---

## 2. Metodología de Simulación de Caídas (Chaos Engineering)

Para simular la caída de un proveedor y forzar la entrada del modelo de *fallback*, utilizaremos una interrupción de capa de configuración:
1. Se editará temporalmente `openclaw.json` para alterar el `baseUrl` o el `model id` del modelo objetivo, provocando un error HTTP `404 Not Found` o un `401 Unauthorized`.
2. Se enviará el prompt mediante `curl` al puerto `8401` (Proxy local del Router).
3. Se auditará la salida en vivo mediante `openclaw logs --follow | grep "claw-llm-router"` comprobando que el motor registre el error y pase al siguiente eslabón.
4. Se restaurará la configuración al finalizar cada bloque.

---

## 3. Casos de Prueba: Validación Nominal (Enrutamiento Base)

Se validará que el clasificador heurístico asigne correctamente el Tier basándose en el análisis de tokens y palabras clave.

*   **Test 1.1 (SIMPLE):**
    *   *Prompt:* `"Hola, dime qué hora es."`
    *   *Resultado Esperado:* Clasificación `SIMPLE` -> Ejecuta `gemini-3-flash`.
*   **Test 1.2 (MEDIUM):**
    *   *Prompt:* `"Resume los beneficios de la programación orientada a objetos en 3 viñetas breves y formato JSON."`
    *   *Resultado Esperado:* Clasificación `MEDIUM` -> Ejecuta `gemini-3.1-pro-low`.
*   **Test 1.3 (COMPLEX):**
    *   *Prompt:* `"Diseña una arquitectura de microservicios en Kubernetes, implementando despliegue continuo."`
    *   *Resultado Esperado:* Clasificación `COMPLEX` -> Ejecuta `gemini-3.1-pro-high`.
*   **Test 1.4 (REASONING):**
    *   *Prompt:* `"I need you to prove Fermat's Last Theorem logically, step by step, using mathematical derivations."`
    *   *Resultado Esperado:* Clasificación `REASONING` -> Ejecuta `gemini-3.1-pro-high`.

---

## 4. Casos de Prueba: Disaster Recovery (Simulación de Fallos)

### Escenario A: Caída de Nivel Básico (MEDIUM)
*   **Acción de Ruptura:** Interrumpir el modelo `gemini-3.1-pro-low` en la configuración.
*   **Prompt Inyectado:** (Prompt de nivel Medium).
*   **Comportamiento Esperado:**
    1. El router clasifica como `MEDIUM`.
    2. Falla la llamada a `gemini-3.1-pro-low`.
    3. El router salta a `MEDIUM_FB` e invoca a `openai-codex/gpt-5.2`.
    4. Respuesta exitosa entregada al usuario indicando el salto en logs.

### Escenario B: Caída Total de Google Antigravity (COMPLEX)
*   **Acción de Ruptura:** Romper el acceso al proveedor completo de `google-antigravity` (afectando a flash, pro-low y pro-high).
*   **Prompt Inyectado:** (Prompt de nivel Complex).
*   **Comportamiento Esperado:**
    1. El router clasifica como `COMPLEX`.
    2. Falla la llamada a `gemini-3.1-pro-high` (`COMPLEX`).
    3. Cae al fallback: `REASONING`.
    4. Falla nuevamente (porque `REASONING` usa `gemini-3.1-pro-high` por defecto).
    5. Cae al fallback `REASONING_FB1` e invoca exitosamente a `openai-codex/gpt-5.3-codex`.

### Escenario C: Evento de Extinción Nivel 2 (Doble Caída en REASONING)
*   **Acción de Ruptura:** Romper acceso a `google-antigravity/gemini-3.1-pro-high` Y romper acceso a `openai-codex/gpt-5.3-codex`.
*   **Prompt Inyectado:** (Prompt de nivel Reasoning).
*   **Comportamiento Esperado:**
    1. El router clasifica `REASONING`.
    2. Falla `gemini-3.1-pro-high`.
    3. Pasa a `REASONING_FB1` (`gpt-5.3-codex`). Falla.
    4. Pasa a la última línea de defensa `REASONING_FB2` (`claude-opus-4-6-thinking`).
    5. La petición se procesa y responde correctamente.

---

## 5. Criterios de Éxito

1.  **Cero Drop de Mensajes:** Ningún mensaje debe devolver error al usuario final, a menos que toda la cadena de fallbacks (incluyendo Claude Opus) esté caída.
2.  **Transparencia de Fallback:** El cambio de modelo debe ocurrir dentro de los parámetros de Timeout (sin colgar la sesión de OpenClaw).
3.  **Trazabilidad:** Cada salto (fallback) debe quedar explícitamente registrado en los logs con la etiqueta `[claw-llm-router] fallback: tier=... error=...`.