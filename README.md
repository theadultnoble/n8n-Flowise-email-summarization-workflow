# n8n-Flowise-email-summarization-workflow
## Gmail → LLM Summary → WhatsApp Briefs (n8n + Flowise)

A workflow that watches Gmail for new emails, extracts and cleans the email content (preferably HTML), sends it to a Flowise Agentflow for summarisation, then delivers a structured brief to WhatsApp via the WhatsApp Cloud API.

This is built for fast inbox triage: you get the important context + action items without rereading long threads.

---

## What this does

- Polls Gmail on a schedule (e.g., every minute)
- Fetches the full email content (not just the snippet)
- Extracts and cleans HTML (removes quoted history, signatures, noisy markup)
- Sends the cleaned email to a Flowise Agentflow that returns a strict, structured summary
- Formats the result into a WhatsApp-compatible message payload
- Sends to WhatsApp via the WhatsApp Cloud API
- Prevents duplicates (message-id based) and retries transient failures

---

## Architecture

**Gmail** → **n8n (trigger + orchestration)** → **Flowise (Agentflow summarizer)** → **n8n (format + WhatsApp payload)** → **WhatsApp Cloud API**

Flowise is responsible only for generating the summary text (and optionally a small structured JSON block). n8n handles:
- authentication + polling
- content extraction/cleanup
- routing + message formatting
- delivery + retries + dedupe
- logging

---

## Why HTML extraction matters

Gmail messages often contain:
- quoted replies (entire conversation history)
- signature blocks
- formatting noise that dilutes meaning
- inline images / attachments

If you summarize only the plain `text` field or a `snippet`, the model may miss key sections or over-focus on irrelevant parts.

This workflow extracts the **HTML body**, then transforms it to a cleaner text version before summarization.

---

## Output format (WhatsApp brief)

The WhatsApp message is intentionally structured so it’s scannable:

- **Subject**
- **Summary (3–5 bullets)**
- **Action Items**
- **Dates/Deadlines**
- **Links (optional)**

Example:

Subject: Vendor contract review needed

Key points:
- Legal wants final review on contract v3
- Vendor updated pricing clause (section 4.2)
- Target signature date is Feb 2

Action items:
- Review clauses 4.2 and 7.1
- Reply with approval or requested edits

Dates:
- Deadline: Feb 2

---

## Prerequisites

### Accounts/access
- A Gmail account with access to the inbox you’re monitoring
- A Meta developer app with **WhatsApp Cloud API** enabled (phone number + permanent token)
- A Flowise instance running (local or hosted)
- An n8n instance running (local or hosted)

### Recommended runtime
- Node.js (if self-hosting n8n / Flowise)
- A stable network connection (OAuth/Gmail requires reliable DNS + outbound access)

---

## Environment variables

Use n8n credentials where possible. Where env vars are required, set these:

### Gmail (n8n credential recommended)
- `GMAIL_CLIENT_ID`
- `GMAIL_CLIENT_SECRET`
- `GMAIL_REDIRECT_URI`

### Flowise
- `FLOWISE_BASE_URL` (e.g., `http://localhost:3000`)
- `FLOWISE_AGENTFLOW_ENDPOINT` (your Agentflow API endpoint)
- `FLOWISE_API_KEY` (if enabled)

### WhatsApp Cloud API
- `WHATSAPP_PHONE_NUMBER_ID`
- `WHATSAPP_ACCESS_TOKEN`
- `WHATSAPP_TO` (destination phone number in international format)
- `WHATSAPP_API_VERSION` (e.g., `v20.0`)

---

## n8n workflow overview (nodes)

Below is the typical node flow. Your exact nodes may differ depending on how you authenticate and how Flowise is exposed.

1. **Gmail Trigger (polling)**
   - Runs every minute (or desired interval)
   - Triggers on new messages

2. **Get Message / Fetch Full Email**
   - Ensures you retrieve full payload (not only snippet)

3. **HTML Extract + Cleanup**
   - Pull HTML body
   - Convert to clean text
   - Remove signatures / quoted replies
   - Optional: limit to most recent reply segment

4. **Deduping**
   - Uses Gmail `messageId` (or `threadId` + `internalDate`) to ensure you never reprocess the same mail
   - Store processed IDs (Data Store node, Redis, DB, or a simple file-based approach)

5. **Flowise Call (HTTP Request)**
   - Sends cleaned email text to your Flowise Agentflow
   - Receives summary output (text or JSON)

6. **WhatsApp Payload Builder**
   - WhatsApp message schemas are mutually exclusive per message type
   - Build exactly one payload type (usually `text`)

7. **Send WhatsApp Message (HTTP Request)**
   - POST to WhatsApp Cloud API endpoint

8. **Logging**
   - Store execution logs and failures (n8n execution history is often enough)

---

## Flowise Agentflow

### Agentflow responsibilities
- Accept cleaned email text from n8n
- Return predictable output using a strict schema
- Avoid hallucinations
- Preserve names, dates, numbers, and links

### Suggested system message (high reliability)

Use a system message that is structured like documentation: role, constraints, schema, and failure-handling rules.

Example:

- Role: “You summarize emails for fast triage.”
- Constraints:
  - Do not invent facts.
  - If action items are not explicit, output “No explicit action items.”
  - Preserve dates/numbers/names exactly.
  - Ignore signatures and quoted history.
- Output schema (fixed order):
  - Subject
  - Key points (3–5 bullets)
  - Action items
  - Dates/Deadlines
  - Links (if any)

### Prompt input

n8n sends something like:

- `subject`
- `from`
- `date`
- `clean_text` (the cleaned email body)

Flowise generates the summary based on `clean_text` first, with subject metadata used only for context.

---

## HTML cleanup strategy (practical)

Recommended approach in n8n:

1) Prefer Gmail’s HTML body (when available)  
2) Strip tags and normalize whitespace  
3) Remove common reply separators like:
- “On ___ wrote:”
- “From: … Sent: … To: … Subject: …”
- “--- Forwarded message ---”
4) Remove signature blocks heuristically:
- sections after “Thanks,” / “Best,” / “Regards,” if trailing content is mostly contact info
5) Hard-limit length (optional) to keep LLM cost and latency stable

---

## WhatsApp message schema note

WhatsApp Cloud API requires **one** message type per request (text, template, image, etc.).  
This constraint often forces branching logic in n8n because each message type has different required fields.

In this workflow, use **text** messages unless you specifically need templates.

---

## Reliability features

### Deduping
- Store processed Gmail message IDs
- Ignore already-processed IDs

### Retries
- Retry Flowise call on 429/5xx with backoff
- Retry WhatsApp send on transient failures

### Timeouts
- Set timeouts on HTTP nodes to avoid workflow hangs

### Rate limiting
- If you poll every minute, ensure you handle bursts (e.g., multiple new emails)
- Add a “Split in Batches” node when needed

---

## Performance tips (n8n)

- Avoid heavy Merge modes unless needed  
  If you use Merge with “combine by all possible combinations,” it can explode the number of items and slow down execution.
- If your Merge output “shows binary first,” ensure you reference the correct JSON path and not the binary payload.
- Reduce LLM input size by cleaning HTML and limiting quoted history.

---

## Common issues & fixes

### 1) `Error: getaddrinfo ENOTFOUND oauth2.googleapis.com`
This is a DNS/network resolution failure when your instance can’t reach Google OAuth endpoints.

Fix checklist:
- Confirm internet access from the machine/container
- Verify DNS settings (try a different DNS resolver)
- If running in Docker, confirm the container has outbound access
- Check firewall/proxy restrictions

### 2) Gmail output shows only `snippet`, not full text
You’re likely using a trigger output that includes a snippet by default.

Fix:
- Use a “Get Message” step after the trigger to fetch the full body
- Prefer HTML part if available

### 3) LLM summary focuses only on an image attachment
This often happens when:
- your text body is empty/short, and the attachment is the only “content”
- you’re passing metadata about attachments more prominently than the email body

Fix:
- Ensure your cleaned body is populated
- Instruct the system message to prioritise body text and treat attachments as optional context

### 4) TypeScript error in n8n Function node
If you return a single object instead of an array of items, you’ll see type mismatch errors.

Rule:
- Always return an array of items: `return [{ json: ... }]`

---

## Minimal payload examples

### Flowise request (from n8n)
```js
{
  subject: $json.subject,
  from: $json.from,
  date: $json.date,
  body: $json.cleanedBody
}
