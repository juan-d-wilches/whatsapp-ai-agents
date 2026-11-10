# Zyprus WhatsApp Attention Channel — Runbook

> Internal runbook, published with identifiers (server, Meta, n8n, phone numbers) redacted.

Inbound-only WhatsApp assistant for Zyprus. It explains what Zyprus does, leads the conversation to qualify the lead, gets consent, says someone from the team ("alguien del equipo", never a name) will write personally from another number, and posts the lead to Discord #leads. Errors go to Discord #alerts.

- **Requirements:** [spec.md](spec.md)
- **Assistant brain:** [system-prompt.md](system-prompt.md) + [knowledge-base.md](knowledge-base.md)
- **Extras beyond the spec + Discord card guide:** [features.md](features.md)
- **Instance:** `https://<your-n8n-host>`, folder **Zyprus Attention Channel** (`<id>`), tag `zyprus-attention-channel`.

## 1. Workflows

| Workflow | ID | Trigger | State | Role |
|---|---|---|---|---|
| Zyprus WhatsApp - Inbound | `<id>` | WhatsApp Trigger (only one per Meta app) | published (router added 2026-10-18) | Router: has messages → split → skip reactions → normalize → Process Message (async) |
| Zyprus WA - Process Message | `<id>` | Execute Workflow | published | Dedupe, read + typing, voice transcription, debounce, daily cap, AI, lead logic, Discord, reply |
| Zyprus WA - Send Reply | `<id>` | Execute Workflow | published | **The only WhatsApp send path**: 24 h window guard, DRY_RUN, recipient guard, Graph error classification, logging |
| Zyprus WA - Send Alert | `<id>` | Execute Workflow | published | Alert formatter (masked number, Bogotá time, execution link, hint by code) + 10 min throttle |
| Zyprus WA - Transcribe Voice Note | `<id>` | Execute Workflow | published | Meta media → size guard → Groq Whisper → `{ok, text}` |
| Zyprus WA - Error Handler | `<id>` | Error Trigger | published | Catch-all for unhandled failures → Send Alert (direct Discord fallback) |
| Zyprus WA - Follow-ups | `<id>` | Every 15 min | published | FU1 (30 min) / FU2 (1 h 30 min) inside the window, quiet hours, 🟢 card for consented leads who went quiet, held-back 🔄 updates |
| Zyprus WA - Ops Monitor | `<id>` | Hourly + daily 08:00 | published | WhatsApp token health check (alerts at most every 3 h) + 📊 daily summary (own Discord channel) / heartbeat |
| Zyprus WA - Data Retention Cleanup | `<id>` | 1st of each month, 03:00 | published | Deletes messages, lead and conversation rows of contacts whose last message is older than 12 months (privacy promise). Discord cards are not touched. |
| Zyprus WA - Test Harness | `<id>` | Webhook (secret path) | **unpublished** | Test entry point (publish only while testing) |

All workflows use timezone `America/Bogota`, save execution data, and have **Zyprus WA - Error Handler** as Error Workflow. Exceptions: Send Alert and the Error Handler itself have none, to avoid loops.

## 2. Data tables

| Table | ID | Content |
|---|---|---|
| `zyprus_wa_config` | `<id>` | **One row of settings**. Edit it in n8n (Overview → Data tables). Changes apply on the next message, no republish needed. |
| `zyprus_wa_conversations` | `<id>` | One row per contact: status, stage, 24 h window, follow-up counters, alert flags, daily AI counter |
| `zyprus_wa_leads` | `<id>` | One row per contact: lead fields, consent, score (hot/warm/cold), summary, suggested next step, `referral_source` (where they found Zyprus, added 2026-10-21) |
| `zyprus_wa_messages` | `<id>` | Every inbound/outbound message (`status` received/sent/dry_run/failed). AI history, dedupe and debounce source. |
| `zyprus_wa_alert_throttle` | `<id>` | Last time each alert key fired + suppressed repeats |

### Config knobs (`zyprus_wa_config`)

| Column | Now | Meaning |
|---|---|---|
| `dry_run` | false | true = nothing goes to WhatsApp; each reply is previewed in #alerts as `[TEST][DRY_RUN]` |
| `restrict_to_test_recipient` / `test_recipients` | **false** (live since 2026-10-20) / `<test recipient>` | When true, live sends only go to the numbers listed (comma-separated, digits only). Set it back to true to test without reaching real leads. |
| `discord_prefix` | "" (empty, live) | Prefix on every Discord post; any non-empty value also adds "PRUEBA" to card footers. Was `[TEST] ` during testing. |
| `phone_number_id` | `<phone_number_id>` | Official Zyprus number <official number> (WABA `<waba_id>`). Used by the token check and as a send fallback; inbound messages carry their own. The old Meta test number was `<test_phone_number_id>`. |
| `llm_model_primary` / `llm_model_fallback` | `deepseek/deepseek-v4.1-flash` / `minimax/minimax-m3` | OpenRouter model IDs |
| `fu1_delay_min` / `fu2_delay_min` | 30 / 90 | Follow-up timing after the lead's last message (changed from 120 / 1200 on 2026-10-20: FU2 goes out one hour after FU1). Checks run every 15 min, so a message can arrive up to 15 min late. |
| `max_followups_total` | 4 | Lifetime cap of follow-ups per conversation (`followups_total`, never reset). Replies reset the per-window counter, but after this many nudges the contact gets no more. |
| `lead_post_wait_min` | 10 | A lead who said yes but never answered the contact-time question gets its 🟢 card posted by Follow-ups this many minutes after our last message |
| `quiet_enabled`, `quiet_start`, `quiet_end` | true, 21:00, 08:00 | No follow-ups in this range (Bogotá). Replies are always immediate. |
| `window_hours` / `window_safety_min` | 24 / 30 | Meta customer-service window and the last minutes in which nothing is sent |
| `debounce_seconds` | 8 | Groups bursts of short messages into one reply |
| `history_limit` | 20 | Past messages sent to the AI |
| `lead_update_cooldown_min` | 30 | Min gap between 🔄 lead updates |
| `max_ai_turns_per_day` | 40 | Per-contact AI replies per day; then one polite notice + alert, then silence until tomorrow |
| `max_audio_bytes` | 1000000 | Voice notes above this size (≈5 min) are not transcribed |
| `contact_promise_es` / `_en` | "lo antes posible, normalmente el mismo día hábil" | Wording of when the team will write |
| `alert_throttle_min` | 10 | Same alert (workflow + node + code) at most once per this many minutes |
| `graph_api_version` | v26.0 | Graph API version for typing indicator and token check (raised from v25.0 on 2026-10-20; Meta was already auto-upgrading v25.0 calls to v26.0 with a deprecation warning) |

## 3. How a message flows

1. **Meta → Inbound router.** Status events and reactions are dropped. Each message starts one async *Process Message* run, so Meta gets its 200 at once.
2. **Dedupe** by `message_id`. Then **read receipt + "escribiendo…"** (live mode only; lasts up to 25 s).
3. **Voice notes:** transcribed with Groq Whisper *before* logging, so the history keeps the transcript.
4. **Debounce:** log the message, wait 8 s, and continue only if it is still the contact's newest inbound message. The surviving run answers every message since the last reply in one go.
5. **Context:** history (failed sends excluded), stored lead, user-turn count, `NEXT_FOCUS` (the next funnel step) and `CONTACT_ALLOWED`, so the bot always leads and never offers contact early. Qualification gate: need + name + business + 2 context facts over ≥5 user turns (volume needs a number), or 4+ facts at once, or 8 turns. The name is asked directly (the WhatsApp profile name is never confirmed back to the person). After consent the bot asks **"¿por dónde nos conociste?"** once (`referral_source`, see §6d), then the contact time once, then closes.
6. **Routing:** daily cap → notice; media without text → fixed reply; everything else → AI (DeepSeek with MiniMax fallback, strict JSON schema, `maxTokens` 4000 because DeepSeek reasons before answering, no context-free auto-fixer; one retry with the full conversation, then the fallback text and an alert).
7. **Decide Actions:** the code, not the AI, decides.
   - Lead fields are merged, never erased, and placeholders ("Not provided"…) are ignored.
   - `lead_complete` = name + business + process + consent. The 🟢 card is posted **once**, on the closing turn (after the source and contact-time questions were answered or skipped), so both are already in it. A lead who consents and then goes quiet gets the card from Follow-ups after `lead_post_wait_min`.
   - `handed_off` starts only on the turn whose reply is the closing message.
   - Reply polish: Spanish guard, colons → commas, emojis only in the opening/closing, and after handoff one bubble with any "te va a escribir" reminder removed.
8. **Lead card first** (🟢 new / 🔄 update / ⛔ no contactar), **then the reply.** If #leads fails, the full card goes to #alerts as `🚨 LEAD SIN PUBLICAR`. Updates are rate-limited to 1 per 30 min per lead (counted from the last card of any kind), except the closing turn and opt-outs, which post at once. A held-back update sets `lead_update_pending`, and Follow-ups posts the full snapshot when the 30 min pass.
9. **Send Reply:** 1–2 bubbles, the second 2 s later. It blocks anything outside the 24 h window, honors DRY_RUN and the recipient allow-list, classifies Graph errors and logs everything.

**Statuses:** `active` → `handed_off` (closing message sent; asking for a person does not skip qualification) · `opted_out` (re-opens if they clearly re-engage) · `closed` (set manually; behaves like handed_off).

## 4. Discord messages

- All posts are color-coded Discord **embeds** (card design and color legend in [features.md](features.md#discord-guide)).
- **#leads:** 🟢 NUEVO LEAD, 🔄 ACTUALIZACIÓN (always the full current lead), ⛔ NO CONTACTAR, 📊 RESUMEN DIARIO. Only people who gave consent ever get a card (the 🟡 LEAD PARCIAL card was removed on 2026-10-20). Every card ends with a `wa.me` link and shows a priority (🔥 Caliente / 🟠 Tibio / 🔵 Frío) with the reason, plus **📲 Nos conoció por** when the lead said where they found Zyprus. The 📊 daily summary has a **📲 Cómo nos conocieron** tile that counts the day's new leads by platform.
- **#alerts:** ⚠️ ALERTA, 🟠 AVISO and 🚨 LEAD SIN PUBLICAR, plus the `[TEST][DRY_RUN]` previews. Each alert shows the node, the error and code, the masked contact (`57300•••4567`), Bogotá time, the execution link and a suggested action.
- **Hints by code:** 190 token, 131030 test recipient (warning), 131047 window, 402 OpenRouter credits, LLM, AUDIO, DAILY_CAP, WINDOW_GUARD, RECIPIENT_GUARD, AUTH, TOKEN_CHECK, DISCORD_LEAD.

## 5. Costs

- **AI:** ≈5k prompt + ≈400–700 output tokens per AI turn ≈ **US$0.001 per turn** on DeepSeek V4.1 Flash. The static prompt is cache-friendly. US$4.55 ≈ 4,000 turns, or several hundred full conversations. The daily cap protects against spam.
- **Groq Whisper:** negligible.
- **WhatsApp:** service messages inside the window, so about zero cost (spec §10.5).

## 6. Agent test results (2026-10-18, DRY_RUN, fake numbers `570000000xxx`)

| # | Scenario | Result | Evidence |
|---|---|---|---|
| A1 | Validation | ✅ All 9 workflows valid, 0 errors (remaining warnings are intentional `executeOnce`) | `n8n_validate_workflow` |
| A2 | Status event | ✅ Ignored, Process Message not called | harness `meta` |
| A3 | Duplicate `message_id` | ✅ Stops at Dedupe, one row in the log | exec 677 |
| A4 | "Hola" from a new contact | ✅ AI disclosure + what to improve, 2 bubbles, conversation row created | exec 673 |
| A5 | Scripted happy path (4 turns) | ✅ Lead complete → one 🟢 card (🔥), `handed_off`, close says David writes personally from his WhatsApp | exec 687 |
| A6 | All info in one message | ✅ (after a prompt fix) disclosure + pitch + close in one turn, 🟢 card | exec 717 |
| A7 | "Quiero hablar con una persona" | ✅ (after a prompt fix) disclosure, 🔴 PIDE HUMANO, `handed_off` | exec 721 |
| A8 | Opt-out | ✅ `opted_out`, polite close, no alert, never followed up | exec 706 |
| A9 | Off-topic (poem + football) | ✅ Light decline, back to the business | exec 709 |
| A10 | Price question | ✅ No price, "David te lo explica con base en tu caso", steers with examples | exec 713 |
| A11 | Image / failed voice note | ✅ Fixed replies, no LLM; AUDIO warning | exec 729 / 732 |
| A12 | Burst of 3 messages | ✅ 2 runs superseded, 1 combined reply | exec 737–739 |
| A13 | After handoff | ✅ No re-qualification; 🔄 update; the 30 min cooldown suppressed a second update | exec 691 / 695 |
| A14 | FU1 | ✅ "Hola Sara, ¿pudiste ver…", counter 0→1 | Follow-ups exec 743 |
| A15 | FU2 | ✅ Final follow-up, counter 1→2 | exec 743 |
| A16 | Quiet hours | ✅ Due follow-ups held while in quiet hours | exec 742 |
| A17 | Window ending + partial data | ✅ One 🟡 LEAD PARCIAL, nothing sent to the user, not repeated (card removed on 2026-10-20) | exec 742 / 743 |
| A18 | Last message is the user's | ✅ Not selected; re-run is idempotent | exec 743 / 746 |
| A19 | Errors | ✅ Recipient guard blocks unlisted numbers; a real Graph failure is classified **190** and logged `failed`; the crash test reaches the Error Handler | harness 750 / 763, Error Handler 757 / 795 |
| A20 | LLM failure | ✅ Retry → fallback text + LLM alert | exec 760 |
| A21 | Real send | ⏳ Needs your number + a fresh token | — |
| A22 | Lead score | ✅ On every card | exec 687 |
| A23 | Voice note → Groq | ✅ Spanish clip transcribed exactly, AI replied | exec 790 |
| A24 | Daily cap (set to 2) | ✅ 3rd message → notice + DAILY_CAP alert, 4th → silent | harness 769 |
| A25 | Token health check | ✅ Detected the **real expired token** (190) and alerted | Ops Monitor exec 747 |
| A26 | Daily summary | ✅ Correct counts, 🔥 leads to contact first, token ❌ | exec 749 |
| A27 | Tone and steering | ✅ "¿qué es n8n?" → one line + back to the funnel. Fixed during tests: disclosure on every first reply, never say *when* David writes, gender-neutral wording, voice-note re-ask. | exec 725 |

All test rows were deleted afterwards; the config is back to its defaults.

> A7 and the 🔴 card were removed on 19-Oct, and replies no longer name David. See 6b.

### 6b. Fix round after David's phone test (2026-10-19)

**What David reported, and the root cause of each:**

| Reported | Root cause | Fix |
|---|---|---|
| A reply in English; lead details lost after the update | DeepSeek used all of `maxTokens` 1000 on hidden reasoning and returned **empty**. The parser's auto-fixer, which can't see the conversation, invented an English reply and filled the lead with "Not provided", which overwrote the real data. | `maxTokens` 4000, fixer removed, placeholder and language guards |
| Offered contact too early; named "David, el fundador" | Turns were counted per bubble, so context questions were skipped. The prompt named David. | User-turn gate (see §3.5), `CONTACT_ALLOWED`, "alguien del equipo" everywhere (prompt, KB, fixed texts, follow-ups) |
| 2 bubbles after handoff, the second a "David te escribe" reminder; kept asking questions | The post-handoff rule said to remind them. | One bubble, no question, and reminder sentences removed in code |
| Colons and emojis | Not restricted | Prompt rule plus enforcement in code |
| Option to ask for a person | — | Removed: the bot keeps qualifying. Right after the consent question it counts as a yes |

**Found during the replay and fixed:**
- One rich sentence opened the gate ("muchísimos" counted as volume).
- Handoff happened on the consent turn, so the closing never went out.
- An update card was posted 6 s after the new card.
- The consent question was repeated word for word.
- An opt-out after the 🟢 card didn't tell #leads.
- A lead who went quiet between consent and the closing would still get follow-ups. Follow-ups now skip any conversation with `lead_notified` = true.

**Replay results** (harness, DRY_RUN, `570000000041` = David's script, `570000000042` = everything in one message):

| Check | Result |
|---|---|
| Language | ✅ Every reply in Spanish |
| Naming | ✅ No "David"/"fundador" (except confirming the lead's own name) |
| Punctuation and emojis | ✅ No colons; emojis only in the greeting and the closings |
| Consent timing | ✅ 041 got it after 5 user turns with a numeric volume; 042 got it in turn 1 (4+ facts) |
| Consent answer | ✅ "Quiero hablar con una persona" right after the consent question = yes, then contact time asked once |
| Closing | ✅ Sent once; `handed_off` only then |
| After handoff | ✅ "Gracias", "otro negocio de flores", "poema", "necesito una persona": 1 bubble each, no question, no reminder, no 🔴 card |
| Lead row | ✅ Kept all real values and merged both businesses |
| Opt-out | ✅ "Cancela todo" → polite close + ⛔ NO CONTACTAR card (WF2 exec 1109) |
| Pending flush | ✅ Follow-ups posted the held-back 🔄 snapshot once and cleared `lead_update_pending` (exec 1111) |

All test rows (fake numbers and David's <test recipient>) were deleted and the harness was unpublished.

### 6c. Improvements after David's follow-up test (2026-10-20)

| Request | Before | Now |
|---|---|---|
| Name | The bot confirmed the profile name ("¿hablo con David?"), which surprised people | Asks directly ("¿con quién tengo el gusto?"); the profile name is internal only (prompt + context line) |
| Lead notification | 🟢 card on the consent turn, then a 🔄 card right after the contact-time answer | One 🟢 card on the closing turn, with the contact time. `Build Turn Context` now detects "contact time already asked" from our last texts (`<id>`) instead of from `lead_notified`; `Decide Actions` posts `new` only if `closingNow` or the time is known |
| Consented lead who goes silent | n/a | Follow-ups action `consented_lead`: `lead_post_wait_min` (10) min after our last message it posts the full 🟢 card (noting no contact time was given), sets `lead_notified`, and the bot still sends the closing if they come back. Consented leads get no nudges |
| Follow-ups after a reply | A reply already restarts the cycle: inbound resets `followups_sent_in_window` and moves `window_expires_at` to last message + 24 h (Meta counts the window from the lead's LAST message, not the first). No lifetime limit | Same behavior plus a lifetime cap `max_followups_total` (4) using the new column `followups_total` (not reset by inbound) |

**Replay results** (harness, DRY_RUN, fake numbers):

| Check | Result |
|---|---|
| Name question | ✅ "¿Con quién tengo el gusto?", no profile-name guess |
| Consent turn | ✅ No card posted, bot asks the contact time |
| Contact-time turn | ✅ Closing sent, `handed_off`, exactly one 🟢 card, no 🔄 |
| After handoff | ✅ "Gracias" → one short bubble, no card |
| Silent consented lead | ✅ Follow-ups run 1537 posted the 🟢 card once and set `lead_notified` |
| Cap | ✅ `followups_total` 4 → nothing sent; 3 → one sent and counter went to 4 |

All fake rows were deleted and the harness was unpublished.

### 6d. "¿Por dónde nos conociste?" + platform counter (2026-10-21)

David asked for the lead's source (mostly TikTok or Instagram) before the lead card, on the card, and counted per platform in the daily summary.

| Where | Change |
|---|---|
| `zyprus_wa_leads` | New column `referral_source` (string). |
| Process Message → `Build Turn Context` | New funnel step `referral_source` between `consent` and `contact_time`. It is asked **once per conversation**: "asked" is detected in *all* our earlier replies (`nos conociste / encontraste / viste…`), not just the last one, so it is never repeated. Skipped if already known or if consent is "no". `CONTACT_ALLOWED` includes it. New output `source_asked`. |
| Process Message → `AI: Zyprus Assistant` + `Parse: Assistant JSON` | Prompt step 6 asks it casually in one question ("De una. Y cuéntame, ¿por dónde nos conociste, TikTok, Instagram u otro lado?"); the AI saves only what the lead said in `lead.referral_source` (optional in the schema so a model that forgets it doesn't fail the turn). Mirror updated in [system-prompt.md](system-prompt.md). |
| Process Message → `Decide Actions` | Normalizes the answer (`tiktok`/`tik tok` → TikTok, `insta`/`IG` → Instagram, also Facebook, Google, YouTube, LinkedIn; anything else kept as said). The 🟢 card never goes out before the source question was asked (unless the AI closes early). Card field **📲 Nos conoció por** (embed + plain-text backup). |
| Process Message → `Save Lead` | Saves `referral_source`. |
| Follow-ups → `Plan Follow-up Actions` | The 🟢 consented-lead and 🔄 pending-update cards show **📲 Nos conoció por** too. A lead who goes quiet after the source question still gets the card after `lead_post_wait_min`. |
| Ops Monitor → `Build Daily Summary` | New tile **📲 Cómo nos conocieron** over the day's new leads, for example `🎵 TikTok 2 · 📸 Instagram 0 · 🔗 Facebook 1 · ❔ Sin dato 1`. TikTok and Instagram always show; other platforms appear when they occur. |

New funnel end: consent → source → contact time → closing + 🟢 card (one extra turn). Leads who had already consented when this went live get the source question once before their closing.

**Testing status:** all 3 workflows pass `n8n_validate_workflow` (0 errors, only the intentional `executeOnce` warnings) and the connections were checked. The edits were saved to the published workflows, and n8n publishes on save (verified on Ops Monitor: new active version 2026-10-21 20:18), so the change is **live**. The harness replay could **not** be run: publishing the Test Harness was blocked by the Claude Code auto-mode classifier. Do a phone test (step 7 below) to confirm on a real conversation. Rollback, if needed: n8n-mcp took a snapshot of each workflow right before this edit (`n8n_workflow_versions`, newest local snapshot dated 2026-10-21 for Process Message, Follow-ups and Ops Monitor). The extra column is harmless if left in place.

## 7. Your tests (step by step)

**0. Before testing**
1. ✅ **Access token refreshed** (verified 19-Oct 00:21 UTC: test number <test number>, quality GREEN). It is still a temporary ~24 h token.
2. In `zyprus_wa_config`, `test_recipients` (`<test recipient>`) and `phone_number_id` (`<test_phone_number_id>`) are already set. Set `dry_run` = **false** when you are ready for real messages.
3. ✅ **Meta app published** and callback URL confirmed (19-Oct).
4. **Inbound was republished on 18-Oct with the router.** Check Meta → Use cases → Connect on WhatsApp → Configuration → Webhooks. The callback URL must still end in `/webhook/<webhook-id>/webhook`, and the `messages` field must be **Subscribed**.

**1. Dashboard test.** In Webhooks, click **Test** on `messages`. Expected: an execution in Inbound; the reply to the fake number goes to #alerts as **🟠 AVISO / RECIPIENT_GUARD** (it's not in your allow-list). Nothing crashes.

**2. From your phone.** Between scenarios, ask the agent to delete your rows so you start fresh.

| # | You send | You should see |
|---|---|---|
| D1 | "Hola" | **Blue ticks + "escribiendo…"** right away, then a reply in ~10–15 s: AI disclosure + what you want to improve |
| D2 | Play a real lead: answer its questions one by one (need, name, business, volume with a number, team size or city), then say yes, answer "TikTok" when asked where you found Zyprus | No contact offer before ~5 turns. Consent asked once, "¿por dónde nos conociste?" once, contact time once. The card shows **📲 Nos conoció por: TikTok** and the next 📊 summary counts it under TikTok. 🟢 NUEVO LEAD with correct fields, priority and a working `wa.me` link. One closing: "alguien del equipo te va a escribir personalmente desde otro número" |
| D3 | 2–3 more messages after D2 (add another business, say thanks) | One bubble each, no questions, no reminder. A 🔄 update with the full lead, right away or up to 30 min later |
| D4 | Fresh: "Quiero hablar con una persona" | It says someone from the team reviews each case and keeps qualifying. No special card |
| D5 | Fresh: "Escríbeme un poema" | Light decline, back to your business |
| D6 | Fresh: **a voice note** about your business | It understands the audio and continues the conversation (tests the real Meta media download) |
| D7 | Fresh: 3 quick messages | One combined reply |
| D8 | Fresh: "No me interesa" | Polite close; no follow-ups later |
| D9 | A photo | "Por ahora leo mensajes de texto y escucho notas de voz…" |

**3. Follow-ups** (Follow-ups is published):
1. Set `fu1_delay_min` = 5, `fu2_delay_min` = 15, `quiet_enabled` = false.
2. Fresh start: "Hola", answer once, then stop replying.
3. Expect FU1 within about 5–20 min and FU2 within about 15–30 min.
4. In a second run, reply to FU1: the counter resets and no FU2 arrives while you keep talking.
5. Restore 30 / 90 / true (current production values).

**4. Error alert.**
1. Put an invalid token in the credential.
2. Send "Hola". Expect ⚠️ ALERTA with code **190** and the token hint.
3. Restore the token and send "Hola" again.

**5. Ops Monitor** (published): every morning at 08:00, 📊 RESUMEN DIARIO shows up in the summary channel. While the token is expired you'll also get a 190 alert at most every 3 h.

## 8. Go-live checklist

**Live since 2026-10-20 (~19:40 Bogotá)** on the official number <official number>.

- [x] **Inbound** updated with the router and republished (18-Oct).
- [x] Official number linked to the app. The app is subscribed to WABA `<waba_id>` (`POST /subscribed_apps` → success, 20-Oct). David's "Hola" to the new number was answered end to end (Process Message exec 1558, both bubbles `sent`).
- [x] New token in the existing credential **Zyprus - WhatsApp Send** (`<id>`, renamed from "(TEST)"; nodes reference it by ID, so no workflow edit was needed).
- [x] `zyprus_wa_config`: `dry_run` = false, `discord_prefix` = "", `restrict_to_test_recipient` = false, `phone_number_id` = `<phone_number_id>`, `graph_api_version` = v26.0.
- [x] **Follow-ups** and **Ops Monitor** published. Keep the **Test Harness** unpublished.
- [x] Token confirmed as never-expiring by David (20-Oct).
- [ ] **David:** in WhatsApp Manager, check the number's display name is approved and two-step verification (PIN) is set.
- [x] Knowledge base v2 deployed (20-Oct): remote from Colombia, cost factors, more FAQ/objections, "SI NO SABES" phrases, privacy and website links (rule 10 in the prompt). Left out on purpose: "puedes pedir una persona cuando quieras", offering the team in the price answer, and the "CÓMO CERRAR" sales section (they fight the bot-led funnel). Harness replay OK, test rows deleted.
- [x] 🟡 partial-lead card removed (Follow-ups + daily summary tile): only consented leads reach Discord.
- [x] OpenRouter low-credit alert dropped by David (20-Oct). A 402 from OpenRouter still raises the LLM failure alert in #alerts.
- [x] **Data retention (12 months, promised in the KB):** `Zyprus WA - Data Retention Cleanup` approved by David and published (20-Oct). Tested with a fake contact last seen 13 months ago (exec 1639): its 2 messages, lead and conversation were deleted and all current rows stayed. Deletion requests to <contact email> are still handled by hand (rows by `wa_id` + Discord cards).
- [ ] Optional: delete the unused Discord credential `Zyprus - Attention Channel OpenRouter Alerts` (`<id>`) and its webhook.
- [x] Archived `TEMP - Subscribe app to Zyprus WABA` deleted (20-Oct).

**WhatsApp limits (unverified business, no payment method):** the 250 cap counts only business-initiated (template) conversations per rolling 24 h. This channel never starts conversations: every reply and follow-up goes inside the 24 h window the lead opened, billed as `free_customer_service` (seen in the status webhooks), so no counter is needed.

**Testing after go-live:** to try something without reaching real leads, set `restrict_to_test_recipient` = true and `discord_prefix` = `[TEST] `, test, then revert both.

## 9. Maintenance

- **Change what the bot says or knows:** edit `system-prompt.md` / `knowledge-base.md` and ask the agent to redeploy the prompt. Never use `{ }` in the prompt; LangChain treats them as variables.
- **Swap the AI model:** change `llm_model_primary` / `llm_model_fallback` in the config table.
- **Reset a contact:** delete their rows (`wa_id`) in conversations, leads and messages.
- **Run agent tests again:** publish the Test Harness, POST to `/webhook/<secret-harness-path>` (modes `meta`, `message`, `burst`, `crash`; see the sticky note), then unpublish it.

## 10. Known limits

- **Voice notes:** a real Meta media download is only verified once you send one (D6). The Groq half is verified.
- **Typing indicator:** needs a real inbound `message_id`, so it's verified in D1.
- **Debounce merging:** for messages sent within a few milliseconds of each other, the combined turn follows arrival order, not WhatsApp timestamps.
- **Update cooldown:** a 🔄 update suppressed by the cooldown isn't re-posted later. The info is always saved in `zyprus_wa_leads`, and the next update card includes the latest summary.
