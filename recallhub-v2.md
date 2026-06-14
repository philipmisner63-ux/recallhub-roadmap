# RecallHub v2 — Working Document

**Status:** Living document. Updated as we come up with ideas. Not a final spec.
**Last touched:** 2026-06-14

---

## North Star

v2 is **two things at once**:

1. **Iterating the existing v1 tiers** so they're the best chat-memory product in their weight class.
2. **Adding a fourth tier — "RecallHub for Agents" (developer / agent builder)** — that uses the same backend but targets a different buyer.

The two surfaces share infrastructure (MCP server, SQLite ingest, semantic search) but have separate GTM, pricing, and UI.

---

## Strategic stance (as of 2026-06-14)

**Ship v1. Polish v1. Then brainstorm v2-for-agents seriously.**

Reasoning: when we deeply brainstorm something together, it develops in unexpected ways. Trying to design v2-for-agents from a doc without doing the work is a roadmap-promise trap. Get v1 in the store, used, generating signal — *then* spend the time on the v2 design with real data.

**What this means operationally:**

- v1.x gets maintenance-mode updates: bug fixes, BYOK onboarding polish, the click-to-import UX, the free-tier retention bump, the upgrade prompts.
- The new engineering work for v2 is just: the agent session JSONL importer + CLI. That's the deliverable. The tier comes later, after signal.
- v2-for-agents design work is **deferred** until we've shipped, learned from real usage, and have time to think.

---

## v1.x — current tier iteration

### Free tier (extension only)

- **Today:** 30 days local IndexedDB retention.
- **Decision needed (Hermes recommends bump to unlimited local):** the 30-day cap is artificial — the storage is local, costs us nothing on the backend, and creates a real friction moment for casual users.
- **Proposed:** bump to unlimited local (browser-imposed quota is the actual ceiling, typically 10-60% of free disk per origin). Add a storage hint in the popup: "2.3 GB used · ~127 days captured · ~28% of 8 GB browser quota."
- **Safer middle ground if "unlimited" feels too generous:** 1 year.
- **Cost:** zero backend. Dev work: ~2 hours (remove the cleanup job, add the storage hint UI).

### Pro Local ($59 one-time)

- **Today:** 7-day trial via email-only (no card). Day-6 Resend reminder with two CTAs. After expiry, captured conversations stay in IndexedDB but `/v1/ingest/conversation` rejects without valid key.
- **Open question:** the 5-step trial flow is leaky (visit site → email → install app → paste key → paste token in extension). Is this intentional (asymmetry with Pro Cloud's card-on-file is the design) or unoptimized (every hand-off is drop-off)? Need conversion data to answer.
- **Proposed v1.x iteration:** ship the trial, get 2-4 weeks of data on completion rate at each step, optimize the drop-off steps.
- **Past conversation import (the "click-to-import" UX):** ships in v3.2.7+ as a hint in the popup. Users click past conversations in the AI's sidebar, the extension captures each.

### Pro Cloud ($10/mo)

- **Today:** 7-day trial via Lemon Squeezy, card-required, "stay" default. Subscription auto-renews unless cancelled.
- **Day-6 reminder** is handled by LS, not Resend.
- **BYOK story:** 6 providers (OpenAI, Anthropic, OpenRouter, Groq, Mistral, Gemini). Onboarding is per-provider — could be smoother with a "pick a default, swap anytime" bundle.
- **Proposed v1.x iteration:** simplify the BYOK onboarding to a "pick a default provider, swap anytime" flow. Keep the 7-day trial, keep the "stay" default — the friction at the start is the value signal.

### Upgrade prompts (cross-tier)

When memory reaches a certain point, prompt the user to upgrade to Pro Cloud. **Three trigger types:**

1. **Storage threshold (primary, ~70% of browser quota):** "Local storage at 73%. Pro Cloud = unlimited sync + semantic search + 7-day free trial."
2. **Conversation count (engagement signal, 100+/250+ conversations):** "250+ captured. Pro Cloud = find by meaning + search across devices + unlimited history."
3. **Search upgrade (contextual, when text search returns N results):** "Found 4 matches for 'kubernetes'. Pro Cloud would also find by meaning."

**Timing rules:**
- Max once per 2 weeks per user (track "dismissed until" in storage)
- "Maybe later" persists 30 days
- After upgrade, never prompt again
- Don't prompt on first install, first conversation, or during focused tasks
- Show in the popup, not as modal/banner

**CTA target:** existing `/upgrade#pro-cloud` page to start. Optimize to direct LS trial checkout later with conversion data.

**Implementation:** ~4-8 hours of extension work. v3.2.7 or v3.2.8.

### Re-trial cooldown

- **Today:** 90 days. Any trial row for an email in the last 90 days → 409.
- **This is fine.** No change proposed.

---

## v2.0 — RecallHub for Agents (new tier, deferred design)

### Why a new tier (not a feature in Pro Cloud)

1. **Different buyer** = different GTM. Pro Cloud sells to end users who care about their chats. Agent tier sells to developers who care about their agents' behavior. Different copy, different channels, different sales motion. Two-tier co-marketing is a mess.
2. **Different cost structure.** Chat memory: ~5-50 messages/session, text, small. Agent memory: hundreds to tool calls per session, structured data, large. A flat $10/mo either bankrupts on heavy users or throttles the differentiator.
3. **Different unit economics, different ceiling.** End-user Pro Cloud at $10/mo caps at, say, $120/yr per user. Developer Pro at $30-50/mo per agent, with a team running 5-20 agents, is $1.5K-$12K/yr per team. Different business.
4. **Brand clarity.** "RecallHub for end users" and "RecallHub for agents" are different things. Forcing them into one tier confuses the buyer.

### What the tier is (one-paragraph framing)

RecallHub for Agents is the **memory layer for AI agents themselves**. Same backend as the chat tier, but the data model is "agent runs" (text + tool calls + reasoning) instead of "conversations" (text only). The buyer is an AI agent builder — someone running Claude Code, Codex, custom LangChain/LlamaIndex stacks. They import their framework's session JSONL, get semantic search across tool calls, and can inspect/halt their agent's memory.

### Buyer, GTM, pricing (TBD)

- **Buyer:** AI agent builders (developers running Claude Code, Codex, custom stacks)
- **Pricing model:** TBD. Options:
  - Per-agent, $30-50/mo
  - Usage-based (storage + search volume)
  - Tiered by team size
- **GTM:** developer-focused. Different marketing from the consumer tiers. GitHub README, HN, dev.to posts, integration with agent frameworks.

### The agent session importer (the actual deliverable)

This is the concrete feature that ships before the tier is productized:

- **CLI:** `recallhub import agent-session <file.jsonl>`
- **Input:** framework session JSONL (already exposed by `hermes sessions export` and similar from other frameworks)
- **Parses:** line-by-line, extracts `tool_use` and `tool_result` records
- **Maps:** to a new `agent_runs` table in the existing SQLite
- **Captures more than chat:** tool calls, intermediate reasoning, file edits, search queries
- **Searchable:** same semantic search infrastructure, but across tool calls not just chat
- **Storage:** lives in user's local context-layer (Pro Local) or in their Pro Cloud account

**Why ship the importer before the tier:** tools and tiers are different. The tool is real value that anyone with the data can use. The tier is a pricing wrapper. Build the tool, see who shows up, then productize the tier around actual usage patterns.

### MCP server surface

The RecallHub MCP server (`mcp__recallhub_recallhub_*`) already exposes:
- `search` — query memory
- `add_fact` — write a fact
- `save_handoff` — save structured handoff
- `search_facts` — query atomic facts
- `status` — health check

For the agent tier, this is the API surface. The CLI importer is bulk-write; the MCP server is live-write. Same data model, different paths in.

### Open questions (v2 tier design — to resolve when we brainstorm)

- [ ] Pricing model (per-agent, usage-based, or tiered)
- [ ] Naming (RecallHub for Agents? Developer Pro? RecallHub Agent?)
- [ ] UI: same app as v1 with a different mode, or a separate product?
- [ ] Multi-agent memory permissions: can agent A read agent B's memory? (Default no, with explicit share.)
- [ ] When does the importer need an "agent identity" primitive vs just a user identity?
- [ ] Frame for the YC narrative: "we're the memory layer for AI" (chat + agent surfaces).

---

## Backlog (add to this list as we come up with ideas)

- [ ] Free tier: bump retention from 30 days to unlimited (or 1 year)
- [ ] Free tier: add storage hint UI in popup
- [ ] Extension popup: add upgrade prompts (storage threshold, conversation count, search upgrade)
- [ ] Extension popup: add click-to-import past conversations hint
- [ ] Pro Local: optimize the 5-step trial flow (after getting conversion data)
- [ ] Pro Cloud: simplify BYOK onboarding ("pick a default, swap anytime")
- [ ] CLI: `recallhub import agent-session <file.jsonl>`
- [ ] SQLite: `agent_runs` table for tool calls + reasoning
- [ ] Search: semantic across tool calls (not just chat)
- [ ] Developer tier: pricing + GTM (deferred)
- [ ] MCP server: agent identity primitive (deferred)

---

## Things I've changed my mind about (or want clarity on)

- Originally I said "30 days is too short" — this is about Free tier retention, not Pro trials. Trials are 7 days, that's separate.
- Originally I said "click-to-import limited to 90 days" — wrong. Click-to-import is unlimited; 90 days is the re-trial cooldown, not an import window.
- The 5-step Pro Local trial flow may be a real conversion leak. Need data to confirm.

---

## What I want clarity on (open for Philip)

- Is the 5-step Pro Local flow intentional asymmetry or unintentional friction?
- Naming for the Developer tier — your call, you brand better than I do.
- Is the consumer funnel still priority #1, or does the agent angle pull rank?
- When do we actually have time to brainstorm v2-for-agents seriously? (My guess: after the trial/funnel data is in.)
