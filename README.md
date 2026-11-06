# Agente de IA para negocios con n8n y OpenAI

**Repositorio:** [`GiBarberi/agentes-de-ia-con-n8n-y-openai`](https://github.com/GiBarberi/agentes-de-ia-con-n8n-y-openai) · **Autora:** Graciela Barberi

**Qué hace hoy el agente:** Bera atiende a clientes y al equipo de un estudio de marketing por un Webhook, recuerda la conversación, envía correos formales por Gmail, avisa al equipo por Slack y marca las urgencias con una respuesta distinta.

Proyecto del curso *Agentes de IA con n8n y OpenAI*. Cada checkpoint suma una pieza al mismo agente: en el Módulo 1, el modelo; en el 2, la memoria; en el 3, las herramientas; y en el 4, lo que lo prepara para producción.

| Checkpoint | Archivo | Qué agrega |
| --- | --- | --- |
| **1. Base técnica** | [`checkpoint1.json`](checkpoint1.json) | Webhook GET con `chatInput`, AI Agent y OpenAI Chat Model |
| **2. Agente con memoria** | [`checkpoint2.json`](checkpoint2.json) | Webhook POST, System Message con identidad y restricciones, Simple Memory (5) y Retry on Fail |
| **3. Gmail, Slack y decisión** | [`checkpoint3.json`](checkpoint3.json) | Herramientas Gmail y Slack con descripciones de negocio, y un If después del agente que separa las urgencias |

---

## Credencial de OpenAI (todos los checkpoints)

- La credencial **`OpenAI account`** (tipo *OpenAI API*) está creada desde el panel **Credentials** de n8n con una API Key propia de OpenAI Platform. n8n la probó con éxito al guardarla.
- En los tres `.json`, el sub-nodo **OpenAI Chat Model** la referencia en el bloque `credentials`, solo con `id` y `name`. **La clave nunca aparece**: ningún archivo contiene una cadena `sk-`.
- **Sobre las ejecuciones de evidencia:** la cuenta de OpenAI Platform no tiene saldo cargado. Con la credencial propia, OpenAI responde `You have no credits remaining` (ejecución 232). Por eso, las pruebas documentadas en este README corrieron con los créditos de IA incluidos en n8n Cloud (*Gateway credits*), que usan los mismos modelos de OpenAI (`gpt-4o` y `gpt-4o-mini`). Con saldo en la cuenta, el flujo funciona igual con la credencial propia, sin cambiar nada más.

---

## Checkpoint 3: agente con Gmail, Slack y lógica de decisión

### Estructura

```
[ Webhook POST ]  ──►  [ AI Agent (Bera) ]  ──►  [ ¿La respuesta es URGENTE? ]
  body.chatInput         gpt-4o · retry 3×2 s         │ sí ──► [ Responder: prioridad alta ]
  responde con nodo            │                       │ no ──► [ Responder: prioridad normal ]
                               │
        ┌──────────────┬───────┴────────┬──────────────────────┐
        ▼              ▼                ▼                      ▼
 [ OpenAI gpt-4o ] [ Simple Memory ] [ Enviar correo      ] [ Avisar al equipo   ]
                    últimas 5         [ formal (Gmail)     ] [ (Slack) #agente-bera]
```

| Requisito | Cómo se cumple |
| --- | --- |
| **Arquitectura heredada** | Webhook `POST` con `{{ $json.body.chatInput }}`, AI Agent con `gpt-4o` y Simple Memory de 5 interacciones, igual que en el Checkpoint 2 |
| **Herramienta Gmail** | Nodo *Gmail Tool* (enviar mensaje) en el puerto de herramientas, con la credencial `Gmail account` (OAuth2) y descripción propia |
| **Herramienta Slack** | Nodo *Slack Tool* (publicar mensaje) en el puerto de herramientas, con la credencial `Slack account` (OAuth2), canal `#agente-bera` y descripción propia |
| **Decisión posterior** | Nodo If **después** del agente: si la respuesta contiene `URGENTE`, va a la rama de prioridad alta |
| **Respuesta al Webhook** | Las dos ramas terminan en un *Respond to Webhook* con código 200. El flujo nunca queda en *Pending*. |
| **Seguridad** | Ninguna clave escrita en los nodos. El JSON solo trae las referencias (`id` y `name`) de las credenciales. |

### Las descripciones de las herramientas

Son lo único que el modelo lee para decidir qué herramienta usar. Las escribí en términos de negocio y dejé explícita la diferencia entre las dos: **Gmail es para gente de afuera y por escrito formal; Slack es para el equipo y avisos rápidos.**

**Enviar correo formal (Gmail):**

> Envía un CORREO FORMAL por Gmail a una persona de afuera del estudio: un cliente, un posible cliente o un proveedor. Usala cuando el usuario pida "mandale un mail", "escribile a", "enviá una propuesta o un presupuesto" o cuando haya que dejar algo por escrito con alguien externo. Necesita el email del destinatario: si el usuario no lo dio, NO la uses y pedíselo. No la uses para avisos internos al equipo (para eso está Slack).

**Avisar al equipo (Slack):**

> Publica un AVISO RÁPIDO para el EQUIPO INTERNO del estudio en el canal de Slack #agente-bera. Usala cuando el usuario pida "avisale al equipo", "contale a los chicos", "dejá un aviso" o cuando haya una novedad interna: cambios de horario de reuniones, tareas, recordatorios o un pedido urgente de un cliente que el equipo tiene que atender ya. Mensajes cortos e informales. No la uses para escribirle a clientes ni a personas de afuera (para eso está Gmail).

Los parámetros (destinatario, asunto, mensaje y texto del aviso) los completa el modelo con `$fromAI()`, también con una descripción de qué va en cada uno.

### La lógica de decisión

El System Message le pide a Bera que, si el pedido es urgente (sitio caído, cliente muy molesto, entrega que vence hoy), avise al equipo por Slack y **empiece su respuesta con `URGENTE:`**. Después de que la IA responde, el If evalúa ese texto:

| Rama | Condición | Respuesta del Webhook |
| --- | --- | --- |
| **Prioridad alta** | `output` contiene `URGENTE` | `{ "prioridad": "alta", "respuesta": "...", "aviso": "Se marcó como urgente: el equipo fue notificado en Slack (#agente-bera)." }` |
| **Prioridad normal** | cualquier otro caso | `{ "prioridad": "normal", "respuesta": "..." }` |

Primero la IA razona y actúa. Después, la lógica de negocio decide cómo se entrega el resultado.

### Cómo probarlo

```bash
curl -X POST https://grabarberi.app.n8n.cloud/webhook/checkpoint3 \
  -H "Content-Type: application/json" \
  -d '{"chatInput": "Avisale al equipo por Slack que la reunión se pasó para las 4"}'
```

El campo `sessionId` es opcional: separa la memoria por usuario.

### Evidencia

Ejecuciones reales contra la Production URL:

| Ejecución | `chatInput` | Qué hizo el agente | Rama del If | Respuesta |
| --- | --- | --- | --- | --- |
| 228 | "Avisale al equipo por Slack que la reunión se pasó para las 4" | Usó **Slack**: publicó "La reunión se pasó para las 4." en #agente-bera | normal | "Listo, le avisé al equipo que la reunión se pasó para las 4." |
| 229 | "Hola, soy Marta de la Panadería Sol. Nuestra web está caída desde hace una hora y estamos perdiendo pedidos, ¡necesito ayuda ya!" | Usó **Slack**: ":rotating_light: URGENTE: La web de la Panadería Sol está caída y están perdiendo pedidos. Necesitan ayuda inmediata." | **alta** | "URGENTE: Ya le avisé al equipo sobre la situación de tu web. Se van a poner en contacto con vos enseguida para solucionarlo." |
| 230 | "Mandale un mail a Marta de la Panadería Sol confirmando que mañana le enviamos la propuesta de su nueva web" | **No** usó Gmail porque faltaba el email: no inventó una dirección | normal | "Necesito el email de Marta para poder enviarle el correo. ¿Me lo pasás, por favor?" |
| 231 | "Mandale un mail a Grace (mi casilla personal) con el asunto \"Propuesta para tu nueva web\", contándole que mañana le enviamos la propuesta" | Usó **Gmail**: envió un correo formal firmado "Bera, asistente de Gi Barberi" (Gmail lo registró como `SENT`) | normal | "Listo, le envié el mail a Grace informándole que mañana le mandamos la propuesta de su nueva web." |

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

La credencial de OpenAI está explicada en [Credencial de OpenAI](#credencial-de-openai-todos-los-checkpoints). El `checkpoint2.json` no contiene ninguna clave.

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

Ver la sección [Credencial de OpenAI](#credencial-de-openai-todos-los-checkpoints) al principio: el sub-nodo **OpenAI Chat Model** referencia la credencial `OpenAI account`, creada en el panel de Credentials de n8n.

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
2. Creá tu credencial en **Credentials → Add Credential → OpenAI API**, pegá tu clave y guardá (tiene que aparecer "Connection tested successfully"). Después abrí el nodo **OpenAI Chat Model** y elegila.
3. Guardá, activá el workflow y probá con la URL de arriba.
