# 🧱 LATAM AI Agents JSON Schema & Pure Output Layer

[![Version](https://img.shields.io/badge/version-1.0.0-blue.svg)]()
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Language](https://img.shields.io/badge/language-ES%20%7C%20EN-green.svg)]()
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)]()
[![Ko-fi](https://img.shields.io/badge/Support-Ko--fi-FF5E5B?logo=ko-fi&logoColor=white)](https://ko-fi.com/yellowtips)

> **ES:** Capa de contexto meta que convierte a un LLM en un motor de datos puro: suprime introducciones ("Aquí tienes tu JSON…"), notas finales y bloques Markdown, y fuerza salidas estrictas en JSON válido (RFC 8259) con códigos de error estandarizados.
> **EN:** A meta context layer that turns an LLM into a pure data engine: suppresses intros ("Here's your JSON…"), trailing notes and Markdown blocks, and enforces strict JSON output (RFC 8259) with standardized error codes.

> 📄 **Versión:** 1.0.0 · **Protocolo:** Open-AI-Context-Standard · **Ámbito:** Global (énfasis LATAM y España) · **Uso:** System prompt injection para n8n, Make, Zapier, scripts Python/Node, agentes custom y cualquier automatización que necesite JSON parseable al 100%.

> 🇪🇸 [Español](#-español) · 🇬🇧 [English](#-english)

---

## 🇪🇸 Español

### 🎯 Propósito del sistema

Si alguna vez has construido una automatización en n8n, Make o Zapier que llama a un LLM y luego hace `JSON.parse()`, conoces el dolor: **la IA devuelve algo como esto**:

> *"¡Claro! Aquí tienes el JSON con la información extraída de la factura:*
> *```json*
> *{ "monto": 1500, "cliente": "..." }*
> *```*
> *Espero que te sea útil. Si necesitas algo más, no dudes en pedírmelo 😊"*

Tu nodo `Parse JSON` explota. La automatización falla. Alguien recibe un email de alerta a las 3am. Y tú vuelves a añadir *"responde solo con JSON, sin explicaciones"* al prompt… que a veces funciona y a veces no.

El problema no es que el modelo no sepa devolver JSON. El problema es que **nada en su entrenamiento lo obliga a hacerlo** y su sesgo natural es ser *conversacional*, *servicial* y *explicativo*. Le falta:

- Un **contrato de output** explícito con inicio y fin definidos (`{` … `}`).
- Un **envelope de respuesta estandarizado** (`status`, `data`, `error`) para que la automatización pueda tomar decisiones sin parsear strings.
- Un **catálogo de códigos de error** (`E001`–`E008`) que reemplacen frases como *"no pude encontrar el correo"*.
- Una **política de fallback** que devuelva error estructurado en lugar de prosa.
- **Conciencia del comportamiento de streaming**: los tokens llegan antes de que el JSON esté cerrado.

Esta capa resuelve exactamente eso. No es un JSON Schema específico de tu dominio: es **el contrato meta** que hace que cualquier LLM respete un formato parseable, con o sin modo JSON nativo.

> 💡 **Contexto LATAM:** herramientas como n8n, Make y Zapier son el stack por defecto de miles de freelancers y agencias hispanohablantes. La mayoría usa ChatGPT Web o modelos locales sin modo JSON nativo, así que esta capa es especialmente útil en la región.

---

### 🧭 Navegación rápida

| Si eres… | Empieza por… |
| :--- | :--- |
| **Builder de automatizaciones** (n8n / Make / Zapier) | [Ejemplos Before / After](#-ejemplos-de-uso-before--after) → [Cómo inyectar](#-cómo-inyectar-esta-capa-de-contexto) |
| **Dev** que integra LLMs vía API en un agente o pipeline | [Contrato JSON](#️-estructura-de-contexto-para-modelos-de-ia) → [Modos JSON nativos](#-modos-json-nativos-vs-esta-capa) |
| **Prompt engineer** que quiere endurecer outputs existentes | [Reglas globales](#️-reglas-globales-de-interpretación-para-la-ia) → [Catálogo de errores](#-catálogo-de-códigos-de-error) |

---

### 🛠️ Reglas globales de interpretación para la IA

Cuando esta capa esté activa, la IA **DEBE** aplicar siempre este protocolo de 5 pasos, en orden:

1. **Aislamiento de cadena.** Iniciar la respuesta exactamente en el carácter `{` y finalizarla en `}`. Nada antes, nada después.
2. **Supresión conversacional.** Prohibido cualquier texto natural antes, después o entre bloques JSON: sin *"Claro"*, sin *"Aquí tienes"*, sin *"Espero que te sirva"*, sin emojis. Prohibido también envolver el JSON en bloques Markdown (```` ```json ````).
3. **Sintaxis estricta RFC 8259.** Comillas dobles en claves y strings, sin comas finales, sin comentarios, sin `NaN`/`Infinity`, escapes Unicode en UTF-8.
4. **Manejo de errores estructurado.** Si faltan datos o el input es ambiguo, devolver `{"status":"error","error":{"code":"E###","message":"..."}}` en lugar de explicaciones en lenguaje natural.
5. **Nunca inventar datos.** Si un campo no está presente en el input, devolver error (`E001` MISSING_FIELD) o `partial` con `missing_fields`. **Nunca** rellenar con `null` o con valores fabricados.

> ⚠️ Si el modelo no puede cumplir el contrato (ej. input vacío), **debe devolver el error estructurado**, no una disculpa en prosa. El cliente que consume el JSON espera un objeto, no una conversación.

---

### 📊 Mapeo de comportamiento del sistema

| Entrada del usuario | ❌ Salida conversacional (rompe automatización) | ✅ Salida requerida por este protocolo | Código de error si aplica |
| :--- | :--- | :--- | :--- |
| Factura con todos los datos | *"¡Claro! Aquí tienes la información extraída: { ... }"* | `{"status":"success","data":{...}}` | — |
| Factura sin email del cliente | *"No pude encontrar el correo del cliente."* | `{"status":"error","error":{"code":"E001","message":"missing_email","field":"email"}}` | E001 |
| Fecha con formato ambiguo (`03/04/2025`) | *"¿A qué te refieres, 3 de abril o 4 de marzo?"* | `{"status":"error","error":{"code":"E004","message":"ambiguous_date","field":"date"}}` | E004 |
| Consulta fuera de dominio | *"Como modelo de lenguaje, no puedo…"* | `{"status":"error","error":{"code":"E005","message":"out_of_scope"}}` | E005 |
| Múltiples facturas en un PDF | *"Encontré 3 facturas: 1) … 2) … 3) …"* | `{"status":"success","data":{"items":[...]}}` | — |
| Tipo de dato incorrecto (monto = `"mil"`) | *"El monto parece ser mil pesos"* | `{"status":"error","error":{"code":"E002","message":"invalid_type","field":"amount"}}` | E002 |
| Input vacío o solo emojis | *"Parece que no me enviaste datos…"* | `{"status":"error","error":{"code":"E008","message":"extraction_failed"}}` | E008 |

> 📌 **Nota técnica:** la razón por la que el envelope `status/data/error` importa es que tu automatización puede hacer *branching* sobre `status` y sobre `error.code` **sin parsear strings en lenguaje natural**. Un `if (error.code === "E001")` es robusto; un `if (response.includes("no pude encontrar"))` es frágil y se rompe con cualquier cambio de tono del modelo.

---

### 🧪 Ejemplos de uso (Before / After)

#### Ejemplo 1 — Extracción de factura en n8n

**Prompt al LLM:** *"Extrae los datos de esta factura: [texto de factura de 3 párrafos con monto, cliente, fecha, folio]."*

**❌ Sin esta capa (output típico que rompe el nodo Parse JSON):**

```
¡Claro! Aquí tienes los datos extraídos de la factura:

{
  "folio": "A-1234",
  "cliente": "ACME S.A. de C.V.",
  "monto": 15000,
  "fecha": "2025-04-03"
}

Espero que te sea útil. Si necesitas algún ajuste, avísame 😊
```

Nodo `Parse JSON` en n8n → **`Unexpected token '¡', "¡Claro! Aquí"... is not valid JSON`**. Automatización rota.

**✅ Con esta capa:**

```json
{"status":"success","data":{"folio":"A-1234","cliente":"ACME S.A. de C.V.","monto":15000,"currency":"MXN","fecha":"2025-04-03"}}
```

Nodo `Parse JSON` → parsea sin error. Automatización continúa. ✅

---

#### Ejemplo 2 — Factura con datos faltantes (Zapier)

**Prompt:** *"Extrae folio, cliente, monto y email de contacto de esta factura."*

**❌ Sin esta capa:**
```
No pude encontrar el correo de contacto en la factura. Los otros datos son: folio A-1234, cliente ACME, monto $15,000. ¿Quieres que busque el email en otro lado?
```

Tu automatización espera `data.email` → `undefined`. El envío de email de confirmación falla silenciosamente. 😱

**✅ Con esta capa:**

```json
{"status":"partial","data":{"folio":"A-1234","cliente":"ACME S.A. de C.V.","monto":15000},"error":{"code":"E001","message":"missing_email","field":"email"},"metadata":{"missing_fields":["email"]}}
```

Ahora tu automatización puede:
- Detectar `status: "partial"`.
- Hacer un `if (error.code === "E001" && error.field === "email")` y disparar un flujo alterno (ej. pedir el email al usuario por Slack).

---

#### Ejemplo 3 — Múltiples items (Make.com)

**Prompt:** *"Extrae todas las facturas del PDF adjunto."*

**✅ Con esta capa (array consistente):**

```json
{"status":"success","data":{"items":[{"folio":"A-1234","monto":15000},{"folio":"A-1235","monto":8500},{"folio":"A-1236","monto":22000}],"count":3}}
```

El módulo `Iterator` de Make procesa cada item sin ambigüedad.

---

### 🗂️ Catálogo de códigos de error

| Código | Nombre | Cuándo se usa | ¿Reintentar? |
| :--- | :--- | :--- | :---: |
| `E001` | `MISSING_REQUIRED_FIELD` | Falta un campo obligatorio en el input | ❌ (requiere input del usuario) |
| `E002` | `INVALID_TYPE` | El dato está pero tiene tipo incorrecto (ej. `"mil"` en vez de `1000`) | ⚠️ (reintentar con prompt más estricto) |
| `E003` | `INVALID_FORMAT` | Formato inválido (email mal formado, fecha no ISO, etc.) | ⚠️ |
| `E004` | `AMBIGUOUS_INPUT` | El input admite múltiples interpretaciones | ✅ (reintentar con contexto extra) |
| `E005` | `OUT_OF_SCOPE` | La consulta está fuera del dominio esperado | ❌ |
| `E006` | `SCHEMA_VIOLATION` | El LLM devolvió JSON que no respeta el schema esperado | ✅ (reintentar con schema explícito) |
| `E007` | `UNKNOWN_INTENT` | No se pudo determinar qué quiere el usuario | ⚠️ |
| `E008` | `EXTRACTION_FAILED` | Input vacío, ilegible o sin datos extraíbles | ❌ |

> 💡 **Buena práctica:** los códigos `E001` y `E005` **no** deben reintentarse automáticamente (el input del usuario no va a cambiar solo). Los códigos `E004` y `E006` sí, con un retry policy de máx. 2 intentos.

---

### ⚙️ Estructura de contexto para modelos de IA

Bloque listo para inyectar como `system prompt`, archivo `.context/` o payload de API. Sintaxis JSON válida y versionada.

```json
{
  "schema_version": "1.0.0",
  "context_id": "latam-ai-agents-json-schema",
  "last_updated": "2026-10-21",
  "locale": ["es-419", "es-ES", "en"],
  "domain": "structured_output_contract",
  "mode": "strict_json_only",

  "output_contract": {
    "format": "JSON_ONLY",
    "standard": "RFC_8259",
    "start_char": "{",
    "end_char": "}",
    "allow_markdown": false,
    "allow_conversational_text": false,
    "allow_trailing_commas": false,
    "allow_comments": false,
    "allow_single_quotes": false,
    "encoding": "utf-8"
  },

  "response_envelope": {
    "required_fields": ["status"],
    "status_values": ["success", "partial", "error"],
    "success_schema": {
      "status": "success",
      "data": "object | array"
    },
    "error_schema": {
      "status": "error",
      "error": {
        "code": "E###",
        "message": "string",
        "field": "string (optional)",
        "hint": "string (optional)"
      }
    },
    "partial_schema": {
      "status": "partial",
      "data": "object | array",
      "error": "see error_schema",
      "metadata": {
        "missing_fields": ["array of field names"]
      }
    }
  },

  "error_codes": {
    "E001": "MISSING_REQUIRED_FIELD",
    "E002": "INVALID_TYPE",
    "E003": "INVALID_FORMAT",
    "E004": "AMBIGUOUS_INPUT",
    "E005": "OUT_OF_SCOPE",
    "E006": "SCHEMA_VIOLATION",
    "E007": "UNKNOWN_INTENT",
    "E008": "EXTRACTION_FAILED"
  },

  "behavior_rules": [
    {
      "id": "R1",
      "priority": "critical",
      "description": "Iniciar la respuesta directamente en '{' y terminar en '}'. Nada antes, nada después."
    },
    {
      "id": "R2",
      "priority": "critical",
      "description": "Suprimir cualquier texto conversacional antes, después o entre bloques JSON."
    },
    {
      "id": "R3",
      "priority": "critical",
      "description": "Suprimir bloques Markdown (```json ... ```). El output debe ser JSON crudo."
    },
    {
      "id": "R4",
      "priority": "critical",
      "description": "Cumplir RFC 8259: comillas dobles, sin comas finales, sin comentarios, sin NaN/Infinity."
    },
    {
      "id": "R5",
      "priority": "high",
      "description": "Si faltan datos, devolver 'status: error' (o 'partial' si hay algunos datos) con el código E### correspondiente."
    },
    {
      "id": "R6",
      "priority": "high",
      "description": "Nunca inventar datos. Si un campo no está en el input, devolver error E001 — no rellenar con null ni con valores fabricados."
    },
    {
      "id": "R7",
      "priority": "medium",
      "description": "Para múltiples items, devolver array en 'data.items' con estructura consistente y un 'count' en metadata."
    }
  ],

  "platform_profiles": {
    "openai_chat_completions": {
      "supports_native_json_mode": true,
      "parameter": "response_format",
      "value": "{ type: 'json_object' }",
      "note": "Combinar con system prompt para garantizar estructura."
    },
    "openai_structured_output": {
      "supports_native_json_mode": true,
      "parameter": "response_format",
      "value": "{ type: 'json_schema', json_schema: { strict: true, schema: {...} } }",
      "note": "Modelos gpt-4o-2024-08-06+ respetan JSON Schema al 100%."
    },
    "anthropic_claude": {
      "supports_native_json_mode": "partial (via tool use)",
      "parameter": "tools + tool_choice",
      "value": "{ name: 'emit_json', input_schema: {...} }",
      "note": "Forzar tool use para obtener JSON estructurado."
    },
    "google_gemini": {
      "supports_native_json_mode": true,
      "parameter": "response_mime_type + response_schema",
      "value": "'application/json'",
      "note": "Soporta JSON Schema nativo en Gemini 1.5+."
    },
    "n8n": {
      "method": "AI Agent node / OpenAI module system message",
      "note": "Añadir un Code node con JSON.parse + try/catch + retry_on_codes."
    },
    "make": {
      "method": "OpenAI (ChatGPT) module system message",
      "note": "Añadir módulo Parse JSON + error handler con Router."
    },
    "zapier": {
      "method": "Webhooks by Zapier + OpenAI system message",
      "note": "Usar Formatter → Text to JSON + Paths para branching por status."
    },
    "ollama": {
      "method": "prompt template + format: json",
      "note": "Ollama soporta 'format: json' que garantiza JSON válido."
    },
    "lm_studio": {
      "method": "prompt template + JSON mode en settings",
      "note": "Activar 'Structured output' si el modelo lo soporta."
    }
  },

  "retry_behavior": {
    "retry_on_codes": ["E004", "E006"],
    "do_not_retry_on_codes": ["E001", "E005", "E008"],
    "max_retries": 2,
    "backoff": "exponential",
    "notes": "E001/E005/E008 requieren cambio en el input del usuario, no en el modelo."
  },

  "streaming_behavior": {
    "partial_json_warning": true,
    "buffer_strategy": "Acumular tokens hasta que JSON.parse() sea válido.",
    "incremental_parsers": ["json-stream", "stream-json", "jsonparse"]
  },

  "fallback_instructions": {
    "trigger": "Input ambiguo, faltan datos críticos, operación fuera de scope, o el modelo devuelve texto conversacional pese a las instrucciones.",
    "action": "Devolver 'status: error' con código E###. NUNCA devolver explicación en lenguaje natural.",
    "disclaimer_template": "Si el LLM no puede cumplir el contrato, debe devolver el error estructurado. El cliente que consume el JSON espera un objeto, no una conversación."
  }
}
```

---

### 🔌 Cómo inyectar esta capa de contexto

#### OpenAI (API) — con JSON mode nativo

```python
import json
from openai import OpenAI

client = OpenAI()
with open("json-schema-rules.json") as f:
    context = json.load(f)

response = client.chat.completions.create(
    model="gpt-4o-2024-08-06",
    response_format={"type": "json_object"},
    messages=[
        {"role": "system", "content": json.dumps(context, ensure_ascii=False)},
        {"role": "user", "content": "Extrae datos de esta factura: [...]"}
    ]
)
data = json.loads(response.choices[0].message.content)
```

#### OpenAI — con JSON Schema structured output (recomendado)

```python
response = client.chat.completions.create(
    model="gpt-4o-2024-08-06",
    response_format={
        "type": "json_schema",
        "json_schema": {
            "name": "invoice_extraction",
            "strict": True,
            "schema": {
                "type": "object",
                "properties": {
                    "status": {"type": "string", "enum": ["success", "partial", "error"]},
                    "data": {"type": "object"},
                    "error": {"type": "object"}
                },
                "required": ["status"],
                "additionalProperties": False
            }
        }
    },
    messages=[
        {"role": "system", "content": json.dumps(context, ensure_ascii=False)},
        {"role": "user", "content": "..."}
    ]
)
```

#### Claude (Anthropic) — vía tool use

```python
response = client.messages.create(
    model="claude-3-5-sonnet-latest",
    system=json.dumps(context, ensure_ascii=False),
    tools=[{
        "name": "emit_json",
        "description": "Emite el JSON estructurado final.",
        "input_schema": {...}
    }],
    tool_choice={"type": "tool", "name": "emit_json"},
    messages=[{"role": "user", "content": "..."}]
)
```

#### Gemini (Google) — con response_mime_type

```python
response = model.generate_content(
    prompt,
    generation_config={
        "response_mime_type": "application/json",
        "response_schema": {...}
    }
)
```

#### n8n

1. Nodo **AI Agent** o **OpenAI** → añade el JSON del contrato en el campo `System Message`.
2. Después, añade un **Code node** con:

```javascript
try {
  return { json: JSON.parse($input.item.json.message.content) };
} catch (e) {
  return { json: { status: "error", error: { code: "E006", message: "parse_failed" } } };
}
```

3. Añade un **Switch** sobre `{{ $json.status }}` para enrutar success / partial / error.

#### Make.com

1. Módulo **OpenAI (ChatGPT) → Create a Completion** → pega el JSON en `System`.
2. Añade **Parse JSON** con el schema del envelope.
3. Añade un **Router** con filtros: `status = success` vs `status = error`.

#### Zapier

1. **Webhooks by Zapier → POST** a OpenAI.
2. **Formatter → Text to JSON** con el schema.
3. **Paths** para branching por `status` y `error.code`.

#### Ollama (local)

```bash
ollama run llama3.1 --format json
```

Combinado con el prompt del contrato en el system message.

---

### 🧬 Modos JSON nativos vs. esta capa

Los proveedores modernos ofrecen modos JSON nativos. ¿Sigue siendo útil esta capa?

| Escenario | Modo nativo | ¿Necesitas esta capa? |
| :--- | :--- | :--- |
| GPT-4o con `response_format: json_schema` | ✅ Garantiza schema | ⚠️ Sí — para el envelope `status/data/error` y códigos E### |
| Claude 3.5 con tool use | ✅ Garantiza estructura | ⚠️ Sí — para el contrato de error y branching |
| Gemini 1.5 con `response_schema` | ✅ Garantiza schema | ⚠️ Sí — para comportamiento consistente |
| ChatGPT Web (sin API) | ❌ No | ✅ **Imprescindible** |
| Ollama / LM Studio (local) | ⚠️ Parcial | ✅ **Recomendado** |
| n8n / Make / Zapier con módulos legacy | ❌ No | ✅ **Imprescindible** |

> 💡 **Regla práctica:** usa siempre esta capa, incluso con modo nativo. El modo nativo garantiza la **sintaxis**; esta capa garantiza la **semántica** (qué hacer cuando falta un dato, cómo estructurar errores, qué código devolver). Se complementan.

---

### 🤖 Compatibilidad

| Plataforma | Método recomendado | Estado |
| :--- | :--- | :---: |
| OpenAI GPT-4o / o1 | `response_format` + system prompt | ✅ |
| Anthropic Claude 3.5+ | `tools` + `tool_choice` | ✅ |
| Google Gemini 1.5+ | `response_mime_type` + `response_schema` | ✅ |
| Llama 3 / Mistral (local, Ollama) | `format: json` + prompt template | ✅ |
| ChatGPT / Claude Web | Copy-paste en system | ✅ |
| n8n | AI Agent node + Code node | ✅ |
| Make.com | OpenAI module + Parse JSON | ✅ |
| Zapier | Webhooks + Formatter | ✅ |
| Cursor / Windsurf | Archivo `.context/` | ✅ |

---

### ☕ Apoya este proyecto de código abierto

Mantengo este repositorio en mi tiempo libre porque sé lo que es pelearse con un `JSON.parse` a las 3am porque el modelo decidió añadir *"¡Claro!"* al inicio. Si esta capa te ahorró horas de debugging o estabilizó tus automatizaciones, puedes invitarme un café:

👉 **[ko-fi.com/yellowtips](https://ko-fi.com/yellowtips)**

No es obligatorio, pero cada aporte mantiene la base actualizada con los cambios de cada proveedor (que no paran de mover parámetros).

---

### 📄 Licencia

**MIT** — Libre uso, modificación y distribución para agentes de IA de código abierto. Ver [`LICENSE`](LICENSE).

> Esta capa de contexto es **orientativa**. Aunque está diseñada para maximizar la consistencia del output, ningún LLM es 100% determinista. Siempre añade validación (`JSON.parse` con try/catch) y manejo de errores en tu pipeline. El autor no se responsabiliza por automatizaciones que fallen por cambios en el comportamiento de los modelos subyacentes.

---

## 🇬🇧 English

### 🎯 Purpose

If you've ever built an automation in n8n, Make or Zapier that calls an LLM and then runs `JSON.parse()`, you know the pain: **the AI returns something like this**:

> *"Sure! Here's the JSON with the invoice data:*
> *```json*
> *{ "amount": 1500, "client": "..." }*
> *```*
> *Hope it helps! Let me know if you need anything else 😊"*

Your `Parse JSON` node explodes. The automation fails. Someone gets an alert email at 3am. And you go back to adding *"respond only with JSON, no explanations"* to the prompt… which sometimes works and sometimes doesn't.

The model doesn't fail because it can't produce JSON. It fails because **nothing in its training forces it to**, and its natural bias is to be *conversational*, *helpful* and *explanatory*. It's missing:

- An explicit **output contract** with defined start and end (`{` … `}`).
- A **standardized response envelope** (`status`, `data`, `error`) so the automation can branch without parsing natural-language strings.
- A **standardized error code catalog** (`E001`–`E008`) replacing phrases like *"couldn't find the email"*.
- A **fallback policy** that returns structured error instead of prose.
- **Awareness of streaming behavior**: tokens arrive before the JSON is closed.

This layer fixes exactly that. It's not a domain-specific JSON Schema: it's the **meta contract** that makes any LLM respect a parseable format, with or without native JSON mode.

> 💡 **LATAM context:** tools like n8n, Make and Zapier are the default stack for thousands of Spanish-speaking freelancers and agencies. Most use ChatGPT Web or local models without native JSON mode, so this layer is especially valuable in the region.

---

### 🛠️ Global interpretation rules

When this layer is active, the AI **MUST** apply this 5-step protocol in order:

1. **Chain isolation.** Start the response exactly with `{` and end with `}`. Nothing before, nothing after.
2. **Conversational suppression.** No natural text before, after or between JSON blocks: no *"Sure"*, no *"Here you go"*, no *"Hope it helps"*, no emojis. No Markdown wrapping either (```` ```json ````).
3. **Strict RFC 8259 syntax.** Double quotes on keys and strings, no trailing commas, no comments, no `NaN`/`Infinity`, UTF-8 escapes.
4. **Structured error handling.** If data is missing or input is ambiguous, return `{"status":"error","error":{"code":"E###","message":"..."}}` instead of natural-language explanations.
5. **Never invent data.** If a field is not present in the input, return an error (`E001` MISSING_FIELD) or `partial` with `missing_fields`. **Never** fill with `null` or fabricated values.

> ⚠️ If the model can't comply (e.g. empty input), **it must return the structured error**, not an apology in prose. The client consuming the JSON expects an object, not a conversation.

---

### 📊 System behavior mapping

Same table as the Spanish section above — copy from there.

---

### 🧪 Examples (Before / After)

#### Example 1 — Invoice extraction in n8n

**Prompt:** *"Extract the data from this invoice: [3-paragraph invoice text with amount, client, date, folio]."*

**❌ Without this layer:**

```
Sure! Here's the extracted data:

{
  "folio": "A-1234",
  "client": "ACME Inc.",
  "amount": 15000,
  "date": "2025-04-03"
}

Hope it helps! Let me know if you need adjustments 😊
```

`Parse JSON` node → **`Unexpected token 'S', "Sure! Here"... is not valid JSON`**. Automation broken.

**✅ With this layer:**

```json
{"status":"success","data":{"folio":"A-1234","client":"ACME Inc.","amount":15000,"currency":"USD","date":"2025-04-03"}}
```

`Parse JSON` node → parses cleanly. Automation continues. ✅

---

#### Example 2 — Invoice with missing fields (Zapier)

**❌ Without this layer:**
```
I couldn't find the contact email on the invoice. The other data is: folio A-1234, client ACME, amount $15,000. Should I look for the email somewhere else?
```

Your automation expects `data.email` → `undefined`. Confirmation email fails silently. 😱

**✅ With this layer:**

```json
{"status":"partial","data":{"folio":"A-1234","client":"ACME Inc.","amount":15000},"error":{"code":"E001","message":"missing_email","field":"email"},"metadata":{"missing_fields":["email"]}}
```

Now your automation can:
- Detect `status: "partial"`.
- `if (error.code === "E001" && error.field === "email")` → trigger alternate flow (e.g. ask user via Slack).

---

#### Example 3 — Multiple items (Make.com)

**✅ With this layer:**

```json
{"status":"success","data":{"items":[{"folio":"A-1234","amount":15000},{"folio":"A-1235","amount":8500},{"folio":"A-1236","amount":22000}],"count":3}}
```

Make's `Iterator` module processes each item without ambiguity.

---

### 🗂️ Error code catalog

Same table as the Spanish section above — copy from there.

---

### ⚙️ Context structure for AI models

Same JSON block as above — copy from the Spanish section. Valid, versioned, ready to inject as a `system prompt`, `.context/` file or API payload.

---

### 🔌 How to inject

Same code snippets as the Spanish section above — copy from there.

#### Quick reference

| Platform | Method |
| :--- | :--- |
| OpenAI API | `response_format: { type: "json_object" }` or `json_schema` |
| Anthropic | `tools` + `tool_choice` |
| Gemini | `response_mime_type: "application/json"` |
| n8n | AI Agent node + Code node with `JSON.parse` |
| Make.com | OpenAI module + Parse JSON |
| Zapier | Webhooks + Formatter → Text to JSON |
| Ollama | `--format json` + prompt template |

---

### 🧬 Native JSON modes vs. this layer

| Scenario | Native mode | Need this layer? |
| :--- | :--- | :--- |
| GPT-4o with `json_schema` | ✅ Schema guaranteed | ⚠️ Yes — for `status/data/error` envelope and E### codes |
| Claude 3.5 with tool use | ✅ Structure guaranteed | ⚠️ Yes — for error contract and branching |
| Gemini 1.5 with `response_schema` | ✅ Schema guaranteed | ⚠️ Yes — for consistent behavior |
| ChatGPT Web (no API) | ❌ No | ✅ **Essential** |
| Ollama / LM Studio (local) | ⚠️ Partial | ✅ **Recommended** |
| n8n / Make / Zapier with legacy modules | ❌ No | ✅ **Essential** |

> 💡 **Rule of thumb:** always use this layer, even with native mode. Native mode guarantees **syntax**; this layer guarantees **semantics** (what to do when a field is missing, how to structure errors, which code to return). They complement each other.

---

### 🤖 Compatibility

Same table as the Spanish section above.

---

### ☕ Support this open-source project

I maintain this repo in my free time because I know what it's like to fight a `JSON.parse` at 3am because the model decided to add *"Sure!"* at the start. If this layer saved you hours of debugging, you can buy me a coffee:

👉 **[ko-fi.com/yellowtips](https://ko-fi.com/yellowtips)**

---

### 📄 License

**MIT** — Free use, modification and distribution for open-source AI agents. See [`LICENSE`](LICENSE).

> This context layer is **informational only**. While designed to maximize output consistency, no LLM is 100% deterministic. Always add validation (`JSON.parse` with try/catch) and error handling in your pipeline. The author is not liable for automations that fail due to changes in underlying model behavior.

---

🔗 **Parte del [LATAM AI Context Ecosystem](https://github.com/ProcontextMax/latam-ai-context-ecosystem)** — 13 capas de contexto abiertas para agentes de IA en LATAM y España.  
🔗 **Part of the [LATAM AI Context Ecosystem](https://github.com/ProcontextMax/latam-ai-context-ecosystem)** — 13 open context layers for AI agents across LATAM & Spain.

<p align="center">
  <sub>Built with 🟡 by <a href="https://ko-fi.com/yellowtips">Yellow Tips</a> · If this helped you, ⭐ the repo and share it.</sub>
</p>
