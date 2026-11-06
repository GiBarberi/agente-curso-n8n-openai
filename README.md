# Agente de IA para negocios con n8n y OpenAI

**Repositorio:** [`GiBarberi/agentes-de-ia-con-n8n-y-openai`](https://github.com/GiBarberi/agentes-de-ia-con-n8n-y-openai) · **Autora:** Graciela Barberi

Proyecto del curso *Agentes de IA con n8n y OpenAI*. Cada checkpoint suma una pieza al mismo agente: en el Módulo 1, el modelo; en el 2, la memoria; en el 3, las herramientas; y en el 4, lo que lo prepara para producción.

| Checkpoint | Archivo | Qué agrega |
| --- | --- | --- |
| **1. Base técnica** | [`checkpoint1.json`](checkpoint1.json) | Webhook GET con `chatInput`, AI Agent y OpenAI Chat Model |

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
