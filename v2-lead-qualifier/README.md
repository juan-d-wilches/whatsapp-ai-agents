# v2 · WhatsApp lead-qualification channel

An inbound-only WhatsApp assistant for my AI automation agency, Zyprus. It explains the service, qualifies the lead step by step (need, name, business, context), asks for consent and posts a scored lead card to Discord. Errors go to a separate alerts channel.

**Status:** deployed on the official WhatsApp Business number from 20 Oct to Nov 2026. It **never received real users**. I paused the project and shut down the server, and the workflow JSON files were lost with it. What remains is the documentation I wrote and used to build and test it.

| File | What it is |
|---|---|
| [spec.md](spec.md) | Requirements written before building: behavior, data model, guardrails, acceptance criteria |
| [runbook.md](runbook.md) | The 10 workflows and 5 data tables, the full message flow, costs, the 27 test scenarios with results, three fix rounds with root causes, and the go-live checklist |
| [system-prompt.md](system-prompt.md) · [knowledge-base.md](knowledge-base.md) | The assistant's prompt and curated knowledge |
| [features.md](features.md) | Extras beyond the spec and the Discord card guide |
| [assets/daily-summary.png](assets/daily-summary.png) | A real daily summary from the Ops Monitor (30 Oct 2026) |

Identifiers (server, Meta, n8n, phone numbers) are redacted.
