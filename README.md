# AI Customer Support

Sistema de automatizacion de atencion al cliente con IA, base de conocimiento, supervision humana y notificaciones multicanal.

## Objetivo

El prototipo recibe consultas desde Google Forms, las almacena en Google Sheets, recupera informacion oficial desde Airtable, utiliza Google Gemini para clasificar y redactar una respuesta, registra el resultado y solicita aprobacion humana por Gmail. Ademas, publica una notificacion interna en Slack y conserva su identificador.

## Stack tecnologico

- **Orquestacion:** Make.
- **Entrada:** Google Forms y Google Sheets.
- **Base de datos:** Airtable (Clientes, Consultas y Base de conocimiento).
- **IA:** Google Gemini 3.1 Flash-Lite.
- **Canales:** Gmail (aprobacion) y Slack (notificacion interna).

## Arquitectura

```text
Google Forms -> Google Sheets Watch Rows
             -> Airtable Search Records (Base de conocimiento activa)
             -> Tools Text Aggregator
             -> Gemini 3.1 Flash-Lite (temperature 0.2; max output tokens 400)
             -> Airtable Create Record (consulta y cliente vinculado)
             -> Slack #consultas-clientes
             -> Airtable Update Record (guarda Slack Timestamp)
             -> Gmail (solicitud de aprobacion humana)

Error Handler Gemini -> Airtable (detalle del error, Estado=Error, KPI=1) -> Retry

Escenario Human Approval:
Gmail Watch Emails -> filtro APROBAR -> Text Parser (Record ID)
-> Airtable Search Records -> Gmail (respuesta de demostracion al aprobador)
-> Airtable Update Record (Estado=Enviado)
```

## Base de datos

### Clientes
Nombre, email, telefono y fecha de registro. Los registros se relacionan con Consultas; se verifico el vinculo desde Make utilizando Smart Links.

### Consultas
Nombre de consulta, cliente relacionado, mensaje original, fecha de recepcion, categoria, prioridad, respuesta IA, estado, email de clientes, Slack Thread ID, Indicador de error y Indicador de error KPI. La ruta normal coloca KPI=0 y la ruta de error coloca KPI=1.

### Base de conocimiento
Titulo, categoria, pregunta frecuente, respuesta oficial, palabras clave y estado. El escenario principal recupera los registros activos.

## Configuracion de Gemini

- Modelo: `gemini-3.1-flash-lite`
- Temperature: `0.2`
- Max Output Tokens: `400`
- Salida estructurada: `categoria`, `prioridad`, `respuesta`.

## Resiliencia

El Error Handler de Gemini registra el mensaje del error en Airtable, marca el Estado como `Error` y `Indicador de error KPI` como `1`. La ruta normal registra KPI `0`. Se habilito el almacenamiento de ejecuciones incompletas y se configuro Retry con tres intentos y un intervalo de un minuto. Se probo un fallo de API y se verifico que Airtable guardara el detalle; luego se restauro la configuracion normal.

## Dashboard de control

[Ver Dashboard de Control en Airtable](https://airtable.com/appuhrzpTjSMatviX/shrd47AtbLkzw9nta)

La vista publica esta agrupada por Estado y muestra fecha, categoria, prioridad, estado e indicador KPI. Se ocultaron nombre, email, mensaje original, respuesta IA y Slack Thread ID.

## Pruebas funcionales

Se probaron consultas de envios, pagos, preguntas no respaldadas por la base de conocimiento, datos incompletos, fallo controlado de Gemini, aprobacion humana y la notificacion de Slack con registro del Timestamp. El usuario informo que todas las pruebas ejecutadas finalizaron correctamente.

## Limitaciones conocidas

- La ruta automatica `RECHAZAR` no esta implementada; el flujo probado cubre la aprobacion `APROBAR`.
- El correo de salida posterior a la aprobacion se envia al responsable/aprobador como demostracion, no al cliente final.
- El filtro de aprobacion busca `APROBAR` en el cuerpo completo del correo y puede coincidir con texto citado.
- Se guarda el Timestamp de Slack, pero la respuesta posterior dentro del hilo no forma parte del flujo probado.
- Google Sheets Watch Rows sigue el punto de inicio y las filas nuevas; el limite de dos bundles es capacidad por ejecucion de prueba, no deduplicacion por si mismo.

## Archivos de entrega sugeridos

- `Diagrama_Arquitectura_AI_Customer_Support.pdf` - arquitectura de los escenarios y rutas.
- `Manual_Operativo_y_Documentacion_AI_Customer_Support.pdf` - modelo de datos, JSON, resiliencia, seguridad, pruebas y costos.
- `Manual_Operativo_y_Documentacion_AI_Customer_Support.docx` - version editable.
- `Blueprint_Sistema_Principal.json` - blueprint exportado desde Make.
- `Blueprint_Human_Approval.json` - blueprint exportado desde Make.
- Capturas de los dos escenarios, Airtable, Slack y Gmail.
- Video de demostracion (3 minutos).

## Matriz de costos (estimacion)

Supuesto ilustrativo: 1.500 tokens de entrada + 300 tokens de salida por consulta; 1.000 consultas. Precios API estandar en USD por millon de tokens, revisados el 20/11/2026.

| Modelo | Input / 1M | Output / 1M | Estimado / 1.000 consultas |
|---|---:|---:|---:|
| Gemini 3.1 Flash-Lite (implementado) | $0.25 | $1.50 | $0.825 |
| Claude Haiku 5.5 (alternativa) | $0.10 | $0.50 | $0.300 |
| Claude Sonnet 5.5 (alternativa) | $2.00 | $10.00 | $6.000 |

El estimado excluye operaciones de Make, costos de otros servicios, impuestos y cualquier variacion del consumo real de tokens. Con los supuestos anteriores, Gemini cuesta aproximadamente 86.25% menos que Claude Sonnet 5.5; Claude Haiku 5.5 es mas economico.

### Fuentes de precios

- [Google Gemini API pricing](https://ai.google.dev/gemini-api/docs/pricing)
- [Anthropic Claude pricing](https://platform.claude.com/docs/en/about-claude/pricing)
