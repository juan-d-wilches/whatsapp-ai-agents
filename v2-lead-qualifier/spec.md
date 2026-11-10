# Zyprus WhatsApp Attention Channel — n8n Build Spec

**Owner:** David Wilches (Zyprus)
**Spec date:** 2026-10-18
**Audience:** the n8n-building Claude agent. Build, test and report back to David.

---

## 0. Instructions for the building agent

1. **Read this whole spec before building.** Follow n8n MCP best practice: SDK reference → node search → type definitions → validate each node → validate the workflow → create/update.
2. **Test as much as you can yourself** with the tools in your scope: validation, pinned data, test executions, data-table reads/writes, Discord test posts and the follow-up scheduler. Section 11 lists what you can test and how.
3. **When a test needs David** (a real WhatsApp message from his phone, the Meta dashboard, publishing the app), do not guess the result. Give him the exact step-by-step procedure from section 12, adapted to what you built, and wait for his results.
4. **Do not touch other workflows** in the instance. In particular, leave the barbershop (`barberia-demo`) and dental (`dental-demo`) workflows and credentials alone.
5. **Never message any WhatsApp number other than David's verified test recipient** during testing. Ask David for that number if you need it and keep it in `CONFIG.TEST_RECIPIENT`.
6. **Prefix every Discord message you send during testing with `[TEST]`.**
7. When you finish, report to David: what was built (workflow names + IDs), what you tested and the results, what he must test (with steps), and open questions.

---

## 1. Goal

An **inbound-only** WhatsApp channel for Zyprus. A business owner writes to the Zyprus number. An AI assistant:

1. explains what Zyprus does,
2. qualifies the lead with a few questions,
3. asks consent for David to contact them,
4. tells them **David (founder of Zyprus) will contact them personally**,
5. posts the lead to the Discord **leads** channel.

David then contacts the lead **personally, from his own WhatsApp**, using the link in the Discord alert. The bot never closes deals, quotes prices or books meetings.

Errors and incidents go to a separate Discord **alerts** channel.

The number never starts conversations. All outbound messages are replies or follow-ups **inside the 24-hour customer service window**. No templates, no messages after the window closes (budget decision).

---

## 2. Existing assets (verified on 2026-10-18)

| Asset | Name | ID / value |
| --- | --- | --- |
| n8n instance | self-hosted VPS | `https://<your-n8n-host>` |
| Existing workflow (active) | `Zyprus WhatsApp - Inbound` | `<id>` (folder `<id>`) |
| Its only node | `WhatsApp Trigger`, updates: `messages` | webhookId `<webhook-id>` |
| Credential: WhatsApp trigger | `Zyprus - WhatsApp Trigger` (whatsAppTriggerApi) | `<id>` |
| Credential: WhatsApp send | `Zyprus - WhatsApp Send (TEST)` (whatsAppApi) | `<id>` |
| Credential: Discord leads | `Zyprus - Attention Channel Leads` (discordWebhookApi) | `<id>` |
| Credential: Discord alerts | `Zyprus - Attention Channel Alerts` (discordWebhookApi) | `<id>` |
| Credential: LLM | `OpenRouter account` (openRouterApi) | `<id>` |
| Credential: Redis (optional) | `Redis account` | `<id>` |
| Data tables | none yet | create them (section 5) |
| Meta app | `Zyprus Attention Channel`, **Unpublished** | test phone number + test WABA |

### Constraints you must respect

- **One callback URL per Meta app.** The `WhatsApp Trigger` node registers its webhook URL with Meta when the workflow is activated. **Keep the existing trigger node in `<id>`** (extend that workflow). Never activate a second workflow containing a WhatsApp Trigger for this app, or it will overwrite the callback URL and break inbound. After any activation, ask David to confirm in Meta (Use cases → Connect on WhatsApp → Configuration → Webhooks) that the Callback URL still ends in `/webhook/<webhook-id>/webhook`.
- **The app is unpublished.** Until David publishes it, Meta only delivers **dashboard test webhooks**, not real messages. Real end-to-end tests depend on David publishing the app (needs a privacy policy URL).
- **The WhatsApp Trigger cannot be executed via MCP.** Put the logic in a sub-workflow (section 4) so you can test it with pinned data or a test harness.
- **The access token is temporary** (about 24 h). Auth errors (Graph error code `190`) are expected until David creates a permanent System User token. Handle them as alerts (section 9), do not crash silently.
- **Test recipients:** the test number can only message up to 5 verified recipient numbers. Sending to any other number fails (commonly error `131030`). Handle gracefully.

### Real trigger output shape (from execution 670, a Meta dashboard test)

```json
{
  "messaging_product": "whatsapp",
  "metadata": { "display_phone_number": "16505551111", "phone_number_id": "123456123" },
  "contacts": [
    { "profile": { "name": "test user name" }, "wa_id": "16315551181", "user_id": "US.13491208655302741918" }
  ],
  "messages": [
    {
      "id": "ABGGFlA5Fpa",
      "timestamp": "1504902988",
      "from": "16315551181",
      "from_user_id": "US.13491208655302741918",
      "type": "text",
      "text": { "body": "this is a text message" }
    }
  ],
  "field": "messages"
}
```

Notes:
- Key conversations by `wa_id` / `from`. Also store `user_id` (Meta is rolling out business-scoped user IDs and usernames), but do not depend on it yet.
- Status updates (sent/delivered/read) arrive with a `statuses` array and **no** `messages` array. Ignore them.
- Use `metadata.phone_number_id` from the payload when replying, and store it per conversation for follow-ups.

---

## 3. CONFIG (one Set node, or a small config data table)

| Key | Value | Notes |
| --- | --- | --- |
| `OWNER_NAME` | `David` | Used in bot messages and alerts |
| `BUSINESS_TZ` | `America/Bogota` | All human-facing times |
| `N8N_BASE_URL` | `https://<your-n8n-host>` | For execution links in alerts |
| `FU1_DELAY_MIN` | `120` | First follow-up, 2 h after the lead's last message |
| `FU2_DELAY_MIN` | `1200` | Second (final) follow-up, 20 h after the lead's last message |
| `WINDOW_HOURS` | `24` | Meta customer service window |
| `WINDOW_SAFETY_MIN` | `30` | Never send within the last 30 min of the window |
| `QUIET_START` / `QUIET_END` | `21:00` / `08:00` | No follow-ups in this range (Bogota time). Replies are always immediate |
| `DEBOUNCE_SECONDS` | `8` | Group bursts of short messages |
| `HISTORY_LIMIT` | `20` | Messages sent to the LLM as context |
| `LEAD_UPDATE_COOLDOWN_MIN` | `30` | Min gap between "lead update" posts |
| `CONTACT_PROMISE_ES` | `lo antes posible, normalmente el mismo día hábil` | David can edit |
| `DRY_RUN` | `false` (use `true` while testing) | When true, WhatsApp sends are replaced by a `[TEST][DRY_RUN]` post to the alerts channel with the would-be message |
| `TEST_RECIPIENT` | ask David | His verified personal number |

---

## 4. Architecture

| # | Workflow | Trigger | Purpose |
| --- | --- | --- | --- |
| WF1 | `Zyprus WhatsApp - Inbound` (existing, `<id>`) | WhatsApp Trigger (keep the existing node) | Thin router: filter, dedupe, hand each message to WF2 |
| WF2 | `Zyprus WA - Process Message` | Execute Workflow Trigger | All conversation logic: state, debounce, LLM, reply, lead alerts |
| WF3 | `Zyprus WA - Follow-ups` | Schedule, every 15 min | In-window follow-ups and partial-lead alerts |
| WF4 | `Zyprus WA - Error Handler` | Error Trigger | Posts any workflow failure to the alerts channel |
| WF5 (testing only) | `Zyprus WA - Test Harness` | Webhook (secret path) or manual | Calls WF2 with fake payloads. Keep inactive except while testing |

Set WF4 as the **error workflow** in the settings of WF1, WF2 and WF3. Put all of them in the same folder as WF1. Tag them `zyprus-attention-channel`.

---

## 5. Data tables (n8n Data Tables)

### 5.1 `wa_conversations` (one row per contact)

| Column | Type | Notes |
| --- | --- | --- |
| `wa_id` | string | Unique key |
| `user_id` | string | From payload, may be empty |
| `profile_name` | string | WhatsApp profile name |
| `phone_number_id` | string | Business number that received the message |
| `language` | string | `es` / `en` |
| `status` | string | `active`, `qualified`, `handed_off`, `opted_out`, `closed` |
| `first_seen_at` | date | |
| `last_inbound_at` | date | Time of the lead's last message (opens/refreshes the window) |
| `last_outbound_at` | date | Time of our last message |
| `window_expires_at` | date | `last_inbound_at + 24h` |
| `followups_sent_in_window` | number | Reset to 0 on every inbound message |
| `last_followup_at` | date | |
| `lead_notified` | boolean | Lead alert already posted |
| `lead_notified_at` | date | |
| `partial_lead_notified` | boolean | Partial alert already posted for the current window |
| `last_lead_update_at` | date | For the update cooldown |
| `human_requested` | boolean | |
| `pending_buffer` | string | Debounce buffer (JSON array of texts) |
| `pending_last_message_id` | string | Debounce: newest message id |
| `message_count` | number | Inbound messages |

### 5.2 `wa_leads` (one row per contact)

| Column | Type |
| --- | --- |
| `wa_id` | string (key) |
| `name` | string |
| `role` | string |
| `business_name` | string |
| `business_type` | string |
| `business_size` | string |
| `location` | string |
| `process_to_fix` | string |
| `current_handling` | string |
| `volume` | string |
| `urgency` | string |
| `budget` | string |
| `preferred_contact_time` | string |
| `contact_consent` | boolean |
| `summary` | string |
| `suggested_next_step` | string |
| `created_at` / `updated_at` | date |

### 5.3 `wa_messages` (conversation log, source of LLM history)

| Column | Type | Notes |
| --- | --- | --- |
| `wa_id` | string | |
| `message_id` | string | WhatsApp id (`wamid...`) for inbound; response id for outbound |
| `direction` | string | `in` / `out` |
| `kind` | string | `reply`, `followup_1`, `followup_2`, `system` |
| `type` | string | `text`, `audio`, `image`, ... |
| `text` | string | |
| `created_at` | date | |

Use `wa_messages` for **dedupe** too: if an inbound `message_id` already exists, stop.

Use data tables rather than in-process chat memory: history must survive VPS restarts and be readable by WF3.

---

## 6. WF1 — Inbound router (extend the existing workflow)

1. `WhatsApp Trigger` (existing; do not recreate).
2. **IF has messages:** continue only if `messages` exists and is non-empty. Otherwise end (status events).
3. **Split** `messages` into items (normally one), carrying `contacts[0]` and `metadata`.
4. **Normalize** (Set/Code) to:
   ```json
   { "wa_id": "...", "user_id": "...", "profile_name": "...", "phone_number_id": "...",
     "message_id": "...", "timestamp": 1504902988, "type": "text", "text": "..." }
   ```
   For non-text types, set `text` to `""` and keep `type`.
5. **Execute Workflow** → WF2, **without waiting** for it to finish, so Meta gets a fast 200.

---

## 7. WF2 — Process message (core logic)

### 7.1 Steps

1. **Dedupe:** look up `wa_messages` by `message_id`. If found → end.
2. **Log inbound** to `wa_messages` (`direction=in`).
3. **Upsert conversation:** create if new (`status=active`, `first_seen_at=now`). Always set `last_inbound_at=now`, `window_expires_at=now+24h`, `followups_sent_in_window=0`, `partial_lead_notified=false`, increment `message_count`, update `profile_name`, `user_id` and `phone_number_id`.
4. **Opted-out contacts:** if `status=opted_out` and the new text does not clearly re-engage, reply once with a short courtesy only if the last outbound message was more than 24 h ago; otherwise end. If they re-engage ("hola, quiero info"), set `status=active`.
5. **Non-text messages** (audio, image, video, document, sticker, location): reply with a fixed text and end, no LLM:
   > Por ahora solo puedo leer mensajes de texto. ¿Me lo puedes escribir? 🙏

   Reactions: ignore silently.
6. **Debounce (recommended):** append the text to `pending_buffer`, set `pending_last_message_id`, wait `DEBOUNCE_SECONDS`, re-read the row. If `pending_last_message_id` ≠ this message's id → end (a newer execution will answer). Otherwise take the buffer as the combined user turn and clear it. You may use Redis (`<id>`) for the buffer and lock instead if it is simpler. If debounce proves unreliable, ship without it and say so in the report.
7. **Build LLM context:** last `HISTORY_LIMIT` rows of `wa_messages` for this `wa_id` (oldest → newest), the current `wa_leads` row, the conversation `status`, `profile_name`, and the current Bogota date/time.
8. **LLM call** (Basic LLM Chain or AI Agent with no tools) via OpenRouter `<id>`, with the system prompt in section 8 and a **Structured Output Parser** (schema in 7.2). Use a fast, low-cost model that reliably returns JSON; confirm the model choice with David. Temperature around 0.3.
9. **Merge lead fields in code:** non-null LLM values overwrite stored values; null never erases. Upsert `wa_leads`.
10. **Compute flags deterministically** (do not trust the LLM alone):
    - `lead_complete = name && (business_type || business_name) && process_to_fix && contact_consent === true`
    - `opt_out`, `human_requested` from the LLM output.
11. **Status transitions:**
    - `opt_out` → `status=opted_out`. Send the LLM's polite closing. No alerts.
    - `human_requested` and not yet notified → post the lead alert with **URGENT** flag (even if incomplete), `status=handed_off`.
    - `lead_complete` and not yet notified → post the lead alert, `lead_notified=true`, `status=handed_off`.
    - Already `handed_off` and the user adds meaningful new info (LLM `lead_update_worth_posting=true`) and the cooldown has passed → post a short **lead update** to the leads channel.
12. **Send reply** with the WhatsApp node (credential `<id>`), `phoneNumberId` from the conversation, recipient `wa_id`. Respect `DRY_RUN`.
13. **Log outbound** to `wa_messages` (`direction=out`, `kind=reply`) and set `last_outbound_at`.
14. **Order on alert:** post the Discord alert **before** sending the "David te contactará" reply. If Discord fails, still reply to the user, and send the full lead payload to the alerts channel so the lead is never lost.

### 7.2 LLM structured output schema

```json
{
  "reply": "string, the WhatsApp message to send, max ~600 characters",
  "language": "es | en",
  "lead": {
    "name": "string|null",
    "role": "string|null",
    "business_name": "string|null",
    "business_type": "string|null",
    "business_size": "string|null",
    "location": "string|null",
    "process_to_fix": "string|null",
    "current_handling": "string|null",
    "volume": "string|null",
    "urgency": "string|null",
    "budget": "string|null",
    "preferred_contact_time": "string|null",
    "contact_consent": "boolean|null"
  },
  "human_requested": "boolean",
  "opt_out": "boolean",
  "off_topic": "boolean",
  "lead_update_worth_posting": "boolean",
  "summary": "string, 1-2 lines for David",
  "suggested_next_step": "string, 1 line for David"
}
```

If parsing fails: retry once. If it fails again, send the fallback text below and post an alert.
> Disculpa, tuve un problema técnico. ¿Me repites tu último mensaje? Si prefieres, David te puede escribir directamente.

---

## 8. Assistant behavior (system prompt)

Write the final prompt from this specification. Keep it in English with Spanish examples; the assistant answers in the user's language (default: Colombian Spanish, neutral and warm, "tú").

### 8.1 Identity and scope

- You are the AI assistant of **Zyprus**, an agency that helps businesses earn more and save time by implementing **AI and automations in their processes**: customer service on WhatsApp/Instagram, lead capture and follow-up, scheduling and reminders, CRM/spreadsheet integrations, reports, and internal AI assistants. (David may edit this list.)
- Zyprus works with **any kind of business**, not one sector.
- **Disclose that you are an AI assistant** in your first message.
- Only discuss Zyprus, the person's business and their processes. Politely decline anything else: general questions, homework, coding help, jokes, other companies. This is required by WhatsApp's 2026 policy on general-purpose AI chatbots.
- **Never** quote prices, promise results, give timelines, book meetings or confirm availability. If asked about price: "Depende del proceso; David te lo explica con base en tu caso."

### 8.2 Conversation goal (in order, one question at a time, max ~60 words per message)

1. Greet, say you are Zyprus's AI assistant, and ask what they want to improve or automate.
2. Understand the **process or leak** (`process_to_fix`).
3. Ask the **name** and the **business** (type, and name if offered).
4. If natural, ask 1–2 of: how they handle it today, volume (chats/leads/orders per week), urgency, location, team size. Budget only if the conversation invites it, gently and optional.
5. In 2–3 sentences, explain how Zyprus could help with **their** case, without promises.
6. Ask **consent**: "¿Te parece bien si David, el fundador de Zyprus, te escribe personalmente para revisar tu caso?" and the **preferred contact time**.
7. Close: "Listo, {name}. David te escribirá personalmente desde su WhatsApp {CONTACT_PROMISE_ES}. Si quieres agregar algo, escríbeme aquí."

Do not interrogate. If the person gives several data points at once, skip those questions. If they are vague, help them with one or two examples relevant to their business.

### 8.3 Personal contact rules

- The contact is always **personal, by David**, through WhatsApp from **his own number** (a different number). Say this explicitly so the lead is not surprised.
- If the person asks for a human at any point → acknowledge, ask only for name + what it is about if missing, set `human_requested=true`, and confirm David will write to them.
- After handoff (`status=handed_off`): stay helpful and brief. Answer simple questions about Zyprus, do not restart qualification, and remind them David will contact them. Set `lead_update_worth_posting=true` only for new information that matters to David (new need, urgency, better contact time).

### 8.4 Opt-out

Messages like "no me interesa", "no me escribas más", "stop" → `opt_out=true`, reply with a short polite closing, no further questions.

### 8.5 Privacy

Before collecting contact details, or together with the consent question, say in one line that the information is only used for David to contact them about their request. Do not ask for ID numbers, card numbers, passwords or sensitive data. If the user shares them, do not repeat them back.

---

## 9. Discord messages

Use the Discord node with the webhook credentials. Plain text (or a simple embed) is fine; keep it scannable on a phone.

### 9.1 Leads channel (`<id>`)

**New lead**

```text
🟢 NUEVO LEAD | Zyprus WhatsApp
Nombre: {name} ({role})
Negocio: {business_name} · {business_type} · {business_size} · {location}
Quiere: {process_to_fix}
Hoy lo maneja: {current_handling}
Volumen: {volume}
Urgencia: {urgency} | Presupuesto: {budget}
Contactar: {preferred_contact_time} | Consentimiento: {sí/no}
Resumen: {summary}
Siguiente paso sugerido: {suggested_next_step}
Escribirle: https://wa.me/{wa_id}
```

- **URGENT / human requested:** prefix `🔴 PIDE HUMANO` and include the fields known so far.
- **Lead update:** `🔄 ACTUALIZACIÓN | {name}`: what changed, plus the wa.me link.
- **Partial lead** (from WF3, section 10): `🟡 LEAD PARCIAL (sin respuesta)`: known fields, consent status, wa.me link.
- Omit empty fields instead of printing "null".

### 9.2 Alerts channel (`<id>`)

```text
⚠️ ALERTA | {workflow_name}
Nodo: {node_name}
Error: {short_message} (código {code})
Contacto: {wa_id_masked}   ← mask as 57300•••4567
Hora: {bogota_time}
Ejecución: {N8N_BASE_URL}/workflow/{workflow_id}/executions/{execution_id}
Acción sugerida: {hint}
```

Hints by case:

| Case | Hint |
| --- | --- |
| Graph error `190` (token expired/invalid) | "Regenerar access token en Meta (API Setup) y actualizar la credencial 'Zyprus - WhatsApp Send (TEST)'. Pendiente: token permanente (System User)." |
| `131030` (recipient not in allowed list) | "Número no verificado como destinatario del número de prueba. Normal en pruebas del dashboard." Severity: warning |
| `131047` (outside 24 h window) | "Se intentó escribir fuera de la ventana de 24 h. Revisar lógica de follow-ups." |
| LLM/parse failure | "Respuesta de IA inválida; se envió mensaje de respaldo." |
| Discord lead post failed | Include the **full lead payload** in the alert |

Avoid alert storms: identical error + same node → max one alert per 10 minutes (store last-alert keys in a data table or Redis).

---

## 10. WF3 — Follow-ups (Schedule, every 15 minutes)

Free-form messages are only allowed inside the 24 h window, and that is the only place this workflow sends. **No template messages and no messages after the window closes** (budget decision).

### 10.1 Eligibility (all must be true)

- `status = active` (not `qualified`, `handed_off`, `opted_out`, `closed`)
- The last message in the conversation is **ours** (`last_outbound_at > last_inbound_at`), meaning we are waiting for the lead
- `now < window_expires_at − WINDOW_SAFETY_MIN`
- Not inside quiet hours (Bogota)

### 10.2 Schedule

| Follow-up | When | Condition |
| --- | --- | --- |
| FU1 | `last_inbound_at + FU1_DELAY_MIN` (2 h) | `followups_sent_in_window = 0` |
| FU2 (final) | `last_inbound_at + FU2_DELAY_MIN` (20 h) | `followups_sent_in_window = 1` |

- If a follow-up becomes due inside quiet hours, send it at `QUIET_END` (08:00) **only if** that time is still before `window_expires_at − WINDOW_SAFETY_MIN`. Otherwise skip it.
- Any inbound message resets `followups_sent_in_window` to 0 (handled in WF2).
- Max 2 follow-ups per window. Nothing after FU2.

### 10.3 Texts (fixed, no LLM, personalized)

Use `{name}` if known, else omit the name. Use `language`.

- **FU1 (es):** "Hola {name}, ¿pudiste ver mi último mensaje? Cuando quieras seguimos; con un par de datos más David puede escribirte con una idea concreta para tu negocio."
- **FU2 (es):** "Hola {name}, te escribo por última vez para no dejar tu consulta en el aire. Si te interesa que David te contacte, respóndeme cualquier mensaje y seguimos. ¡Gracias!"
- **FU1 (en):** "Hi {name}, did you see my last message? Whenever you're ready we can continue; with a couple more details David can reach out with a concrete idea for your business."
- **FU2 (en):** "Hi {name}, one last message so your request doesn't get lost. If you'd like David to contact you, just reply and we'll continue. Thanks!"

After sending: log to `wa_messages` (`kind=followup_1|followup_2`), increment `followups_sent_in_window`, set `last_followup_at` and `last_outbound_at`. Respect `DRY_RUN`.

### 10.4 Partial lead at window end

When `now ≥ window_expires_at − 60 min`, `status=active`, `partial_lead_notified=false`, and at least `process_to_fix` **or** `business_type` is known → post **🟡 LEAD PARCIAL** to the leads channel and set `partial_lead_notified=true`. Include `contact_consent` so David decides whether to reach out personally. Do not send anything to the user.

### 10.5 Cost note

Follow-ups and replies are free-form messages inside the window ("service messages"). As of October 2026, providers report the first 1,000 service messages per month per number are free, then a small per-message fee (about USD 0.0008 in Colombia). Expected cost: about zero.

---

## 11. Tests the agent must run itself

Use pinned data / `test_workflow` on WF2 (or WF5 harness) with the payload shape from section 2. Use a fake `wa_id` such as `570000000001` with `DRY_RUN=true`, except where noted. Clean the test rows from the data tables after each scenario.

| # | Scenario | Input | Expected |
| --- | --- | --- | --- |
| A1 | Validation | — | All workflows pass `validate_workflow`; every node passes `validate_node_config` |
| A2 | Status event ignored | Payload with `statuses`, no `messages` | WF1 ends, no WF2 call |
| A3 | Duplicate message | Same `message_id` twice | Second run ends at dedupe; one row in `wa_messages` |
| A4 | New contact greeting | "Hola" | Conversation row created; reply discloses AI and asks what they want to improve; outbound logged |
| A5 | Happy path | Scripted 5–7 turns: need → name + business → volume → consent yes + time | `wa_leads` filled; `lead_complete` true; **one** `[TEST]` lead post in leads channel; `status=handed_off`; closing reply mentions David contacts personally from his WhatsApp |
| A6 | Info all at once | One message with name, business, need and consent | Lead completes in 1–2 turns, no redundant questions |
| A7 | Human request | "Quiero hablar con una persona" | `🔴 PIDE HUMANO` post; `status=handed_off` |
| A8 | Opt-out | "No me interesa, no me escribas" | `status=opted_out`; polite close; no alert; WF3 never selects it |
| A9 | Off-topic | "Escríbeme un poema" / "¿Quién ganó el partido?" | Polite decline, back to the topic; `off_topic=true` |
| A10 | Price question | "¿Cuánto cuesta?" | No price; says David explains it based on the case |
| A11 | Non-text | `type=audio` | Fixed "solo texto" reply; no LLM call |
| A12 | Burst | 3 messages from the same contact within 3 s | One combined reply (if debounce is implemented) |
| A13 | After handoff | New message after A5 | Brief reply, no re-qualification, no duplicate lead alert; update post only if meaningful and the cooldown passed |
| A14 | FU1 | Set `last_inbound_at = now − 2h05m`, `last_outbound_at` after it, `status=active`, run WF3 | FU1 sent (dry-run post), counter = 1 |
| A15 | FU2 | Set `last_inbound_at = now − 20h05m`, counter = 1 | FU2 sent, counter = 2 |
| A16 | Quiet hours | Due time inside 21:00–08:00 | Deferred to 08:00 or skipped per 10.2 |
| A17 | Window end | `window_expires_at` in 45 min, partial data | One `🟡 LEAD PARCIAL` post; no message to the user |
| A18 | No FU when not waiting | Last message is the user's | WF3 sends nothing |
| A19 | Error path | Force a failing node (e.g. invalid phone number id) | WF4 posts `[TEST]` alert with execution link |
| A20 | LLM JSON failure | Mock invalid LLM output | One retry, then fallback text + alert |
| A21 | Real send (only if David agrees) | `DRY_RUN=false`, recipient = `TEST_RECIPIENT` | Message arrives on David's phone. If error 190 → ask David to refresh the token |

Report each scenario as pass/fail with the execution ID.

---

## 12. Tests David must run (step by step)

The agent cannot do these. Provide this list to David, adjusted to the final build.

### 12.0 Before testing

1. **Publish the Meta app.** App settings → Basic: add a Privacy Policy URL, app icon and category, then save. Click **Publish** in the left menu. Until this is done, real messages do not reach n8n.
2. **Refresh the access token** if it is older than about 24 h: WhatsApp → API Setup → Generate token. Paste it into the n8n credential `Zyprus - WhatsApp Send (TEST)`.
3. **Check the webhook:** Use cases → Connect on WhatsApp → Configuration → Webhooks. The Callback URL must end in `/webhook/<webhook-id>/webhook`, and the **messages** field must be **Subscribed**.
4. Confirm with the agent that `DRY_RUN=false` and that WF1, WF2, WF3 are active.

### 12.1 Dashboard test

1. In Webhooks, click **Test** on the `messages` field and send it.
2. Expect: an execution in WF1. The reply to the fake number fails with `131030`, and a **warning** appears in the alerts channel. Nothing crashes.

### 12.2 Real conversations from your phone (send to the test number)

Between scenarios, ask the agent to delete your rows in `wa_conversations`, `wa_leads` and `wa_messages` so you start fresh.

| # | What you send | What you should see |
| --- | --- | --- |
| D1 | "Hola" | Reply within ~10 s: AI disclosure + a question about what you want to improve |
| D2 | Play a real lead: e.g. "Tengo una tienda de perfumes y respondo WhatsApp todo el día a mano", then your name, volume, "sí, que me escriba David en la tarde" | Leads channel gets 🟢 NUEVO LEAD with correct fields and a working wa.me link. The bot says David will write to you personally from his WhatsApp |
| D3 | Send 2 more messages after D2 | Short answers, no new questions, no duplicate lead post |
| D4 | Fresh start: "Quiero hablar con una persona" | 🔴 PIDE HUMANO in the leads channel |
| D5 | Fresh start: "Escríbeme un poema" | Polite decline, redirects to your business |
| D6 | Fresh start: send a voice note | "Solo texto" reply |
| D7 | Fresh start: 3 quick messages in a row | One combined reply (if debounce was built) |
| D8 | Fresh start: "No me interesa" | Polite close; no follow-ups later |

### 12.3 Follow-up test (shortened timing)

1. Ask the agent to temporarily set `FU1_DELAY_MIN=5` and `FU2_DELAY_MIN=15`, and to disable quiet hours.
2. Fresh start: send "Hola", answer the first question, then **stop replying**.
3. Expect FU1 about 5–20 min later (the scheduler runs every 15 min), then FU2 about 15–30 min after your last message.
4. Reply to FU1 instead, in a second run: the counter resets and no FU2 arrives while you keep talking.
5. Ask the agent to restore `120` / `1200` and quiet hours.

### 12.4 Error alert test

1. In n8n, temporarily change the access token in `Zyprus - WhatsApp Send (TEST)` to an invalid value.
2. Send "Hola" from your phone.
3. Expect: ⚠️ ALERTA in the alerts channel with error 190 and the token hint.
4. Restore the real token and send "Hola" again to confirm it works.

### 12.5 Check on your phone

- Both Discord channels notify you, and the lead card is readable without scrolling much.
- The wa.me link opens a chat with the lead from your own WhatsApp.

---

## 13. Acceptance criteria

- [ ] Only one active WhatsApp Trigger for this Meta app; callback URL unchanged
- [ ] No reply ever sent outside the 24 h window; no template messages
- [ ] Max 2 follow-ups per window, none in quiet hours, none to `handed_off` / `opted_out` contacts
- [ ] Exactly one lead alert per lead; updates rate-limited
- [ ] Every failure produces an alert with an execution link; no lead is lost if Discord fails
- [ ] History, state and leads persist across n8n restarts (data tables)
- [ ] Bot never quotes prices, never answers off-topic requests, always discloses it is an AI, always says David contacts personally
- [ ] Agent tests A1–A20 pass (A21 if David allowed it); David's checklist (section 12) handed over

## 14. Out of scope (later)

- Template messages and follow-ups after 24 h (paid; revisit when there is budget)
- Voice-note transcription (a Groq credential exists; reuse the barbershop transcription pattern later)
- Take-over flow where David replies from Discord through the API
- Moving to the real Zyprus number and a permanent System User token (`Zyprus - WhatsApp Send (LIVE)`)
- Mark-as-read and typing indicator (optional; verify the current Graph API format before adding)
