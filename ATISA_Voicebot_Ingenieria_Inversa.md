# ATISA Voicebot — Ingeniería Inversa

Un voicebot de calificación de leads sobre Vapi + n8n. Así está armado por dentro, y por qué se cae.

**239 nodos · 4 workflows · 6 estados · 7 servicios externos**

---

## 1. El sistema en una frase

Voicebot de calificación de leads inmobiliarios/comerciales para ATISA. Vapi.ai maneja la voz; n8n maneja toda la lógica. Atiende la llamada, identifica al cliente en Supabase, lo enruta a un "agente especializado" según su etapa (flow_step), extrae datos de la conversación con IA y los sincroniza con Zoho CRM. Si hace falta, transfiere a un humano vía Twilio.

### Revelaciones clave

- **No usa una sola plataforma:** hay 2 cuentas de Twilio (ATISA/"TendencIA" y "American Phone") y un endpoint propio en api.tendencia.ai. El proveedor M Consulting es TendencIA.
- **Los "agentes Navi" NO son IA:** son prompts de texto guardados en nodos Set que n8n le devuelve a Vapi. Vapi corre la conversación; n8n solo le pasa el prompt y las tools.
- **La IA real (modelos de lenguaje) solo corre DESPUÉS de la llamada,** para extraer variables y resumir.
- **Mezcla de modelos sin criterio:** Gemini 2.5 Flash, GPT-4.1, GPT-4o, o4-mini y Claude 3.7 Sonnet. Notas de los devs dicen "Probar con Gemini" / "Probar con GPT 4.1 o o4 mini" / "REVISAR TIEMPO": se entregó experimental.

---

## 2. Stack real

- **Voz:** Vapi.ai (eventos assistant-request, end-of-call-report)
- **Orquestación:** n8n (4 workflows, 239 nodos)
- **BD:** Supabase (tablas users, call_flows, user_property_interests, zoho_updates, call_logs, properties)
- **CRM:** Zoho (30 llamadas HTTP, versiones v2/v2.1/v7/v8 mezcladas)
- **Telefonía:** Twilio (2 cuentas: México y EE.UU.)
- **IA post-llamada:** OpenRouter + OpenAI
- **Endpoint propio:** api.tendencia.ai/webhooks/vapi

**Conteo por workflow:** Call Control 78 · Agents B2B 98 · End of Call 42 · Schedule Meeting 21.

---

## 3. Modelo de datos + Riel de estados

Todo gira en torno a `call_flows.flow_step` (entero) = etapa del embudo.

| flow_step | Estado |
|-----------|--------|
| 1 | Nuevo Cliente |
| 2 | Inmuebles |
| 3 | Comercial Renta |
| 4 | Industrial |
| 5 | Cliente Existente |
| 6 | Comercial Compra |

Cada estado carga un prompt "Set prompt …".

> **Nota:** "Transferencia a Humano" NO es un flow_step; se dispara por la tool `transfer-human`.

---

## 4. Los cuatro flujos

### Call Control — el recepcionista
**Trigger:** webhook POST /fd902016… (Vapi al entrar la llamada).

Verifica assistant-request → detecta número (ATISA vs secundario) → Get User + Get Call Flow en Supabase → If userExists (existe: revisa reincidencia <20 min; no existe: busca/crea Lead en Zoho + usuario) → Switch "Flow Path" por flow_step carga el prompt → responde a Vapi.

**Banderas rojas:** dos "Create Lead" (Zoho v7 y v2); control de horario con bug conocido (nota "REVISAR TIEMPO"); nodo Wait de 1s = parche de race condition.

### Agents B2B — el cerebro post-llamada y transferencias
**Trigger:** webhook POST /9b04a605…, lo invocan las tools atisa-transfer / transfer-human.

Pide GET api.vapi.ai/call?limit=15 y trata de adivinar la llamada correcta → identifica cuenta Twilio (MN México / USN EE.UU.) → IA: "Obtain summary" (Gemini) y "Output Variables" (Gemini, parser auto-reparable) extrae JSON {users_update, property_interests_update, call_flows_update{next_step, local_variables}, var_updated} → actualiza Supabase → clasifica proyecto ("Plaza Pacífico" / "Centro Comercial Pacífico", o4-mini) → actualiza Zoho por layout (Habitacional/Comercial Compra/Renta/Industrial, endpoints distintos MN y USN) → transfiere vía Twilio.

**Banderas rojas:** emparejamiento de llamada frágil (limit=15); TODA la lógica duplicada MN/USN; modelos distintos para la misma tarea; si el parser no repara el JSON, se cae el flujo.

### Schedule Meeting — agendamiento
**Trigger:** webhook POST /447b8622…, tool atisa-schedule-meeting.

Lee usuario + zoho_updates + intereses → If: no hay cita → crea Contacto (v8) + Evento (v8) + notas; ya hay cita → actualiza Evento (v2.1) → marca en Supabase. Un solo agente IA (Claude 3.7 Sonnet) para interpretar fecha/hora.

**Bandera roja:** mezcla 3 versiones de la API de Zoho (v2, v2.1, v8) en un mismo flujo.

### End of Call — cierre
**Trigger:** webhook POST /cd37b32b…, filtra type == end-of-call-report.

Evita doble proceso si fue atisa-transfer; si el usuario colgó (hang-up) igual guarda la transcripción. IA: GPT-4.1 + GPT-4o + Claude 3.7 en paralelo → actualiza call_flows/users/intereses + notas en Zoho. Wait de 3s (el más largo).

**Bandera roja:** tres modelos de IA distintos en un solo flujo de cierre (costo, latencia y puntos de falla x3).

---

## 5. Diagnóstico de inestabilidad

### 8.1 — CRÍTICO — Emparejamiento de llamadas frágil
Usa GET /call?limit=15 y adivina cuál es; con llamadas simultáneas agarra la equivocada o no la halla → desconexiones y datos cruzados.
*Fix:* usar el call.id que Vapi ya manda en el webhook.

### 8.2 — ALTO — Parches de Wait fijos
5 nodos (1s×4, 3s) para "esperar" sincronizaciones; no garantizan nada → síntoma de race condition no resuelta.

### 8.3 — ALTO — Lógica duplicada MN/USN
Ramas copy-paste; los arreglos se aplican en una y se olvidan en la otra → comportamiento inconsistente según el número.

### 8.4 — ALTO — Versiones de la API de Zoho mezcladas (v2/v2.1/v7/v8)
Al deprecarse una, fallan solo algunos nodos → fallos intermitentes difíciles de diagnosticar.

### 8.5 — MEDIO — Dependencia de parsers de IA "auto-reparables"
Si la reparación falla, se cae el flujo; modelos distintos varían el formato y agravan esto.

### 8.6 — MEDIO — Estado mutable sin transacciones
El flow_step se lee, procesa y reescribe; dos eventos casi simultáneos se pisan → cliente en estado incorrecto. Sin locking ni idempotencia real.

### 8.7 — MEDIO — Indecisión técnica documentada por los propios devs
"Probar con Gemini", "Probar con GPT 4.1 o o4 mini", "REVISAR TIEMPO" → se entregó experimental.

---

## 6. Mapa de disparadores

| Disparador | Workflow | Webhook |
|------------|----------|---------|
| Llamada entrante (assistant-request) | Call Control | /fd902016… |
| Tool atisa-transfer / transfer-human | Agents B2B | /9b04a605… |
| Tool atisa-schedule-meeting | Schedule Meeting | /447b8622… |
| end-of-call-report | End of Call | /cd37b32b… |

---

## 7. Recomendaciones

1. **Rediseñar el emparejamiento de llamadas** (usar call.id del webhook, no buscar entre 15). Mayor impacto.
2. **Eliminar los Wait fijos;** resolver sincronización con confirmaciones reales o idempotencia.
3. **Unificar MN/USN** en una sola rama parametrizada.
4. **Congelar una sola versión de la API de Zoho** y un solo modelo de IA por tarea.
5. **Evaluar si 239 nodos** en 4 workflows acoplados por estado compartido justifican seguir en n8n o conviene migrar la lógica a código.

---

## Cierre — dependencia del proveedor

El endpoint api.tendencia.ai/webhooks/vapi y las credenciales de Twilio/Vapi están atadas a la cuenta del proveedor (TendencIA / M Consulting). Antes de cualquier decisión, confirmar qué infraestructura es de ATISA y cuál del proveedor, porque eso define si pueden operar el sistema sin ellos.
