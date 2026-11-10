# Zyprus WhatsApp Channel: features added beyond the original spec

The spec asked for an inbound WhatsApp assistant that qualifies leads, posts them to Discord and sends two follow-ups inside the 24 h window. Everything below was added on top of that to make the channel feel premium, stay cheap and never fail silently.

## For the lead (what they experience)

| Feature | What it does | Why it matters |
|---|---|---|
| **Read receipts + "escribiendo…"** | As soon as a message arrives, the lead sees blue ticks and the typing indicator while the AI prepares the answer. | It feels like a person is on the other side, not a slow bot. |
| **Voice notes understood** | Audios are transcribed with Groq Whisper and answered like text. Images and files get a friendly "por ahora leo texto y escucho notas de voz". | Most Colombian leads send audios; the spec only replied "solo texto". |
| **Human, Colombian tone** | Short WhatsApp-style messages, "tú", natural words ("listo", "de una", "uff, eso quita mucho tiempo"). No generic AI phrases, gender-neutral wording, never the same sentence twice. **No colons** (they read like a bot) and **emojis only in the opening and closing messages**. Both rules are also enforced in code, not just in the prompt. | It is the first demo of Zyprus quality every lead sees. |
| **Replies in 1–2 bubbles** | Two bubbles only for the greeting and the first consent question; every other reply is one bubble. | Easier to read on a phone, and more natural. |
| **The bot leads the conversation** | The system works out the next missing piece of information (need → name → business → context → consent → where they found us → contact time) and the AI always ends with exactly one question. If the user goes off track ("¿qué es n8n?", "¿cuánto cuesta?"), it answers in one line and steers back. | Leads are qualified quickly and consistently; nobody can drag the bot into a random chat. |
| **Contact is earned, never offered early** | The bot never mentions being contacted until the lead is understood: need, name, business and 2 context facts across at least 5 user turns (a vague "muchísimos" doesn't count as volume). Someone who gives 4+ facts at once gets there in one turn; after 8 turns it moves on anyway. Asking "¿puedo hablar con una persona?" doesn't skip this: the bot says each case is reviewed personally by someone from the team and keeps qualifying. Asking it right after the consent question counts as a yes. | The team only gets leads worth calling, and the bot never feels pushy. |
| **"Alguien del equipo"** | The bot never names anyone and never says "el fundador". The person who follows up is always "alguien del equipo". | It works for any team size and keeps the owner out of the script. |
| **Clean handoff** | The closing message ("Alguien del equipo te va a escribir personalmente desde otro número…") is sent once. After that, every reply is one short bubble that acknowledges what they said, with no question and no reminder. New details (another business, a better time) go to the team. | No repetitive, fake-sounding reminders, and no wasted tokens. |
| **Curated knowledge base** | Services, examples by business type, how Zyprus works, FAQs and objection handling, kept in [knowledge-base.md](knowledge-base.md). If something isn't there, the bot says "eso lo revisa alguien del equipo contigo". | The bot never invents prices, clients or promises. |
| **Bursts merged into one reply** | If someone sends "hola" + "tengo un spa" + "quiero agendar citas" in a row, the bot waits 8 seconds and answers all three at once. | No triple answers, no confusion. |
| **Name asked directly** | The bot asks "¿con quién tengo el gusto?" instead of guessing from the WhatsApp profile name, and doesn't use any name until the person gives it. | Guessing "¿hablo con David?" surprised people; asking is more natural. |

## For David (what you get)

| Feature | What it does |
|---|---|
| **One lead card, with the contact time** | The 🟢 card is posted once, after the lead answers when to be contacted, so it already carries the best time. If they say yes and then go quiet, Follow-ups posts the card 10 min later (`lead_post_wait_min`), noting that no time was given. One notification per lead instead of 🟢 plus an immediate 🔄, and nobody who consented is left unreported. |
| **Follow-up cap** | A reply to a follow-up restarts the 2-follow-up cycle inside the new 24 h window (WhatsApp counts the window from the lead's last message), but each conversation gets at most 4 follow-ups in total (`max_followups_total`). Leads who already consented never get nudges. Persistent but not pushy. |
| **Where each lead found Zyprus** (2026-10-21) | Right after the lead accepts contact, the bot asks once, casually, "¿por dónde nos conociste, TikTok, Instagram u otro lado?". The answer is normalized (TikTok, Instagram, Facebook…), saved in `referral_source`, shown on the lead card as **📲 Nos conoció por**, and counted per platform in the daily summary. If they skip it, the bot moves on without insisting. |
| **Lead priority score** | Every lead is rated 🔥 Caliente / 🟠 Tibio / 🔵 Frío with a one-line reason, so you know who to call first. |
| **Redesigned Discord cards** | Color-coded cards with grouped fields and a clickable title that opens the lead's WhatsApp chat (§ Discord guide below). |
| **Daily summary at 08:00** | A 📊 card in the dedicated summary channel with the last 24 h: conversations, new contacts, leads by priority, **new leads by platform** (📲 Cómo nos conocieron: TikTok, Instagram, others, "Sin dato"), voice notes, opt-outs, alerts, token status, and the 🔥 leads to contact first. If it doesn't arrive, something is down (it doubles as a heartbeat). |
| **Consent first** | Only people who said yes to being contacted ever reach Discord. Someone who stops answering before consenting is never posted (the old 🟡 partial-lead card was removed so the channel matches the privacy policy). |
| **"Contact them first" links** | Every lead card and the daily summary include direct `wa.me` links. |
| **Simulation mode (DRY_RUN)** | While testing, replies are not sent to WhatsApp; they show up in #alerts as 🧪 Simulación cards with the exact text the client would see. |

## Protection and reliability

| Feature | What it does |
|---|---|
| **Daily AI cap per contact** | After 40 AI replies in one day, the contact gets one polite message, you get an alert, and the bot goes quiet with them until tomorrow. Protects your OpenRouter credit from spam or bots. |
| **Hourly WhatsApp token check** | Pings Meta every hour. If the token expires or is invalid, you get an alert (at most every 3 h) **before** a real lead hits the problem. |
| **Backup AI model** | If DeepSeek fails or returns an empty or invalid answer, the turn is retried with the full conversation, then MiniMax answers. If both fail, the lead gets a polite Spanish fallback ("Disculpa, tuve un problema técnico…") and you get an alert. A reply is never invented without the conversation. |
| **Lead data never overwritten** | Placeholders a model might write ("Not provided", "no especificado", "unknown") can never replace real data, and a Spanish-speaking lead never gets an English reply. |
| **Hard 24 h window guard** | No message can ever be sent outside the WhatsApp customer-service window, even by mistake. |
| **Test-recipient guard** | While testing, real messages can only go to the numbers listed in the config (`test_recipients`). |
| **Lead never lost** | If posting to #leads fails, the full lead is sent to #alerts instead. |
| **Smart error alerts** | Errors are translated into plain actions ("Regenerar el token…", "Recargar créditos de OpenRouter…"), with a link to the exact n8n execution. Repeated identical alerts are grouped (max one every 10 min). |
| **Duplicate protection** | Meta sometimes delivers the same message twice; the bot answers only once. |
| **Settings without touching workflows** | Timings, quiet hours, models, caps, test mode and contact wording live in one data table (`zyprus_wa_config`), so you can change them in seconds. |
| **Test harness** | A dedicated test workflow can simulate any conversation, burst, voice note or failure without a phone. |

## Discord guide

**#leads**

| Card | Color | Meaning |
|---|---|---|
| 🟢 Nuevo lead | Green | Lead complete and consented: write to them. Includes **📲 Nos conoció por** (TikTok, Instagram…) when they said it. |
| 🔄 Actualización | Blue | An existing lead added something important. The card always carries the **full current lead** plus what changed. At most one every 30 min per lead; anything held back is posted by Follow-ups once the 30 min pass, so no detail is lost. |
| ⛔ No contactar | Dark red | A lead you already received asked to cancel ("cancela todo", "no me interesa"). Don't write to them. |
| 📊 Resumen diario | Purple | 08:00 digest. Posted in its own **summary channel** (credential `Zyprus - Attention Channel Summary`), not in #leads. |

**#alerts**

| Card | Color | Meaning |
|---|---|---|
| ⚠️ Error | Red | Something failed; the card says what to do. |
| 🟠 Aviso | Orange | Expected or minor issue (blocked test send, daily cap, voice note not understood). |
| 🚨 Lead sin publicar | Dark red | #leads failed; the lead's data is in the card, contact them from here. |
| 🧪 Simulación | Gray | Test mode only: a reply that was **not** sent to WhatsApp. |

Cards marked `[TEST]` at the start and "PRUEBA" in the footer come from test mode (`discord_prefix` set) or from the Test Harness. Since go-live (2026-10-20) real cards carry no marker.
