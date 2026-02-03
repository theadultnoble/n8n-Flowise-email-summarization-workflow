# Email-summary-automation-workflow

An n8n automation workflow that watches a connected Gmail account for new emails, extracts and cleans the email content (preferably HTML), sends it to a Flowise Agentflow to categorize the type of email it is act as an orchestrator, and route the email to a custom sub-agent to perform the summarisation. It then returns the generated structured summary back to n8n to send the summary to the users WhatsApp DM via the WhatsApp Cloud API.

This is built for fast inbox triage: you get the important context + action items without rereading long threads.

## What this does

- Polls Gmail on a schedule (e.g., every minute)
- Fetches the full email content (not just the snippet)
- Extracts and cleans HTML (removes quoted history, signatures, noisy markup)
- Sends the cleaned email to a Flowise Agentflow that returns a strict, structured summary
- Formats the result into a WhatsApp-compatible message payload
- Sends to WhatsApp via the WhatsApp Cloud API
- Prevents duplicates (message-id based) and retries transient failures

## Why HTML extraction matters

Gmail messages often contain:

- quoted replies (entire conversation history)
- signature blocks
- formatting noise that dilutes meaning
- inline images / attachments

If you summarize only the plain `text` field or a `snippet`, the model may miss key sections or over-focus on irrelevant parts.

This workflow extracts the **HTML body**, then transforms it to a cleaner text version before summarization.

## Architecture

**Gmail** → **n8n (trigger + orchestration)** → **Flowise (Agentflow summarizer)** → **n8n (format + WhatsApp payload)** → **WhatsApp Cloud API**
Flowise is responsible only for generating the summary text.
n8n handles:

- authentication + polling
- content extraction/cleanup
- routing + message formatting
- delivery + retries + dedupe
- logging

### Hosting the automation

This automation is containerized using Docker and hosted on a private free Oracle Cloud Virtual Machine. This ensures zero downtimes and an easily reproducable set up.

**Step 1- Create the Oracle Cloud “Always Free” VM**

1. Create an Oracle Cloud account.
2. Go to Compute → Instances → Create instance.
3. Image: Ubuntu 22.04 (or 24.04).
4. Shape: pick an Always Free eligible shape:
   - Ampere (ARM) is usually best if available.
5. Networking:
   - Create/choose a VCN + subnet
   - Assign a public IPv4
6. SSH keys:
   - Add your public key (or let OCI generate one and download it)

Note: Public IP & Username (usually `ubuntu`)

SSH in:

```bash
ssh -i /path/to/key ubuntu@YOUR_PUBLIC_IP
```

The OCI instance is behind a VCN firewall so ports for both n8n(TCP 80) and Flowise(TCP 443) must be allowed. A reverse proxy is recommended, so keep 5678/3000 closed to the public.

**Step 2- Install Docker + Docker Compose**
In the Virtual Machine:

```bash
sudo apt update
sudo apt -y upgrade

# Docker
curl -fsSL https://get.docker.com | sudo sh

# Allow your user to run docker without sudo (re-login after)
sudo usermod -aG docker $USER
newgrp docker

# Docker compose plugin
sudo apt -y install docker-compose-plugin
docker compose version
```

Create a project folder + persistent volumes:

```bash
mkdir -p ~/automation-stack
cd ~/automation-stack
mkdir -p n8n_data flowise_data postgres_data
```

**Step 3- docker-compose.yml**

This runs-

- _Postgress_: For n8n state persistence across VM reboots;
  1. Crash-safe execution
  - handles concurrent writes
  - survives restarts cleanly
  - preserves partial execution state
  2. Reliable polling + retries
  - frequent writes
  - overlapping executions
  - retry logic when Gmail or WhatsApp fails

- _n8n_: Main automation flow;
  Replace the The host IP with the VM's OCI instance public IP. Also replace the passwords + encryption key with real values.
  Generate a strong encryption key:

```bash
openssl rand -hex 32
```

- _Flowise_: Generate summary based on email type.

**Step 4 - Start the stack**

```bash
docker compose up -d
docker compose ps
```

## Saving Inference Cost with Input format (Using TOON )

The Input format of the LLMs on flowise utilize TOON over JSON format to conserve token processing and cost. It is intentionally structured so it’s scannable.

For example:

```YAML

```

## Prerequisites

### Accounts/access

- A Gmail account with access to the inbox you’re monitoring
- A Meta developer app with **WhatsApp Cloud API** enabled (phone number + permanent token)
- A Flowise instance running (local or hosted)
- An n8n instance running (local or hosted)

### Recommended runtime

- Node.js (if self-hosting n8n / Flowise)
- A stable network connection (OAuth/Gmail requires reliable DNS + outbound access)

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

## n8n workflow overview (nodes)

Below is the typical node flow.

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
   - Sends cleaned email text in TOON format to your Flowise Agentflow
   - Receives summary output (text or TOON)

6. **WhatsApp Payload Builder**
   - WhatsApp message schemas are mutually exclusive per message type
   - Build exactly one payload type (usually `text`)

7. **Send WhatsApp Message (HTTP Request)**
   - POST to WhatsApp Cloud API endpoint

8. **Logging**
   - Store execution logs and failures (n8n execution history is often enough)

## Flowise Agentflow

### Agentflow responsibilities

- Accept cleaned email text(TOON) from n8n
- Return predictable output using a strict schema
- Avoid hallucinations
- Preserve names, dates, numbers, and links

### Suggested system message (high reliability)

Use a system message that is structured like documentation: role, constraints, schema, and failure-handling rules.
A fitting system message exists inside the n8n and Flowise folder in this repo.

### Prompt input

n8n sends something like:

- `subject`
- `from`
- `date`
- `clean_text` (the cleaned email body)

Flowise generates the summary based on `clean_text` first, with subject metadata used only for context.

---

## WhatsApp message schema note

WhatsApp Cloud API requires **one** message type per request (text, template, image, etc.).  
This constraint often forces branching logic in n8n because each message type has different required fields.

In this workflow, use **text** messages unless you specifically need templates.

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

- Ensure bursts handling (e.g., multiple new emails)

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
