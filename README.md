# Proyecto final - n8n y AI Agent

## De que se trata

El flujo recibe una consulta de una persona interesada en una propiedad y hace una primera precalificacion. La idea es ordenar el ingreso de leads, reunir la informacion disponible y dejar una respuesta lista para el equipo comercial.

No intenta cerrar una operacion ni confirmar una visita. Cuando la consulta no alcanza o aparece un error, queda marcada para pedir datos o revisar con una persona.

## ROI estimado

Leer una consulta, revisar la informacion disponible y decidir como seguir lleva aproximadamente entre 5 y 8 minutos. Con unas 40 consultas por semana son alrededor de 4 horas de trabajo repetido.

El workflow hace ese primer filtro en segundos. En un volumen normal puede ahorrar unas 3 horas semanales y, si llegan mas consultas, mantiene el mismo criterio sin sumar otra tarea manual. Para este prototipo el costo de implementacion es principalmente el tiempo de armado y configuracion; tomando 6 horas de trabajo a un valor estimado de USD 10 por hora, son unos USD 60 iniciales. Los casos comerciales especiales siguen necesitando revision humana.

## Como queda armado

Webhook -> Edit Fields -> Google Sheets -> Code in JavaScript -> If

Desde el If salen dos caminos:

- datos validos: AI Agent -> Respond to Webhook
- datos incompletos o problema con la fuente: Respond to Webhook

Google Sheets aporta una segunda fuente con datos comerciales. El Code ordena las filas, conserva el mensaje original, identifica el lead y calcula si ya estan presentes el tipo de propiedad, la zona y el presupuesto.

El agente usa Gemini, memoria simple por lead y una Simple Vector Store con `models/gemini-embedding-2`.

![Canvas del workflow](capturas/flow-n8n.png)

## Entrada y salida

El webhook recibe por GET los campos `lead_id` y `mensaje`.

Ejemplo de prueba:

`/webhook-test/precalificacion-leads?lead_id=validacion-001&mensaje=Busco%20un%20departamento%20de%202%20ambientes%20en%20Caballito%20con%20presupuesto%20de%20150000%20USD`

La respuesta es JSON con:

- `clasificacion`: caliente, tibio o frio
- `accion`: contactar_agente, enviar_opciones, pedir_datos o revision_humana
- `mensaje`: texto breve para continuar la conversacion
- `confianza`: valor entre 0 y 1

En una prueba completa el flujo devolvio una clasificacion y una accion concreta. En una prueba sin mensaje devolvio una respuesta de pedido de datos, sin ejecutar la etapa de IA.

## Eleccion del modelo

Para este prototipo se eligio Gemini en la nube. El volumen esperado es bajo y permite probar el agente sin mantener una maquina con Ollama encendida. Tambien simplifica la conexion con embeddings y el uso de n8n Cloud.

La contra es depender de internet y del proveedor. Por eso el prompt limita lo que puede afirmar y el flujo agrega reintentos y continuidad en Google Sheets.

## Por que un AI Agent

Una cadena simple podria redactar un texto, pero en este caso el nodo tiene que elegir entre varias acciones y consultar contexto antes de responder. Por eso se usa un AI Agent conectado a Google Sheets, memoria y la herramienta de recuperacion.

## RAG y memoria

La Simple Vector Store se usa como herramienta del agente. Los embeddings de Gemini permiten buscar los fragmentos relacionados con la consulta. En este flujo se usa el modelo `models/gemini-embedding-2`, que es el disponible en la cuenta de n8n.

La carga documental esta separada en [workflow-ingesta-rag.json](workflow-ingesta-rag.json). Ese flujo usa Manual Trigger, Default Data Loader, Recursive Character Text Splitter, Simple Vector Store y el mismo modelo de embeddings. Primero se ejecuta la ingesta y despues se prueba el workflow principal.

La memoria simple usa `lead_id` como clave de sesion y guarda las ultimas interacciones de cada lead.

## System Prompt

```text
Sos un agente de precalificacion comercial para una inmobiliaria. La entrada incluye datos calculados por nodos anteriores.

Regla prioritaria: si datos_completos=true, consideralo una validacion confiable. En ese caso esta prohibido clasificar como frio, usar pedir_datos o volver a preguntar tipo de propiedad, zona o presupuesto. Debes clasificar como tibio y usar enviar_opciones, o como caliente y usar contactar_agente.

Si datos_completos=false, usa pedir_datos solo por los datos que falten. Analiza tambien Google Sheets y el contexto RAG, pero no contradigas la validacion previa salvo que encuentres una contradiccion clara. Si la base documental no tiene coincidencias, continua con los datos disponibles.

No inventes precios, disponibilidad, requisitos ni caracteristicas. No confirmes visitas ni prometas operaciones. Responde en español, claro y breve.

Devuelve solo JSON valido, sin markdown, con clasificacion, accion, mensaje y confianza. La confianza debe ser un numero entre 0 y 1.
```

## Controles y errores

Google Sheets tiene reintentos y queda configurado para continuar con el mensaje de error como salida normal. El nodo Code revisa si llegaron filas y si alguna contiene un error.

El If valida el mensaje y el estado de la hoja antes de dejar pasar la consulta al agente. Si no hay mensaje, si la hoja esta vacia o si la API devuelve un error, la ejecucion no queda esperando y la respuesta se mantiene en el camino de revision.

El prompt tambien funciona como limite: no inventa informacion, no confirma visitas y deriva los casos ambiguos. Antes de responder, el flujo valida los campos del JSON y reemplaza clasificaciones invalidas por revision_humana. Esto evita que una salida del modelo termine confirmando una visita o una disponibilidad que no fueron verificadas.

![Resultado de prueba](capturas/resultado-prueba.png)

## Archivos

- [workflow-agente-precalificacion.json](workflow-agente-precalificacion.json)
- [workflow-ingesta-rag.json](workflow-ingesta-rag.json)
- [resultado de prueba en JSON](capturas/resultado-prueba.json)
- [diagrama del flujo](capturas/arquitectura.svg)

Las credenciales y los identificadores reales no se incluyen en el repositorio. Al importar el workflow hay que seleccionar las credenciales propias de Google Sheets y Gemini, y revisar el documento de la hoja.

## Como probarlo

1. Importar el JSON en n8n.
2. Seleccionar las credenciales de Google Sheets y Gemini.
3. Revisar el documento y el nombre de la hoja.
4. Ejecutar el workflow y llamar al webhook de prueba.
5. Revisar el JSON que devuelve Respond to Webhook.

No se suben claves, tokens ni datos personales.
