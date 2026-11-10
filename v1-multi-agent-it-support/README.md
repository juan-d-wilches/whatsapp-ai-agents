# v1 · Multi-agent IT support on WhatsApp

A WhatsApp assistant for a small IT managed-services business. An LLM classifies each message and routes it to one of two agents:

- **Alex**, L1 IT support. Answers from a knowledge base (VPN, networking, accounts and identity, devices) with RAG on Qdrant. The agent picks the category, and the search is filtered by it. When it can't solve the issue, it escalates a ticket to L2.
- **Valeria**, sales. Answers from the service catalog and sends qualified leads (BANT) to the team.

**Status:** built for an acquaintance's business and never used.

## Workflows

| File | Role |
|---|---|
| `01-router-multi-agent-inbound.json` | Webhook from Evolution API. Ignores the bot's own messages, applies the Redis rate limit (8 messages/min), tracks new vs. returning users and the active agent per session, runs the LLM intent classifier and calls the right agent |
| `02-agent-it-support-alex.json` | Tool-calling agent: OpenRouter model, Redis chat memory, Qdrant vector store, escalation tool |
| `03-agent-commercial-valeria.json` | Same pattern with the service catalog and a lead tool |
| `04-tool-it-ticket-dispatcher.json` | Sends the L2 escalation by WhatsApp |
| `05-tool-commercial-lead-dispatcher.json` | Sends the qualified lead by WhatsApp |

## Import

1. Import the five files into n8n.
2. Create these credentials: OpenRouter, Redis, Qdrant, Hugging Face, and a Header Auth for Evolution API. Select them in each node.
3. Set the environment variable `ALERT_PHONE`, the number that receives tickets and leads.
4. In the router and in both agents, select the sub-workflows again in the Execute Workflow and tool nodes. Their IDs change on import.

**Known gap, fixed in v2:** none of the 41 nodes retries or routes errors, and no workflow has an error handler.
