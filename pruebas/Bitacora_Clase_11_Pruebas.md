---
tipo: bitacora-clase
curso: Automatización inteligente de procesos con n8n
periodo: EAM-2026II
corte: 3
clase: 11
tema: Pruebas y confiabilidad de flujos con IA
fecha: 2026-11-09
equipo: Argoty, Cruz, Hernández, Sinisterra
---

# Bitácora de la clase 11

## Revisión integral del proyecto y pendientes para la sustentación

**Curso:** Automatización inteligente con n8n

**Tema:** Pruebas y confiabilidad de flujos con IA

Esta bitácora documenta la revisión del diseño con la versión actual del 7.0. Las pruebas formales y sus evidencias corresponden a la sustentación. Localizar una validación en un nodo no demuestra que funcione en todos los casos; por eso los resultados se registran como pendientes.

## 1. Identificación

- **Fecha de la clase:** 2026-11-09
- **Proyecto:** Seguimiento y clasificación automática de leads — Inmonova S.A.S.
- **Integrantes presentes:** Johan Steven Hernández Torres, Javier Leonardo Argoty Roa, Luis Eduardo Sinisterra Montaño, Simón David Cruz Suazo
- **Versión revisada:** `Workflow_07_Agente.json` — “Seguimiento leads 7.0 - Main” (ID `GRulfo4U6gWr0wOi`), exportado el 2026-11-09.
- **Workflows relacionados:**
  - **7.0 Main** — webhook Telegram, agente IA, guardado en Baserow, confirmación al chat.
  - **6.2 Subworkflow_PrepararLead** (ID `GxDZlth0DF976v72`) — clasifica tipo de correo y nivel de interés.
  - **6.3 Manejador** (ID `Vh3rn4FIK4HlDMhU`) — Error Trigger; registra incidentes en Baserow. **No está asignado al 7.0** (ver P-01); sí está asignado al 6.4.
  - **6.4 Lab** (ID `d1x3RFcujJXrEzr1`) — Schedule Trigger para probar el manejador con fallo forzado; inactivo.
- **Diagrama revisado:** canvas del 7.0 en n8n. No hay diagrama actualizado del Corte 3; los diagramas existentes son del Corte 1 y Corte 2 en `04_Diagramas/`.

## 2. Recorrido completo del proyecto

El usuario escribe en lenguaje natural al bot de Telegram. El webhook recibe el POST, responde 200 inmediato y pasa el texto al Agente de captación de leads (Google Gemini, memoria de ventana de 10 mensajes). El agente conversa y recopila **cuatro** datos: nombre, correo, interés y mensaje. El canal, `chat_id`, `update_id`, la fecha y `formMode` los asigna `Parsear mensaje` desde el evento real, no la IA.

El agente devuelve un JSON con `estado: listo` cuando tiene todo, `estado: recopilando` cuando falta algo (con `respuesta_usuario`) o `estado: sin_registro` cuando el mensaje no requiere un lead. `Parsear mensaje` valida el JSON y la estructura; `Contrato completo?` solo deja pasar `listo` con nombre, correo con formato `x@y.z`, interés en `ALTO/MEDIO/BAJO`, mensaje y `chat_id`.

Si pasa, el flujo invoca el 6.2, que agrega `es_empresarial` y `accion`. Luego un HTTP POST guarda el lead en Baserow (2 reintentos, `continueErrorOutput`). En éxito, el bot confirma. En fallo tras los reintentos, se registra un incidente en Baserow y el bot avisa que el lead quedó en revisión manual.

Si el agente falla, su salida de error va a `Fallo del agente`, pero **esa rama no tiene conexión de salida**: el usuario no recibe respuesta y no queda incidente. Es una brecha (RC-08).

| Etapa | Workflow y nodo | Qué debe garantizar | Destino |
|---|---|---|---|
| Recepción | 7.0 · Webhook → Responder 200 | Recibe el POST y responde 200 inmediato para evitar reintentos de Telegram | Agente |
| Consulta de fuentes | No aplica | El proceso no consulta catálogo, disponibilidad ni otra fuente externa; el agente solo interpreta el mensaje | — |
| Conversación e IA | 7.0 · Agente + Simple Memory | Mantener contexto por chat y usuario; recopilar los 4 datos sin inventar | Parsear mensaje o Fallo del agente |
| Parseo de salida | 7.0 · Parsear mensaje | Verificar JSON válido, estado permitido y estructura del lead; agregar metadatos del evento | Contrato completo? |
| Validación | 7.0 · Contrato completo? | Comprobar estado `listo`, nombre, correo con formato `x@y.z`, interés permitido, mensaje y `chat_id` | PrepararLead o Responder conversación |
| Clasificación | 6.2 · If correo + Switch interés | Asignar `es_empresarial` y `accion` según las reglas de negocio | HTTP · Guardar fila |
| Guardado externo | 7.0 · HTTP · Guardar fila Baserow1 | Crear la fila en la tabla de leads (1227763) con los 9 campos, con 2 reintentos y salida de error | Confirmar o ruta de incidente |
| Respuesta final | 7.0 · Confirmar / Aviso de error / Responder conversación | El usuario recibe confirmación, aviso de error o pregunta de seguimiento | Fin |
| Fallos y recuperación | 7.0 · HTTP Registrar Incidente → Aviso de error / 6.3 Manejador | Registrar el incidente antes de avisar; el 6.3 cubre errores no controlados si se asigna | Tabla de incidentes (1282332) |

**Nota de implementación:** el body del HTTP de guardado manda `formMode: 'test'` fijo, aunque `Parsear mensaje` calcula `'telegram'`. Las filas quedan como `test`. Es una brecha de calidad de datos (RC-12, P-09).

## 3. Revisión de situaciones por etapa

| ID | Etapa y situación concreta | Resultado esperado y efecto que debe evitarse | Protección actual y ubicación | Estado de revisión y justificación |
|---|---|---|---|---|
| RC-01 | Recepción · Lead válido completo en un turno | Lead guardado en Baserow y confirmación al chat. Evitar: fila duplicada, datos incompletos o confirmación sin fila | Webhook + `Contrato completo?` + HTTP con 2 reintentos | Protección localizada — requiere comprobación end-to-end |
| RC-02 | Conversación · Mensaje que no requiere lead (`sin_registro`: agradecimiento o consulta ajena) | Responder sin guardar. Evitar: crear una fila con datos nulos o inventados | `Parsear mensaje` acepta `sin_registro`; el If lo manda a `Responder conversación` | Protección localizada — requiere comprobación |
| RC-03 | Recepción · Mensaje sin texto (foto, sticker, audio) | Pedir texto y no guardar. Evitar: error técnico sin respuesta al usuario | Ninguna validación de `message.text` antes del agente; el agente recibe `''` | Brecha identificada — P-02 |
| RC-04 | Conversación · Datos incompletos en varios turnos | El agente pregunta dentro de la ventana de 10 mensajes; no guarda hasta tener todo. Evitar: guardado parcial o pérdida de datos | Simple Memory (`chat_id + from_id`, ventana 10 mensajes) | Protección localizada — sin aviso al agotar la ventana; requiere comprobación |
| RC-05 | Conversación · Interés ambiguo (“más o menos”) | El agente pregunta y no asume. Evitar: clasificación inventada | systemMessage: usa `null` y pregunta si no alcanza | Protección localizada — depende del modelo |
| RC-06 | Validación · Correo inválido o en el límite (`sin @`, `sin punto`, `x@y`) | Rechazar y pedir corrección. Evitar: guardar un correo inválido | Regex `/^[^\s@]+@[^\s@]+\.[^\s@]+$/` en `Contrato completo?` | Protección localizada — probar antes, en y después del formato |
| RC-07 | Parseo · El agente devuelve texto libre, JSON inválido o un estado no permitido | Detener el flujo y registrar el fallo. Evitar: continuar con datos incorrectos | `JSON.parse` + `throw` en `Parsear mensaje` | Brecha identificada — el 7.0 no tiene `errorWorkflow`; el error no llega al 6.3 ni queda registrado |
| RC-08 | IA · Google Gemini no disponible | Avisar al usuario y registrar el incidente. Evitar: error silencioso sin respuesta | Salida de error del agente → `Fallo del agente` | Brecha identificada — `Fallo del agente` no tiene salida; la rama termina sin aviso ni registro (P-03) |
| RC-09 | Clasificación · Lead empresarial con interés ALTO | `es_empresarial=true`, `accion="Notificacion inmediata"`. Evitar: clasificación incorrecta | 6.2 · If correo + Switch ALTO | Protección localizada — verificada en Corte 2; requiere comprobación end-to-end con el agente |
| RC-10 | Clasificación · Correo personal de dominio no listado (`@yahoo.com`, `@icloud.com`) | Clasificar como personal (`false`). Evitar: marcarlo como empresarial | El If solo excluye `@gmail`, `@hotmail` y `@outlook`; el resto da `true` | Brecha identificada — P-08 |
| RC-11 | Guardado · Baserow no responde tras los 2 reintentos | Crear el incidente y avisar al usuario. Evitar: confirmación falsa | `continueErrorOutput` → `HTTP Registrar Incidente` → `Aviso de error` | Protección localizada — requiere comprobación |
| RC-12 | Guardado · Error 400 de Baserow (por ejemplo, `canal` no admitido o `formMode` inválido) | Tomar la ruta de incidente y avisar. Evitar: error silencioso o fila con datos de prueba | `continueErrorOutput`; el body manda `formMode: 'test'` fijo | Brecha identificada — la ruta existe, pero el `formMode` está fijo en `test` (P-09) |
| RC-13 | Respuesta · El lead se guardó, pero falla `Confirmar registro al chat` | Avisar o registrar el estado parcial. Evitar: el usuario cree que no se guardó y reintenta | Sin `onError` en el nodo de confirmación | Brecha identificada — el fallo es visible en Executions, pero no queda en incidentes (P-05) |
| RC-14 | Fallos · Falla `HTTP Registrar Incidente` | Avisar al usuario y dejar rastro. Evitar: ni incidente ni aviso | El nodo no tiene `onError` | Brecha identificada — si falla el registro, la ejecución termina sin aviso (P-04) |
| RC-15 | Fallos · Error no controlado en el 7.0 (en `Parsear`, 6.2, Telegram o HTTP) | Llegar al 6.3 y registrar el incidente. Evitar: error silencioso | 6.3 existe, pero el 7.0 tiene `errorWorkflow: None` | Brecha identificada — P-01 |
| RC-16 | Repetición · El mismo lead se envía dos veces | Una sola fila. Evitar: duplicado | Ninguna; `update_id` está disponible pero no se usa | Brecha identificada — P-06; no estaba documentada en la C08 |
| RC-17 | Cambio de contexto · El mismo chat inicia un lead nuevo después de cerrar | Reiniciar la recopilación sin reutilizar el lead anterior. Evitar: mezclar datos de dos leads | La memoria conserva los 10 mensajes; el prompt pide reiniciar, pero depende del modelo | Brecha identificada — P-10 |
| RC-18 | Uso indebido · POST al webhook sin ser Telegram | Rechazar la solicitud. Evitar: inyectar leads falsos | El webhook no tiene secret token ni autenticación | Brecha identificada — P-07 |
| RC-19 | Uso indebido · Jailbreak (“ignora las instrucciones”) | Mantener el rol y no revelar el systemMessage. Evitar: cambiar reglas o formato de salida | systemMessage delimita el rol y trata los mensajes como datos | Protección localizada — depende del modelo |
| RC-20 | Información · Dos usuarios en chats distintos | No mezclar datos entre conversaciones. Evitar: que un usuario vea datos de otro | Clave de memoria `chat_id + from_id` | Protección localizada — requiere comprobación |
| RC-NA-01 | Consulta de fuentes | No aplica: el proceso no consulta catálogo, disponibilidad ni otra fuente externa; el agente solo interpreta el mensaje | — | No aplica justificado |
| RC-NA-02 | Límites numéricos | No aplica: el proyecto no maneja montos, cupos ni cantidades; los límites son el formato de correo y el enum de interés | — | No aplica justificado |

**Etapas o situaciones que todavía faltan por revisar:**

- Confirmar el nombre real de la tabla `1282332` y si la opción `Telegram` existe en la columna `canal`.
- Confirmar que el nodo `PrepararLead'` apunta al 6.2 correcto (`GxDZlth0DF976v72`).
- Verificar el comportamiento del agente con `maxIterations: 4` en conversaciones largas.
- Verificar si el `formMode: 'test'` es aceptado por la columna de Baserow.

## 4. Casos previstos para comprobar el comportamiento

### Caso CP-01: Lead válido en lenguaje natural, un turno

- **Revisión asociada:** RC-01, RC-09, RC-12
- **Workflow y etapas involucradas:** 7.0 completo + 6.2
- **Qué se busca comprobar:** El agente extrae los datos, el contrato los acepta, 6.2 clasifica y Baserow recibe la fila con los 9 campos
- **Condiciones iniciales:** 7.0 activo, túnel activo, opción `Telegram` en la columna `canal`, 6.2 activo
- **Entrada exacta:** `Hola, me llamo Laura Gómez, mi correo es laura@miempresa.com, tengo interés alto en un apartamento para comprar en Armenia.`
- **Resultado esperado:** Fila en la tabla de leads (1227763) con `nombre`, `correo`, `interes=ALTO`, `mensaje`, `canal=Telegram`, `submittedAt`, `formMode=telegram`, `es_empresarial=true`, `accion="Notificacion inmediata"`; confirmación al chat
- **Criterio de aceptación:** Una sola fila con los 9 campos correctos, `formMode=telegram`, y confirmación recibida en el chat
- **Efecto que no debe ocurrir:** Fila duplicada, clasificación incorrecta, confirmación sin fila o `formMode=test`
- **Evidencia que se buscará:** Captura del chat, ejecución en Executions, fila en Baserow antes y después
- **Cómo se preparará de forma controlada:** Cuenta de Telegram de prueba; verificar que no existe la fila antes de ejecutar
- **Resultado observado:** Pendiente de ejecución.
- **Conclusión de la prueba:** Pendiente de ejecución.
- **Evidencia de ejecución:** Pendiente de ejecución.

### Caso CP-02: Conversación de varios turnos

- **Revisión asociada:** RC-04, RC-05
- **Workflow y etapas involucradas:** 7.0 · Agente + Simple Memory
- **Qué se busca comprobar:** El agente mantiene contexto y no guarda hasta tener todos los datos
- **Condiciones iniciales:** Mismas que CP-01; primer mensaje sin nivel de interés
- **Entrada exacta:** Turno 1: `Hola, soy Pedro Ruiz, pedro@hotmail.com, quiero info de un apartamento.` Turno 2: `Interés medio, estoy comparando opciones.`
- **Resultado esperado:** El bot pregunta el interés tras el turno 1; tras el turno 2 guarda con `interes=MEDIO`, `es_empresarial=false`, `accion="Registrar seguimiento en 2 o 3 dias"`
- **Criterio de aceptación:** Una sola fila con los datos del turno 2; no hay fila parcial del turno 1
- **Efecto que no debe ocurrir:** Guardado parcial tras el turno 1 o pérdida del nombre/correo entre turnos
- **Evidencia que se buscará:** Capturas de ambos turnos, dos ejecuciones en Executions y la fila en Baserow
- **Cómo se preparará de forma controlada:** Misma cuenta de prueba; verificar que no hay fila de Pedro Ruiz antes
- **Resultado observado:** Pendiente de ejecución.
- **Conclusión de la prueba:** Pendiente de ejecución.
- **Evidencia de ejecución:** Pendiente de ejecución.

### Caso CP-03: Mensaje sin texto (foto o sticker)

- **Revisión asociada:** RC-03
- **Workflow y etapas involucradas:** 7.0 · Webhook → Agente → Parsear mensaje
- **Qué se busca comprobar:** El flujo no lanza error técnico; el bot responde pidiendo texto
- **Condiciones iniciales:** 7.0 activo, túnel activo
- **Entrada exacta:** Enviar una foto sin caption al bot
- **Resultado esperado:** El bot responde solicitando texto; ninguna fila en Baserow
- **Criterio de aceptación:** Sin error en Executions y sin fila nueva en Baserow
- **Efecto que no debe ocurrir:** Error técnico en `Parsear mensaje` sin respuesta al usuario
- **Evidencia que se buscará:** Captura del chat y de la ejecución
- **Cómo se preparará de forma controlada:** Foto de prueba desde la cuenta de prueba
- **Resultado observado:** Pendiente de ejecución.
- **Conclusión de la prueba:** Pendiente de ejecución.
- **Evidencia de ejecución:** Pendiente de ejecución.

### Caso CP-04: Mensaje que no requiere lead (`sin_registro`)

- **Revisión asociada:** RC-02
- **Workflow y etapas involucradas:** 7.0 · Agente → Parsear mensaje → Contrato completo?
- **Qué se busca comprobar:** Un agradecimiento o una consulta ajena no crea una fila
- **Condiciones iniciales:** 7.0 activo, túnel activo, sin fila previa de la cuenta de prueba
- **Entrada exacta:** `Gracias, ya me contactaron.`
- **Resultado esperado:** El agente devuelve `sin_registro`, el bot responde y no se guarda nada
- **Criterio de aceptación:** Sin fila nueva en Baserow y respuesta recibida en el chat
- **Efecto que no debe ocurrir:** Fila con datos nulos o inventados
- **Evidencia que se buscará:** Captura del chat, ejecución y tabla de leads antes/después
- **Cómo se preparará de forma controlada:** Cuenta de prueba; verificar que no existe la fila antes
- **Resultado observado:** Pendiente de ejecución.
- **Conclusión de la prueba:** Pendiente de ejecución.
- **Evidencia de ejecución:** Pendiente de ejecución.

### Caso CP-05: Fallo de Baserow tras 2 reintentos

- **Revisión asociada:** RC-11, RC-12
- **Workflow y etapas involucradas:** 7.0 · HTTP Guardar → HTTP Registrar Incidente → Aviso de error
- **Qué se busca comprobar:** Tras los reintentos, se crea el incidente y el bot avisa al usuario
- **Condiciones iniciales:** Copia del workflow con un **ID de tabla inválido** en el nodo de guardado; el nodo de incidentes conserva la tabla y la credencial válidas
- **Entrada exacta:** Lead válido completo (mismos datos que CP-01)
- **Resultado esperado:** Sin fila en la tabla de leads; fila en la tabla de incidentes con `estado=pendiente_revision`; aviso al chat
- **Criterio de aceptación:** Fila de incidente en Baserow y aviso recibido
- **Efecto que no debe ocurrir:** Confirmación falsa al usuario o ningún registro del incidente
- **Evidencia que se buscará:** Capturas de ambas tablas, del chat y de la ejecución
- **Cómo se preparará de forma controlada:** Copia del workflow; **no desactivar la credencial**, porque el nodo de incidentes usa la misma; restaurar la tabla al terminar
- **Resultado observado:** Pendiente de ejecución.
- **Conclusión de la prueba:** Pendiente de ejecución.
- **Evidencia de ejecución:** Pendiente de ejecución.

### Caso CP-06: Error no controlado y activación del manejador 6.3

- **Revisión asociada:** RC-07, RC-15
- **Workflow y etapas involucradas:** 7.0 (errorWorkflow) → 6.3 Manejador
- **Qué se busca comprobar:** Con `errorWorkflow` configurado, los errores no controlados del 7.0 llegan al 6.3 y quedan registrados
- **Condiciones iniciales:** Copia del 7.0 con `errorWorkflow` = `Vh3rn4FIK4HlDMhU` (6.3); forzar un error con `Stop and Error` o un JSON inválido en una copia
- **Entrada exacta:** Ejecución manual de la copia con el error forzado
- **Resultado esperado:** El 6.3 se activa y crea una fila con workflow, nodo y mensaje del error
- **Criterio de aceptación:** Fila en la tabla de incidentes con los datos correctos del error forzado
- **Efecto que no debe ocurrir:** Error silencioso sin registro
- **Evidencia que se buscará:** Captura de la tabla de incidentes y de la ejecución del 6.3
- **Cómo se preparará de forma controlada:** Configurar `errorWorkflow` solo en la copia; no modificar el 7.0 real durante la prueba
- **Resultado observado:** Pendiente de ejecución.
- **Conclusión de la prueba:** Pendiente de ejecución.
- **Evidencia de ejecución:** Pendiente de ejecución.

### Caso CP-07: El mismo lead enviado dos veces

- **Revisión asociada:** RC-16
- **Workflow y etapas involucradas:** 7.0 completo
- **Qué se busca comprobar:** Si el mismo mensaje se envía dos veces, se crea una sola fila
- **Condiciones iniciales:** 7.0 activo; contar las filas antes de la prueba
- **Entrada exacta:** Enviar dos veces el mensaje de CP-01, con segundos de diferencia
- **Resultado esperado:** Una sola fila en Baserow
- **Criterio de aceptación:** El conteo de filas aumenta en 1, no en 2
- **Efecto que no debe ocurrir:** Dos filas sin trazabilidad
- **Evidencia que se buscará:** Captura del chat, dos ejecuciones y conteo de filas antes/después
- **Cómo se preparará de forma controlada:** Cuenta de prueba; borrar las filas de la prueba al terminar
- **Resultado observado:** Pendiente de ejecución.
- **Conclusión de la prueba:** Pendiente de ejecución.
- **Evidencia de ejecución:** Pendiente de ejecución.

### Caso CP-08: Lead nuevo en el mismo chat

- **Revisión asociada:** RC-17
- **Workflow y etapas involucradas:** 7.0 · Agente + Simple Memory
- **Qué se busca comprobar:** Después de cerrar un lead, el mismo chat puede registrar otro sin mezclar datos
- **Condiciones iniciales:** Un lead ya registrado en ese chat; 7.0 activo
- **Entrada exacta:** `Quiero registrar otro lead: Ana Torres, ana@hotmail.com, interés bajo, quiere vender.`
- **Resultado esperado:** Una fila nueva con los datos de Ana; no reutiliza nombre, correo ni mensaje del lead anterior
- **Criterio de aceptación:** La fila nueva contiene solo los datos de Ana
- **Efecto que no debe ocurrir:** Mezcla de datos entre los dos leads
- **Evidencia que se buscará:** Captura del chat, ejecución y filas antes/después
- **Cómo se preparará de forma controlada:** Cuenta de prueba con un lead previo identificable
- **Resultado observado:** Pendiente de ejecución.
- **Conclusión de la prueba:** Pendiente de ejecución.
- **Evidencia de ejecución:** Pendiente de ejecución.

### Caso CP-09: Límites del correo y dominio personal

- **Revisión asociada:** RC-06, RC-10
- **Workflow y etapas involucradas:** 7.0 · Contrato completo? + 6.2 · If correo
- **Qué se busca comprobar:** El formato del correo se valida y un dominio personal no listado se clasifica como personal
- **Condiciones iniciales:** 7.0 y 6.2 activos
- **Entrada exacta:** Cuatro objetos con `estado='listo'`, `nombre='Laura Gómez'`, `interes='ALTO'`, `mensaje='Quiere comprar un apartamento en Armenia.'`, `chat_id=123456789`, `canal='Telegram'`, `submittedAt='2026-11-09T10:00:00.000Z'`, `formMode='telegram'`, y `correo` igual a `laura@miempresa.com`, `laura@miempresa`, `laura@.com` y `laura@yahoo.com`, respectivamente
- **Resultado esperado:** El primero se guarda; el segundo y el tercero se rechazan; el cuarto se guarda con `es_empresarial=false`
- **Criterio de aceptación:** Solo se guardan los correos con formato válido y `yahoo.com` queda como personal
- **Efecto que no debe ocurrir:** Correo inválido guardado o `yahoo.com` marcado como empresarial
- **Evidencia que se buscará:** Salida de `Contrato completo?`, filas creadas y clasificación en Baserow
- **Cómo se preparará de forma controlada:** Copia del 7.0 con un nodo Set que inyecta cada juego directamente en `Contrato completo?`; así se prueba la regla sin depender de la respuesta del agente. Borrar las filas al terminar
- **Resultado observado:** Pendiente de ejecución.
- **Conclusión de la prueba:** Pendiente de ejecución.
- **Evidencia de ejecución:** Pendiente de ejecución.

### Caso CP-10: POST al webhook sin ser Telegram

- **Revisión asociada:** RC-18
- **Workflow y etapas involucradas:** 7.0 · Webhook Telegram
- **Qué se busca comprobar:** El webhook rechaza una solicitud que no viene de Telegram
- **Condiciones iniciales:** 7.0 activo y URL del túnel conocida
- **Entrada exacta:** `curl -X POST <url-del-webhook> -H 'Content-Type: application/json' -d '{"message":{"text":"lead falso","chat":{"id":1},"from":{"id":1}}}'`
- **Resultado esperado:** La solicitud se rechaza (401/403) y no se crea ninguna fila
- **Criterio de aceptación:** Sin fila nueva en Baserow y respuesta de rechazo
- **Efecto que no debe ocurrir:** Lead falso guardado
- **Evidencia que se buscará:** Salida de `curl`, ejecución en Executions y tabla de leads
- **Cómo se preparará de forma controlada:** Solo contra la instancia de prueba; no usar datos reales
- **Resultado observado:** Pendiente de ejecución.
- **Conclusión de la prueba:** Pendiente de ejecución.
- **Evidencia de ejecución:** Pendiente de ejecución.

### Caso CP-11: El lead se guarda, pero falla la confirmación

- **Revisión asociada:** RC-13
- **Workflow y etapas involucradas:** 7.0 · HTTP Guardar → Confirmar registro al chat
- **Qué se busca comprobar:** Si Baserow guarda y Telegram falla, el sistema deja rastro del estado parcial
- **Condiciones iniciales:** Copia del 7.0 con una credencial de Telegram inválida solo en `Confirmar registro al chat`; Baserow válido
- **Entrada exacta:** Lead válido completo (mismos datos que CP-01)
- **Resultado esperado:** Fila guardada en Baserow; el fallo queda visible en Executions y, si se implementa P-05, en incidentes
- **Criterio de aceptación:** La fila existe y el fallo tiene traza localizable
- **Efecto que no debe ocurrir:** El usuario cree que no se guardó y no hay forma de detectarlo
- **Evidencia que se buscará:** Captura de la fila, de la ejecución fallida y del chat
- **Cómo se preparará de forma controlada:** Copia del workflow; restaurar la credencial al terminar
- **Resultado observado:** Pendiente de ejecución.
- **Conclusión de la prueba:** Pendiente de ejecución.
- **Evidencia de ejecución:** Pendiente de ejecución.

### Caso CP-12: Falla el registro del incidente

- **Revisión asociada:** RC-14
- **Workflow y etapas involucradas:** 7.0 · HTTP Guardar (error) → HTTP Registrar Incidente → Aviso de error
- **Qué se busca comprobar:** Si falla el registro del incidente, el usuario igual recibe un aviso o queda una traza
- **Condiciones iniciales:** Copia con el guardado y el registro de incidentes apuntando a tablas inválidas
- **Entrada exacta:** Lead válido completo (mismos datos que CP-01)
- **Resultado esperado:** El usuario recibe un aviso; el fallo queda visible en Executions
- **Criterio de aceptación:** Aviso al chat o traza localizable
- **Efecto que no debe ocurrir:** Silencio total, sin aviso ni registro
- **Evidencia que se buscará:** Captura del chat y de la ejecución
- **Cómo se preparará de forma controlada:** Copia del workflow; no tocar la tabla real
- **Resultado observado:** Pendiente de ejecución.
- **Conclusión de la prueba:** Pendiente de ejecución.
- **Evidencia de ejecución:** Pendiente de ejecución.

### Caso CP-13: Gemini no disponible

- **Revisión asociada:** RC-08
- **Workflow y etapas involucradas:** 7.0 · Agente → Fallo del agente
- **Qué se busca comprobar:** Si el modelo falla, el usuario recibe un aviso y queda un incidente
- **Condiciones iniciales:** Copia del 7.0 con una credencial de Gemini inválida
- **Entrada exacta:** `Hola, quiero información de un apartamento.`
- **Resultado esperado:** El bot avisa que no pudo procesar la solicitud; no se guarda nada; queda un incidente
- **Criterio de aceptación:** Aviso al chat y fila de incidente
- **Efecto que no debe ocurrir:** Silencio total o error técnico sin respuesta
- **Evidencia que se buscará:** Captura del chat, de la ejecución y de la tabla de incidentes
- **Cómo se preparará de forma controlada:** Copia del workflow; restaurar la credencial al terminar
- **Resultado observado:** Pendiente de ejecución.
- **Conclusión de la prueba:** Pendiente de ejecución.
- **Evidencia de ejecución:** Pendiente de ejecución.

### Caso CP-14: Límite de la memoria de 10 mensajes

- **Revisión asociada:** RC-04
- **Workflow y etapas involucradas:** 7.0 · Agente + Simple Memory
- **Qué se busca comprobar:** Qué ocurre cuando la conversación supera la ventana de 10 mensajes sin completar el lead
- **Condiciones iniciales:** 7.0 activo; cuenta de prueba; conversación sin cerrar
- **Entrada exacta:** Once mensajes breves que no completen el correo hasta el último, por ejemplo: saludo, nombre, interés, ubicación, presupuesto, plazo, pregunta, aclaración, confirmación, “sigo pensando” y el correo en el mensaje 11
- **Resultado esperado:** El agente conserva los datos o vuelve a pedirlos; no guarda un lead parcial ni inventa el correo
- **Criterio de aceptación:** Sin fila parcial en Baserow y comportamiento del agente documentado
- **Efecto que no debe ocurrir:** Guardado con datos faltantes o pérdida silenciosa del nombre
- **Evidencia que se buscará:** Transcripción del chat, ejecuciones y tabla de leads
- **Cómo se preparará de forma controlada:** Cuenta de prueba; borrar las filas al terminar
- **Resultado observado:** Pendiente de ejecución.
- **Conclusión de la prueba:** Pendiente de ejecución.
- **Evidencia de ejecución:** Pendiente de ejecución.

## 5. Pendientes para la sustentación

| ID | Revisión asociada | Pendiente y consecuencia de no resolverlo | Prioridad y motivo | Responsable | Criterio de cierre y caso asociado | Estado |
|---|---|---|---|---|---|---|
| P-01 | RC-07, RC-15 | Asignar el `errorWorkflow` del 7.0 al 6.3. Sin esto, los errores no controlados no se registran ni avisan. | Alta — error silencioso | Javier Argoty | CP-06 con fila en incidentes | Pendiente |
| P-02 | RC-03 | Validar que `message.text` no está vacío antes del agente. Sin esto, una foto o sticker deja al agente con entrada vacía. | Alta — error sin respuesta | Javier Argoty | CP-03 sin error técnico y sin fila | Pendiente |
| P-03 | RC-08 | Conectar `Fallo del agente` a un aviso al chat y/o al registro de incidentes; corregir el texto obsoleto (“consulta de prueba”, “inventario”). | Alta — error sin respuesta | Javier Argoty | CP-13 con aviso y sin fila | Pendiente |
| P-04 | RC-11, RC-14 | Agregar timeout al HTTP de guardado y `onError` al HTTP de incidentes. Sin esto, un cuelgue o un fallo del incidente deja al usuario sin aviso. | Alta — pérdida de trazabilidad | Javier Argoty | CP-05 y CP-12 con aviso o traza | Pendiente |
| P-05 | RC-13 | Manejar el estado parcial: si el lead se guardó y falla la confirmación, registrar o avisar. | Media — confusión del usuario | Simón Cruz | CP-11 con fila y traza | Pendiente |
| P-06 | RC-16 | Idempotencia o documentar los duplicados. `update_id` está disponible y no se usa. | Media — filas duplicadas | Luis Sinisterra | CP-07 con una sola fila o impacto documentado | Pendiente |
| P-07 | RC-18 | Agregar secret token o autenticación al webhook. Sin esto, cualquiera con la URL inyecta leads. | Alta — dato no autorizado | Javier Argoty | CP-10 rechaza el POST sin secret | Pendiente |
| P-08 | RC-10 | Corregir la regla `es_empresarial`: hoy solo excluye gmail, hotmail y outlook. | Media — clasificación incorrecta | Javier Argoty | CP-09 clasifica yahoo/icloud como personal | Pendiente |
| P-09 | RC-12 | Usar `$json.formMode` en el body del HTTP; hoy manda `'test'` fijo. | Media — dato de prueba en producción | Javier Argoty | CP-01 con `formMode=telegram` | Pendiente |
| P-10 | RC-17 | Reiniciar la memoria cuando el mismo chat inicia un lead nuevo. | Media — mezcla de datos | Javier Argoty | CP-08 sin mezclar datos | Pendiente |
| P-11 | RC-11, RC-12 | Confirmar la tabla de incidentes: la C08 usó `Incidentes` (1281229) y el 7.0 usa `1282332`. Confirmar nombre y columnas, y corregir el workflow/nodo hardcodeados en el incidente (dicen 6.1 y “HTTP · Guardar fila Baserow”). | Alta — trazabilidad | Luis Sinisterra | CP-05 con tabla y metadatos correctos | Pendiente |
| P-12 | RC-15 | Actualizar el diagrama del Corte 3 o registrar el canvas como referencia. | Baja — documentación | Johan Hernández | Diagrama en `04_Diagramas` o enlace | Pendiente |
| P-13 | RC-13, RC-14 | Agregar `onError` a los nodos de Telegram (`Responder conversación`, `Aviso de error`). | Media — respuesta perdida | Javier Argoty | CP-11/CP-12 con traza | Pendiente |
| P-14 | RC-04 | Documentar los límites del agente: ventana de 10 mensajes y `maxIterations: 4`. | Baja — límite conocido | Luis Sinisterra | CP-14 con comportamiento documentado | Pendiente |
| P-15 | RC-01 | Confirmar que el nodo `PrepararLead'` apunta al 6.2 correcto (`GxDZlth0DF976v72`). | Media — la clasificación no se ejecuta | Luis Sinisterra | CP-01 ejecuta el 6.2 sin error | Pendiente |

**Cambios realizados durante esta clase:** Ninguno. La revisión es del diseño actual del 7.0. El `errorWorkflow` sigue en `None` hasta aplicar P-01.

## 6. Revisión entre integrantes del grupo

| Integrante | Etapas o situaciones revisadas | Observación o duda identificada | Acuerdo del grupo y pendiente asociado |
|---|---|---|---|
| Johan Hernández | Recorrido completo y diagrama | No hay diagrama actualizado del Corte 3; solo existen los del Corte 1 y 2 | Registrar el canvas como referencia y actualizar el diagrama (P-12) |
| Javier Argoty | `Parsear mensaje`, `Contrato completo?`, `Fallo del agente`, `errorWorkflow`, `formMode` | El `errorWorkflow` del 7.0 está en `None` y el HTTP manda `formMode: 'test'` fijo | Aplicar P-01, P-02, P-03 y P-09 antes de la sustentación |
| Luis Sinisterra | HTTP Guardar, ruta de incidentes, tabla `1282332` | No está confirmado el nombre de la tabla ni la opción `Telegram` en `canal` | Verificar antes de CP-01 (P-11) |
| Simón Cruz | Confirmación al chat, memoria por sesión, duplicados | Si el túnel cambia, hay que re-registrar el webhook; la memoria por chat se conserva | Documentar la limitación y ejecutar CP-07/CP-11 (P-05, P-06) |

### Comprobación final del equipo

- [x] Incluimos todos los workflows relacionados y las ramas del proceso.
- [x] Revisamos todas las etapas y registramos las situaciones pertinentes.
- [x] Justificamos los casos que no aplican y reconocimos lo que falta revisar.
- [x] Localizamos las protecciones existentes y distinguimos las brechas.
- [x] Consideramos la respuesta al usuario y el efecto real en los destinos.
- [x] Los casos previstos contienen condiciones, entradas y criterios concretos.
- [x] Los pendientes tienen responsable y una forma de comprobar su cierre.
- [x] Registramos las observaciones y acuerdos entre integrantes.
- [x] Los resultados de las pruebas siguen pendientes de ejecución, sin resultados inventados.
- [x] Las capturas y los ejemplos no exponen contraseñas, tokens ni datos personales reales. No se adjuntan capturas en esta entrega; los ejemplos son ficticios.

## 7. Conclusión de la clase

**¿Qué permite afirmar la revisión del diseño actual?**

Que el 7.0 tiene protecciones localizadas en recepción, parseo, validación, clasificación y guardado, y que las reglas de negocio viven en el 6.2 y no en el modelo. No permite afirmar que el flujo funcione end-to-end: en esta clase no se ejecutaron las pruebas.

**¿Cuáles son las brechas más importantes y por qué?**

P-01: el `errorWorkflow` del 7.0 está en `None`, así que cualquier error no controlado queda sin registro ni aviso. P-03: `Fallo del agente` no tiene salida, por lo que el usuario no recibe respuesta si Gemini falla. P-04: si falla el registro del incidente, tampoco hay aviso. P-07: el webhook no tiene autenticación. P-09: el `formMode` sale como `test` aunque el flujo calcule `telegram`. P-11: la tabla de incidentes no está confirmada (1282332 frente a 1281229) y el registro hardcodea el workflow 6.1.

**¿Qué debe quedar listo y qué deberán demostrar en la sustentación?**

Aplicar P-01, P-02, P-03, P-04, P-07, P-09, P-11 y P-13. Ejecutar CP-01 a CP-14 y aportar las evidencias correspondientes. Explicar P-05, P-06, P-08 y P-10 con su consecuencia y la mitigación prevista.

**¿Qué limitaciones o dudas siguen abiertas?**

La estabilidad de la salida del agente ante variaciones de redacción no se ha medido con múltiples ejecuciones. La caída del túnel obliga a re-registrar el webhook. La memoria de 10 mensajes puede perder datos en conversaciones largas. La regla de `es_empresarial` solo contempla tres dominios personales.
