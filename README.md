# Agente de IA para negocios con n8n y OpenAI

**Repositorio:** [`GiBarberi/agentes-de-ia-con-n8n-y-openai`](https://github.com/GiBarberi/agentes-de-ia-con-n8n-y-openai) · **Autora:** Graciela Barberi

Proyecto del curso *Agentes de IA con n8n y OpenAI*. Cada checkpoint suma una pieza al mismo agente: en el Módulo 1, el modelo; en el 2, la memoria; en el 3, las herramientas; y en el 4, lo que lo prepara para producción.

| Checkpoint | Archivo | Qué agrega |
| --- | --- | --- |
| **1. Base técnica** | [`checkpoint1.json`](checkpoint1.json) | Webhook GET con `chatInput`, AI Agent y OpenAI Chat Model |
| **2. Agente con memoria** | [`checkpoint2.json`](checkpoint2.json) | Webhook POST, System Message con identidad y restricciones, Simple Memory (5) y Retry on Fail |

---

## Checkpoint 2: agente con webhook y Simple Memory

El agente ahora se llama **Bera**: es la asistente virtual de *Gi Barberi*, un estudio de marketing digital y diseño web. Recibe mensajes por POST y recuerda la conversación.

### Estructura

```
[ Webhook POST (chatInput) ]  ──────►  [ AI Agent (Bera) ]
  método POST · respuesta 200            System Message: identidad + tono + restricciones
  body: { "chatInput": "..." }           prompt: {{ $json.body.chatInput }}
                                         Retry on Fail: 3 intentos × 2 s
                                               │
                              ┌────────────────┴────────────────┐
                              ▼                                 ▼
                [ OpenAI Chat Model (gpt-4o) ]      [ Simple Memory (últimas 5) ]
                  Retry on Fail: 3 × 2 s              Context Window: 5
```

| Requisito | Cómo se cumple |
| --- | --- |
| **Nombre del flujo** | `Checkpoint 2: Agente con Memoria` |
| **Webhook** | Método `POST`, HTTP Response Code `200`. Responde con la salida del agente. |
| **Entrada** | El prompt del agente es `{{ $json.body.chatInput }}`: el prefijo pasó de `query` a `body` porque el dato viaja en el cuerpo de la petición. |
| **Modelo** | `gpt-4o` |
| **System Message** | Identidad (Bera, de Gi Barberi), tono (cálido, profesional, en "vos") y restricciones explícitas: solo habla de marketing, diseño y los servicios del estudio, y no inventa precios, plazos ni clientes. |
| **Simple Memory** | Context Window Length = `5` (recuerda las últimas 5 interacciones) |
| **Robustez** | Retry on Fail activado en el AI Agent y en el OpenAI Chat Model: 3 intentos con 2 s de espera |

### System Message

```text
Sos Bera, la asistente virtual de Gi Barberi, un estudio de marketing digital y diseño web de Buenos Aires.

Identidad y tono:
- Tu tono es cálido, profesional y cercano, en español rioplatense (vos).
- Respondés en frases cortas y claras, sin tecnicismos innecesarios.

Objetivo:
- Saludar a la persona, recordar su nombre y lo que te cuenta durante la conversación.
- Orientarla sobre los servicios del estudio: diseño y desarrollo web, identidad de marca, SEO y campañas de publicidad digital.
- Si quiere avanzar, pedirle su nombre, su email y una breve descripción del proyecto para que el equipo la contacte.

Restricciones:
- Solo hablás de marketing digital, diseño, sitios web y los servicios del estudio. Si te preguntan otra cosa (política, salud, tareas escolares, etc.), decí amablemente que no es tu tema y volvé a ofrecer ayuda con su proyecto.
- No inventes precios, plazos ni casos de clientes: si te los piden, explicá que el equipo los confirma en una propuesta personalizada.
- Si no sabés algo, decí que no lo sabés en lugar de suponer.
```

### Clave de la conversación (Context Key)

La memoria usa como clave `{{ $json.body.sessionId || 'conversacion-demo' }}`:

- Si el POST trae solo `chatInput` (como pide la consigna), todos los mensajes comparten la conversación `conversacion-demo`, y la prueba de dos mensajes funciona.
- Si además trae un `sessionId`, cada usuario tiene su propia memoria. Es lo que va a necesitar el agente cuando lo conectemos a Slack o WhatsApp.

### Cómo probarlo

**Postman:** método `POST`, URL de abajo, *Body → raw → JSON*:

```json
{ "chatInput": "Hola, soy Carlos" }
```

**curl:**

```bash
curl -X POST https://grabarberi.app.n8n.cloud/webhook/checkpoint2 \
  -H "Content-Type: application/json" \
  -d '{"chatInput": "Hola, soy Carlos"}'
```

- Test URL: `https://grabarberi.app.n8n.cloud/webhook-test/checkpoint2` (con "Listen for Test Event" activo)
- Production URL: `https://grabarberi.app.n8n.cloud/webhook/checkpoint2`

### Evidencia

Ejecuciones reales contra la Production URL:

| Ejecución | `chatInput` | Respuesta del agente |
| --- | --- | --- |
| 225 | "Hola, soy Carlos" | "¡Hola, Carlos! Un gusto saludarte. ¿En qué puedo ayudarte hoy?" |
| 226 | "¿Cómo me llamo?" | "Te llamás Carlos. ¿En qué te gustaría que te ayude hoy?" ✅ **memoria** |
| 227 | "¿Me recomendás un remedio para el dolor de cabeza?" | "Entiendo que te sentís mal, pero no es mi área de especialización. Si querés, puedo ayudarte con temas de marketing digital o diseño web. ¿Te interesa algún proyecto en particular?" ✅ **restricción** |

La credencial de OpenAI es la misma que en el Checkpoint 1 (créditos de IA de n8n Cloud). El `checkpoint2.json` no contiene ninguna clave.

---

## Checkpoint 1: base técnica conectada

### Estructura

```
[ Webhook (chatInput) ]  ──────►  [ AI Agent ]
  método GET                        prompt: Eres un asistente técnico. Responde de forma
  param: chatInput                  breve y en el idioma del mensaje a esto:
  responde con el último nodo       {{ $json.query.chatInput }}
                                          │
                                          ▼
                              [ OpenAI Chat Model (gpt-4o-mini) ]
                                temperatura 0.4
```

| Requisito | Cómo se cumple |
| --- | --- |
| **Nombre del flujo** | `PF - Checkpoint 1 - Base Técnica` |
| **Estructura mínima** | Webhook → AI Agent, con el sub-nodo OpenAI Chat Model enchufado abajo |
| **Entrada dinámica** | El Webhook recibe `chatInput` por la URL y el prompt lo lee con la expresión `{{ $json.query.chatInput }}`. No hay ningún texto fijo. |
| **Modelo** | `gpt-4o-mini` |
| **Sin claves en el flujo** | No hay ninguna API Key en el prompt, en los parámetros ni en notas. El `.json` no contiene ninguna cadena `sk-`. |

### Credencial de OpenAI

El modelo usa la credencial **"Gateway credits"**, que n8n Cloud gestiona desde su propio panel de credenciales. Son créditos de IA incluidos en el plan que dan acceso a los modelos de OpenAI sin cargar saldo en una cuenta propia de OpenAI Platform.

Elegí esta opción para no generar costos durante el curso. Cumple con el mismo principio de seguridad: la clave vive cifrada en n8n y nunca aparece en el flujo ni en el repositorio.

**Para usar tu propia clave** al importar el flujo:

1. En platform.openai.com, cargá saldo (mínimo 5 USD) y creá una *Secret Key*.
2. En n8n, andá a **Credentials → Add Credential → OpenAI API**, pegá la clave y guardá. Tiene que aparecer "Connection tested successfully".
3. Abrí el nodo **OpenAI Chat Model** y elegí esa credencial.

### Cómo probarlo

**Test URL** (con el workflow abierto y "Listen for Test Event" activo):

```
https://grabarberi.app.n8n.cloud/webhook-test/checkpoint1?chatInput=Hola+Agente
```

**Production URL** (el workflow está activo):

```
https://grabarberi.app.n8n.cloud/webhook/checkpoint1?chatInput=Hola+Agente
```

Se puede abrir directamente en el navegador. El Webhook está configurado para responder con el resultado del último nodo, así que devuelve la respuesta del agente en el momento:

```json
{ "output": "<respuesta del agente>" }
```

### Evidencia

Ejecución con `chatInput = "Hola Agente, ¿qué es n8n?"`, sin errores:

```json
{
  "output": "Hola, n8n es una herramienta de automatización de flujos de trabajo que permite conectar diferentes aplicaciones y servicios para automatizar tareas sin necesidad de codificación. Es de código abierto y ofrece una interfaz visual para diseñar flujos de trabajo."
}
```

### Cómo importarlo

1. En n8n: **Workflows → Import from File** → `checkpoint1.json`.
2. Abrí el nodo **OpenAI Chat Model** y asigná una credencial de OpenAI (la tuya o los créditos de n8n).
3. Guardá, activá el workflow y probá con la URL de arriba.
