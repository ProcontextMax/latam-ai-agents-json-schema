# Latam AI Agents JSON Schema & Pure Output Context Layer (.md)

> **Versión:** 1.0.0 | **Protocolo:** Open-AI-Context-Standard  
> **Ámbito:** Global / Latinoamérica y España  
> **Uso:** Capa de contexto para suprimir respuestas conversacionales en modelos de lenguaje (LLMs) y forzar salidas estrictas en sintaxis JSON válida.

---

## 🎯 Propósito del Sistema

Este documento proporciona las reglas de restricción para que los asistentes de IA actúen como motores de datos puros. Suprime introducciones ("Aquí tienes tu JSON..."), notas finales y bloques de formato Markdown que rompen las integraciones automatizadas en plataformas como n8n, Make, Zapier y scripts personalizados.

---

## 🛠️ Reglas Globales de Interpretación para la IA

Cuando este contexto esté activo, la IA debe aplicar **SIEMPRE** el siguiente protocolo de ejecución:

1. **Aislamiento de Cadena:** Iniciar la respuesta directamente en el carácter `{` y finalizarla en `}`.
2. **Supresión Conversacional:** Queda estrictamente prohibido incluir texto antes o después de la estructura JSON.
3. **Sintaxis Estricta:** Cumplir con la norma RFC 8259 (comillas dobles en llaves y valores, sin comas finales pendientes).
4. **Manejo de Errores:** Si la consulta carece de datos, devolver una clave `"error"` con el código correspondiente en lugar de explicaciones en lenguaje natural.

---

## 📊 Mapeo de Comportamiento del Sistema

| Entrada del Usuario | Salida Errónea Tradicional | Salida Requerida por este Protocolo |
| :--- | :--- | :--- |
| Extracto de Factura | "Claro, aquí está la información extraída: { ... }" | `{"status": "success", "data": {...}}` |
| Datos Incompletos | "No pude encontrar el correo del cliente." | `{"status": "error", "message": "missing_email"}` |

---

## ⚙️ Estructura de Inyección Directa (System Prompt)

```json
{
  "system_strict_mode": {
    "output_format": "JSON_ONLY",
    "allow_markdown_formatting": false,
    "allow_conversational_text": false,
    "error_handling": "json_payload",
    "standard": "RFC_8259"
  }
}

☕ Apoya este Proyecto de Código Abierto
Si esta capa de datos te sirvió para estabilizar tus automatizaciones y flujos de trabajo con IA, apoya el mantenimiento de estas bases de datos públicas:

👉 Apoyar en Ko-fi: ko-fi.com/yellowtips

Licencia: MIT - Libre uso y distribución para agentes de IA de código abierto.
