# AI Platform History Export Paths

Research compiled by Hermes, 2026-06-13. Each platform's user-facing export
path, file format, data scope, and gotchas. Ordered by how easy the export is
to use as the basis for a programmatic importer.

---

# ⚠️ PREFERRED WORKFLOW (read this first) — click-to-import, not download-to-import

**Most platforms do NOT need a download-import pipeline.** The browser
extension already captures live conversations via `content.js`. When the
user clicks a past conversation in the platform's history sidebar, the
platform re-renders the conversation into the live chat, the extension's
URL observer + MutationObserver fires, and the message is ingested via
`POST /v1/ingest/conversation` with backend-side dedup by content hash.

This means for ~7 of 8 platforms, the "history importer" UX is just:
"open the AI's site, click each past conversation in the sidebar." The
extension does the rest. No download, no parse, no schema.

**When the download path IS needed** (only these cases):
- The platform doesn't have a clickable past-conversation list
- The user wants the "official" archive for compliance / portability
- The live re-render drops specific content (file uploads, images, etc.)
- A platform is missing entirely from the extension's capture list

**Documented below** for completeness + the cases where the download
path IS the right answer. Build order recommendation revised:

1. User docs + popup UI hint (1 day)
2. Click-to-import verification per platform (test in extension)
3. Download-import pipeline only for the platforms that need it

---

## 1. ChatGPT (OpenAI) — ✅ EASY

- **Path:** Settings → Data Controls → Export Data → email download
  (https://chatgpt.com/#settings/DataControls)
- **Format:** ZIP containing `conversations.json` + `chat.html` (rendered) + `user.json` + `shared_conversations.json` + others
- **Time to receive:** queued, email within minutes to a few hours
- **Data scope:** ALL conversations including DALL-E prompts, custom GPTs, shared links
- **File size:** 100-200 MB for active users
- **Schema (conversations.json):**
  ```json
  [
    {
      "title": "string",
      "create_time": "unix-epoch-float",
      "update_time": "unix-epoch-float",
      "mapping": {
        "<message_id>": {
          "id": "string",
          "message": {
            "id": "string",
            "author": { "role": "user|assistant|tool|system" },
            "content": { "content_type": "text|code|multimodal_text", "parts": ["string|array"] },
            "create_time": "unix-epoch-float"
          },
          "parent": "<parent_message_id>|null",
          "children": ["<child_id>"]
        }
      },
      "current_node": "<message_id>"
    }
  ]
  ```
- **Gotchas:**
  - The `mapping` is a DAG, not a flat list. Walk from `current_node` following `parent` chain (or collect all messages, sort by `create_time`).
  - Image attachments: DALL-E images live in `asset_pointer` URLs (signed S3 URLs that expire). Original uploaded images are in `content.parts[]` as image URLs.
  - `chat.html` is a rendered version (good for human review, not for parsing).
  - Code interpreter files: stored externally with `file_citation`/`file_path` pointers.

---

## 2. Claude (Anthropic) — ✅ EASY

- **Path:** Settings → Privacy → Export data
  (https://claude.ai/settings/data-privacy-controls)
  Web only — iOS/Android cannot initiate.
- **Format:** ZIP — confirmed via official docs
- **Time to receive:** queued, email download link
- **Download link expiry:** 24 hours
- **Data scope:** ALL conversation data + user account data
- **Schema:** Standard Anthropic format. The ZIP typically contains:
  - `conversations.json` — array of conversations
  - Per-conversation format: `{ uuid, name, created_at, updated_at, chat_messages: [{ uuid, text, sender, created_at, content, attachments, files }] }`
  - User profile, account info
- **Gotchas:**
  - No official public schema doc, but the format is widely understood (the `chat-exporter` GitHub tools parse it reliably).
  - For Team/Enterprise plans, only the Primary Owner can export the org's data. Individual exports from member accounts return only that member's data.

---

## 3. Gemini (Google) — ⚠️ MODERATE

- **Path:** takeout.google.com → "Deselect all" → check **"My Activity"** (not "Gemini Apps" — the Reddit-threaded trick)
  - Gemini Apps in Takeout exports app config/settings, not conversations.
  - My Activity is the bucket where Gemini conversation events actually live.
- **Format:** JSON (select JSON over the default HTML when prompted — HTML wraps JSON in markup and is a pain to parse)
- **Time to receive:** queued, email
- **Schema:** `My Activity.json` — array of activity records, each with a `title` field, `time` (ISO 8601), and a `products` array (`["Gemini"]`). Conversation content lives in the `details` field or in a separate `Gemini/My Activity/<conversation-id>.json` per-conversation file (depends on export options).
- **Gotchas:**
  - The default Takeout HTML format wraps JSON inside HTML — parse the HTML or specifically request JSON.
  - Gemini Business / Workspace accounts are governed by Workspace data export rules, not personal Takeout.
  - "My Activity" only logs activity when Gemini Apps Activity is enabled in your account.
- **Phoenix Grove tip:** the only reliable way to get conversations is to enable Gemini Apps Activity, wait for activity to be recorded, then export My Activity as JSON.

---

## 4. Microsoft Copilot — ⚠️ MODERATE (CSV, not conversations)

- **Path:** account.microsoft.com/privacy → Privacy → Empower your productivity → Copilot → Your Copilot app activity history → Copilot apps → **Export all activity history**
- **Format:** CSV
- **Time to receive:** queued, download link
- **Schema:** Standard Microsoft privacy CSV: `{ Timestamp, App, Activity Type, Description, ConversationId, ... }`
  - **Not full conversation transcripts.** This is a "what did you do in Copilot" activity log, not a "give me every message" export.
- **For full conversation transcripts:** There is a Copilot Activity Export API mentioned in Microsoft's docs, but it's not user-facing — enterprise admin tier only.
- **Per-session alternative:** No native per-chat export. Workarounds: print to PDF (browser → Print → Microsoft Print to PDF), or browser extension.
- **Gotcha:** the CSV is activity-level metadata, not message content. For a message-level importer, this isn't sufficient on its own — needs the API or third-party capture.

---

## 5. Perplexity — ❌ NO NATIVE BULK EXPORT

- **Path:** No built-in bulk export. Per-thread export only.
- **Per-thread workaround:** Open the thread in a browser, scroll to the top (forces all messages to load), then use a third-party tool.
- **Third-party tools:**
  - `simwai/perplexity-ai-export` on GitHub — exports to Markdown files
  - Browser extensions (search "Perplexity export" in Chrome Web Store)
  - One-click sync to Airtable (medium writeup)
- **API alternative:** Perplexity has an API but it doesn't expose history retrieval. You'd have to scrape the web UI.
- **Gotcha:** Perplexity has actively resisted a bulk export feature. Multiple feature requests in their community forum have been open for 2+ years.

---

## 6. Grok (xAI) — ⚠️ PARTIAL

- **Path:** Grok mobile app → Settings → Data Controls → "Download data" (per xAI Consumer FAQ)
  - Web: Settings → Data Controls → "Export data" (less reliable per community reports)
- **Format:** JSON
- **Time to receive:** queued, email
- **Schema:** Per `xtrace.ai/blog/export-grok-conversations` and the xAI legal FAQ, the export contains conversation history + user data in a JSON bundle. Schema not officially documented.
- **Gotchas:**
  - Web-only export is incomplete per user reports — mobile app gives fuller export.
  - Multiple third-party userscripts and Chrome extensions exist (e.g., `enhanced-grok-export`) because the official path is unreliable.
  - No official API for history retrieval.

---

## 7. DeepSeek — ❌ NO NATIVE UI EXPORT

- **Path:** No user-facing export UI. Must email support with GDPR-style data request.
- **Email request:** contact@deepseek.com with subject "GDPR Data Export Request"
- **Format:** JSON (per community reports)
- **Time to receive:** Days to weeks (manual process)
- **Third-party tools:** Many — Chrome extensions (search "DeepSeek exporter"), browser scripts, etc.
- **API alternative:** No public API for history.
- **Gotcha:** DeepSeek is the worst case for programmatic import. The only reliable path is third-party browser tools that scrape the live UI.

---

## 8. Meta AI — ⚠️ PARTIAL

- **Path 1 (Meta AI web app):** meta.ai → Settings → Data & privacy → Manage your information → Download your information
- **Path 2 (Messenger / Instagram AI chats):** https://privacycenter.instagram.com/dialog/download-chat-history-with-ai/
  - Or use the `/download-all-ai-info` command in an individual chat with an AI to get a link
- **Format:** JSON (typically)
- **Time to receive:** queued, often HOURS to DAYS for full account
- **Schema:** Standard Meta data download bundle — JSON for each product (Meta AI, Messenger, Instagram) with conversations in their respective folders.
- **Gotchas:**
  - Meta's full data download is huge (10+ years of data) and slow. Expect long waits.
  - Meta AI on WhatsApp has different export rules than Meta AI on the web.
  - Facebook's data download UI has changed multiple times — paths above are current as of 2026 but may break.

---

# Summary: Implementation Priority

| Platform | Native export | Format | Effort to import | Notes |
|---|---|---|---|---|
| **ChatGPT** | ✅ Yes | ZIP/JSON | Low | Best-documented schema, large user base |
| **Claude** | ✅ Yes | ZIP/JSON | Low | Clean format, official docs |
| **Gemini** | ⚠️ My Activity only | JSON | Medium | Have to find the right Takeout checkbox |
| **Copilot** | ⚠️ Activity CSV | CSV | High | No full message export; CSV is metadata only |
| **Perplexity** | ❌ No | n/a | High | Third-party tools only |
| **Grok** | ⚠️ Mobile-only | JSON | Medium | Mobile app gives better export than web |
| **DeepSeek** | ❌ No | n/a | High | GDPR email request only, days to weeks |
| **Meta AI** | ⚠️ Bundled | JSON | Medium | Slow but works; bundle includes other products |

**Top 3 to build first (covers ~80% of user data):**
1. **ChatGPT** — biggest user base, cleanest format
2. **Claude** — same population, clean format
3. **Gemini** — Google's user base is huge, JSON is parseable

**The other 5** are deprioritized until we see real demand — the engineering effort per platform is significant, and the formats are messier.

---

# Appendix: Things the research didn't settle (verify before implementing)

1. **Claude's exact conversations.json schema** — official docs only mention the path, not the field names. Confirm by running an actual export before writing the parser.
2. **Grok's mobile-only export** — couldn't confirm if the mobile-only claim is current. Test from both web and mobile before declaring.
3. **Meta AI's exact conversation file location** in the data bundle. Could be `messages/` or `ai_conversations/` — verify by running an actual download.
4. **Perplexity's "no API"** — confirmed as of search date but Perplexity adds features quickly. Re-check before investing in a scraper.
