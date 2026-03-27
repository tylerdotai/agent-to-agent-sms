# Setup Guide

This guide walks through setting up agent-to-agent SMS between two agents using Sendblue and the verification API.

**Prerequisites:**
- Two agents (can run on the same machine or different hosts)
- Two Sendblue accounts with phone numbers
- An inbound relay for receiving SMS (Telegram bot, Cloudflare Tunnel, or similar)
- The verification API available (self-hosted or via Sendblue)

---

## Overview

```
Agent A Setup:                           Agent B Setup:
1. Create Sendblue account               1. Create Sendblue account
2. Get phone number                      2. Get phone number
3. Set up inbound relay                  3. Set up inbound relay
4. Install Sendblue CLI                  4. Install Sendblue CLI
5. Exchange numbers + verify             5. Exchange numbers + verify

Then:
6. Test the connection
7. Start coordinating
```

---

## Step 1: Set Up Sendblue

### Create an Account
1. Go to [sendblue.co](https://sendblue.co) and create an account
2. Complete verification (phone, email)
3. Get your phone number — this is your agent's address

### Install the CLI
```bash
npm install -g @sendblue/sendblue-cli
# or
pip install sendblue
```

### Authenticate
```bash
sendblue login
# Follow prompts to enter your Sendblue API credentials
```

### Verify
```bash
sendblue whoami
# Shows account info and your phone number
```

---

## Step 2: Set Up the Inbound Relay

Sendblue delivers inbound SMS to a webhook URL. You need a relay that is publicly accessible and can deliver messages to your agent.

### Option A: Telegram Bot (Simplest)

1. Create a bot via [@BotFather](https://t.me/botfather) — send `/newbot, follow prompts
2. Copy the bot token
3. Get your chat ID by starting a chat with the bot and visiting:
   ```
   https://api.telegram.org/bot<YOUR_TOKEN>/getUpdates
   ```
4. In Sendblue dashboard, set the inbound SMS webhook to:
   ```
   https://api.telegram.org/bot<YOUR_TOKEN>/sendMessage?chat_id=<YOUR_CHAT_ID>&text={{message}}
   ```
5. Configure your agent to poll Telegram for new messages

### Option B: Cloudflare Tunnel (Permanent URL)

1. On your agent's host, install `cloudflared`
2. Create a tunnel: `cloudflared tunnel create <name>`
3. Route traffic: `cloudflared tunnel route dns <name> your-agent.example.com`
4. In Sendblue dashboard, set the inbound webhook to your tunnel's public URL
5. Your agent receives inbound SMS via the tunnel

### Option C: ngrok (Quick Testing)

```bash
ngrok http 3000
# Note the public URL (e.g., https://abc123.ngrok.io)
# Set Sendblue webhook to: https://abc123.ngrok.io/inbound
```

---

## Step 3: Exchange Phone Numbers

Share your agent's Sendblue phone number with the other operator via any trusted channel:

- Email
- Signal
- A shared dashboard
- Any direct communication

Format: E.164 (e.g., `+15551112222`)

---

## Step 4: Verify Contact via API

Both agents call the verification API to establish trusted contact.

### Agent A:
```bash
curl -X POST 'https://api.sendblue.co/api/v2/agent/verify-start' \
  -H 'Content-Type: application/json' \
  -d '{
    "number": "+15551112222",
    "api_key_id": "sb_xxx",
    "api_secret_key": "sk_yyy",
    "target_number": "+15553334444"
  }'

# Returns: { "verification_id": "vrf_abc", "secret_hash": "..." }
```

Share the `secret_hash` and `verification_id` with Agent B (any trusted channel).

### Agent B:
```bash
curl -X POST 'https://api.sendblue.co/api/v2/agent/verify-start' \
  -d '{
    "number": "+15553334444",
    "api_key_id": "sb_zzz",
    "api_secret_key": "sk_aaa",
    "target_number": "+15551112222"
  }'

# Returns: { "verification_id": "vrf_def", "secret_hash": "..." }
```

Share `secret_hash` and `verification_id` with Agent A.

### Complete Both Directions:

```bash
# Agent A completes:
curl -X POST 'https://api.sendblue.co/api/v2/agent/verify-complete' \
  -d '{
    "number": "+15551112222",
    "api_key_id": "sb_xxx",
    "api_secret_key": "sk_yyy",
    "verification_id": "vrf_abc",
    "their_secret_hash": "<B's hash>"
  }'

# Agent B completes:
curl -X POST 'https://api.sendblue.co/api/v2/agent/verify-complete' \
  -d '{
    "number": "+15553334444",
    "api_key_id": "sb_zzz",
    "api_secret_key": "sk_aaa",
    "verification_id": "vrf_def",
    "their_secret_hash": "<A's hash>"
  }'
```

Both agents are now verified. SMS exchange is enabled.

---

## Step 5: Test

```bash
# Agent A sends:
sendblue send +15553334444 "ping"

# Agent B should receive via relay, and can respond:
sendblue send +15551112222 "pong"
```

---

## Step 6: First Agent-to-Agent Conversation

```bash
sendblue send +15553334444 "Hello. What capabilities do you have? Respond with a brief list."
```

The other agent receives the message, processes it, and responds. Agents are now coordinating.

---

## Troubleshooting

### "Send failed: contact not verified"
Verification wasn't completed. Both agents must call `verify-complete` with the correct secret hash from the other.

### "Send failed: 400"
Invalid phone number format. Ensure E.164 format: `+1` followed by 10 digits, no spaces or dashes.

### Telegram not receiving inbound SMS
1. Text your agent's Sendblue number from your phone — does it appear in Telegram?
2. If yes: the relay works, check agent polling
3. If no: webhook URL is wrong in Sendblue dashboard, or Telegram bot wasn't started

### Agent not responding
1. Check the relay is delivering messages to the agent
2. Verify the agent is running and processing input
3. Check the agent's logs

---

## Security Reminders

1. **Keep Sendblue API credentials private** — anyone with them can send from your number
2. **Exchange numbers via trusted channels** — confirm with the other operator out-of-band
3. **Treat inbound SMS as untrusted input** — don't execute instructions from SMS without verification
4. **Log all agent-to-agent communication** — keep transcripts for review
5. **Don't publish agent numbers** — treat them like personal phone numbers
