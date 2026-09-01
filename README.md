# Checkpoint 4 — Sincronización del Cerebro Agéntico con Ecosistemas de Negocio

**Valentín García del Cueto** · Curso de IA y Automatizaciones con n8n — Coderhouse

Workflow de n8n que conecta un agente de IA de soporte con tres herramientas reales de una tienda e-commerce: **Gmail** (casilla de soporte), **Salesforce** (CRM) y **Slack** (canal del equipo de operaciones), las tres autenticadas vía OAuth2.

![Workflow completo](evidencias/01-canvas-workflow.png)

---

## Qué hace

Cuando entra un mail a la casilla de soporte, el workflow:

1. **Descarta los correos automáticos** (auto-replies, fuera de oficina, rebotes, `no-reply@`) para no entrar en un bucle infinito de respuestas.
2. **Limpia y blinda el mensaje** antes de que lo vea el agente: saca el HTML, corta los adjuntos, trunca el texto y neutraliza intentos de prompt injection.
3. **Recupera el historial del cliente** desde Airtable, así el agente sabe si es la primera vez que escribe o si ya hubo interacciones previas.
4. **El agente de IA clasifica y redacta**: le asigna categoría y prioridad al ticket y escribe un borrador de respuesta.
5. **Sincroniza el contacto en el CRM**: primero lo busca por email con una query SOQL y recién después decide si lo actualiza o lo crea.
6. **Guarda la respuesta como borrador en Gmail**, nunca la manda sola. Un humano tiene que revisarla y aprobarla.
7. **Avisa al canal de Slack** con un resumen corto del ticket.
8. **Actualiza la memoria del cliente** en Airtable para la próxima vez que escriba.

---

## El flujo

```
[Gmail Trigger: mail entrante]
        │
        ▼
① [IF — ¿es auto-reply?] ── Sí ─▶ (corta el bucle)
        │ No
        ▼
   [Blindaje del payload]  →  [Memoria: Airtable]
        │
        ▼
   [AI Agent: clasifica y redacta]
        │
        ▼
④ [Set — limpia y valida el payload]
        │
        ▼
② [Look up SOQL — ¿el contacto ya existe?]
   ┌────┴─────┐
  Sí           No
   ▼            ▼
[Update]    [Create]
        │
        ▼
③ [Create Draft — espera aprobación humana]
        │
        ▼
④ [Set — payload liviano]  →  [Slack]  →  [Actualiza memoria]
```

---

## Las cuatro compuertas de seguridad

| # | Nodo | Para qué sirve |
|---|------|----------------|
| ① | `IF - Filtro Anti Auto-Reply` | Frena el bucle infinito de auto-respuestas. 18 condiciones sobre asunto, remitente y cabeceras RFC 3834. |
| ② | `Salesforce - Look Up Contacto` | Busca antes de crear. Salesforce **no tiene upsert** de contactos, así que sin este paso cada mail generaría un duplicado. |
| ③ | `Gmail - Create Draft (HITL)` | Human-in-the-loop. El mail queda en Borradores; no existe ningún nodo `Send` en todo el workflow. |
| ④ | `Set - Payload Limpio CRM` | Deja solo los campos necesarios y descarta binarios. Evita el Error 400 y no satura Slack. |

### ① El filtro anti auto-reply, funcionando

Un correo con asunto `Auto-reply: fuera de la oficina` entra al workflow y muere en `STOP - Bucle Cortado`. No llega al agente, no toca el CRM, no genera respuesta.

![Filtro anti auto-reply](evidencias/02-if-anti-autoreply-stop.png)

### ② Look up antes del Create

El contacto se crea una sola vez. Las corridas posteriores del mismo remitente entran por la rama Update.

![Contacto en Salesforce](evidencias/05-contacto-salesforce.png)

### ③ Human-in-the-loop

La respuesta del agente queda en la bandeja de Borradores, con el pie que indica categoría, prioridad y vencimiento de SLA. Nadie la envía hasta que un humano aprieta Enviar.

![Borrador en Gmail](evidencias/03-borrador-gmail-hitl.png)

### ④ Payload limpio hacia Slack

El canal recibe texto plano acotado: sin adjuntos, sin objetos anidados, sin el cuerpo completo del mail.

![Notificación en Slack](evidencias/06-mensaje-slack-ops.png)

---

## Memoria persistente

Cada remitente tiene una única fila en Airtable, identificada por `Session_ID` (su email). La escritura es un upsert, no un insert: por más veces que escriba el mismo cliente, la fila es siempre la misma y el contador de interacciones se incrementa.

![Memoria en Airtable](evidencias/04-memoria-airtable.png)

---

## Mínimo privilegio

- **Gmail** → `gmail.readonly` + `gmail.compose`. Sin permiso de envío, sin descarga de adjuntos.
- **Slack** → `chat:write` + `channels:read`. Sin subida de archivos ni lectura de usuarios.
- **Airtable** → PAT acotado a la base de memoria: `data.records:read`, `data.records:write`, `schema.bases:read`.
- **Salesforce** → la credencial de n8n solicita el scope `full` de forma fija y el campo no es editable desde la interfaz gráfica. Es una limitación del conector, no una decisión de diseño: al acotar los ámbitos únicamente a `api` + `refresh_token`, la autorización falla con `invalid_scope`.

### Autenticación OAuth2 verificada

Las cuatro credenciales, con el semáforo de conectividad en verde.

| Gmail | Salesforce |
|---|---|
| ![Gmail](evidencias/07-credencial-gmail-verde.png) | ![Salesforce](evidencias/08-credencial-salesforce-verde.png) |

| Slack | Airtable |
|---|---|
| ![Slack](evidencias/09-credencial-slack-verde.png) | ![Airtable](evidencias/10-credencial-airtable-verde.png) |

---

## Defensas adicionales

Más allá de lo que pide la rúbrica, el workflow hereda del Módulo 3 dos capas que se resuelven en Code nodes deterministas, nunca delegadas al LLM:

- **Anti prompt-injection**: un filtro con 9 patrones escanea asunto y cuerpo antes de que el agente los vea. Si detecta algo, neutraliza el contenido y escala el ticket a prioridad ALTA.
- **Anti inyección SOQL**: el email se sanitiza de comillas antes de entrar a la query de Salesforce.
- **Taxonomía cerrada**: categoría y prioridad se validan contra una lista fija de valores permitidos. Si el modelo devuelve algo fuera de esa lista, cae a un valor por defecto.
- **Fallback de parseo**: si el agente devuelve texto libre en vez de JSON, el workflow sigue funcionando con una respuesta genérica en lugar de romperse. El borrador siempre se genera.
- **SLA determinista**: el vencimiento se calcula con Luxon en un Code node según la prioridad (4h / 24h / 72h). El modelo no interviene.

---

## Cómo importarlo

1. En n8n: listado de workflows → **Import from File** → `checkpoint4_valentin_garciadelcueto.json`.
2. Las credenciales vienen referenciadas por ID. Si se importa en otra instancia, hay que volver a seleccionarlas.
3. Probar nodo por nodo con **Test step** antes de activar el workflow.

---

## Stack

n8n (self-hosted, Docker) · Gmail · Salesforce · Slack · Airtable · Groq (`gpt-oss-20b` para el agente, `llama-3.1-8b-instant` para los resúmenes) · Luxon

---

Este workflow es la continuación del Módulo 3 (memoria persistente híbrida) y forma parte del proyecto integrador que evoluciona módulo a módulo hasta el Proyecto Final.
