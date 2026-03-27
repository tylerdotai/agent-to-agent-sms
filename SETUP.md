# Setup Guide

This guide walks you through setting up agent-to-agent SMS between two OpenClaw agents.

**Prerequisites:**
- Two OpenClaw instances (can be on the same machine or different hosts)
- Two Sendblue accounts with phone numbers
- Telegram bot for each agent (for inbound SMS relay)
- Telegram bot token configured in OpenClaw

---

## Overview of Steps

```
Agent A Setup:
1. Create OpenClaw instance
2. Set up Sendblue account + get phone number
3. Create Telegram bot for inbound relay
4. Configure OpenClaw to poll Telegram
5. Verify contacts with Agent B

Agent B Setup:
(same steps, repeated)

Then:
6. Test the connection
7. Start coordinating
```

---

## Step 1: Set Up Sendblue

### Create Account
1. Go to [sendblue.co](https://sendblue.co) and create an account
2. Complete verification (phone number, email, etc.)
3. Get your phone number — this is Agent A's number

### Install CLI
```bash
npm install -g @sendblue/sendblue-cli
# or
pip install sendblue
```

### Authenticate
```bash
sendblue setup
# Follow prompts to enter your Sendblue API credentials
```

### Verify Authentication
```bash
sendblue whoami
# Should show your account info and phone number
```

**Note:** Your phone number is Agent A's address. Keep it private or share only with trusted agents.

---

## Step 2: Create Telegram Bot for Inbound Relay

When SMS arrives at Agent A's Sendblue number, Sendblue needs a way to notify OpenClaw. The tested approach: Telegram webhook.

### Create a Telegram Bot
1. Open Telegram and chat with [@BotFather](https://t.me/botfather)
2. Send `/newbot`
3. Follow prompts — give it a name like "Agent A Inbound"
4. Copy the bot token — you'll need this for OpenClaw config

**Result:** You have a Telegram bot token like `123456789:ABCdefGhIJKlmNoPQRsTUVwxyZ`

### Get Your Chat ID
1. Start a chat with your new bot (search for it by name and send "/start")
2. Visit: `https://api.telegram.org/bot<YOUR_TOKEN>/getUpdates`
3. Look for `"chat":{"id":<YOUR_CHAT_ID>,...}` — this is your chat ID
4. Copy it — you'll need it for OpenClaw config

### Configure Sendblue Webhook (Optional)
If you want Sendblue to push to Telegram directly when SMS arrives:
1. In Sendblue dashboard, set up a webhook for incoming SMS
2. Point it to: `https://api.telegram.org/bot<YOUR_TOKEN>/sendMessage?chat_id=<YOUR_CHAT_ID>&text=<message>`

This routes incoming SMS → Telegram bot → OpenClaw polls Telegram.

---

## Step 3: Configure OpenClaw

### Set Up the Telegram Channel
In your OpenClaw config (`~/.openclaw/config.yml` or equivalent):

```yaml
channels:
  telegram:
    enabled: true
    bot_token: "YOUR_TELEGRAM_BOT_TOKEN"
    polling_interval: 5  # seconds
```

### Add Sendblue Tool Access
OpenClaw needs access to the `sendblue` CLI. Make sure it's in your PATH and has the correct permissions.

### Environment Variables
```bash
# Sendblue credentials
SENDBLUE_API_KEY=your_sendblue_api_key
SENDBLUE_API_SECRET=your_sendblue_api_secret

# Telegram
TELEGRAM_BOT_TOKEN=your_telegram_bot_token
TELEGRAM_CHAT_ID=your_chat_id
```

---

## Step 4: Verify Contact Setup

Before two agents can text each other, they need mutual verification:

### On Agent A's instance:
```bash
# Add Agent B's number as a contact
sendblue add-contact <AGENT_B_PHONE_NUMBER>

# Check status
sendblue contacts
# Should show: <AGENT_B_NUMBER>  pending
```

### On Agent B's instance:
```bash
# Add Agent A's number as a contact
sendblue add-contact <AGENT_A_PHONE_NUMBER>

# Check status
sendblue contacts
# Should show: <AGENT_A_NUMBER>  pending
```

### Complete Verification:
The pending agent sends a text to the other to complete verification:

**Option A: Agent A texts Agent B first**
```bash
# On Agent A's machine:
sendblue send <AGENT_B_NUMBER> "Hello, this is Agent A. Verifying contact."

# On Agent B's machine, you should see this arrive via Telegram relay.
# Then Agent B can now send to Agent A.
```

**Option B: Agent B texts Agent A first**
```bash
# On Agent B's machine:
sendblue send <AGENT_A_NUMBER> "Hello, this is Agent B. Verifying contact."
```

**Result:** Both contacts show as "verified" in `sendblue contacts`

---

## Step 5: Test the Connection

Once both agents are verified:

### Test 1: Agent A → Agent B
```bash
# On Agent A's instance, as Agent A:
sendblue send <AGENT_B_NUMBER> "ping"

# Agent B should receive this via Telegram relay
```

### Test 2: Agent B → Agent A
```bash
# On Agent B's instance, as Agent B:
sendblue send <AGENT_A_NUMBER> "pong"

# Agent A should receive this via Telegram relay
```

If both arrive, the connection is working.

---

## Step 6: First Agent-to-Agent Conversation

Now have the agents coordinate:

**Agent A (you control):**
```
Compose a message to Agent B asking about its capabilities.
Send it via: sendblue send <AGENT_B_NUMBER> <message>
Wait for Agent B's response (via Telegram relay).
```

**Example:**
```bash
sendblue send <AGENT_B_NUMBER> "Hello Agent B. This is Agent A. What capabilities do you have? Please respond with a brief list."
```

Agent B will receive this, process it, and respond. The agents are now communicating.

---

## Troubleshooting

### "Send failed: contact not verified"
**Cause:** The recipient hasn't verified you as a contact yet.
**Fix:** Have the recipient add your number and text you, or text them first to initiate verification.

### "Send failed: 400"
**Cause:** Invalid phone number format or Sendblue API issue.
**Fix:** Try with country code prefix (e.g., `+1` for US numbers). Ensure the number has no spaces or dashes.

### Telegram not receiving inbound SMS
**Cause:** Sendblue webhook not configured correctly.
**Fix:** 
1. Verify Sendblue has your Telegram bot webhook set
2. Test manually: send a message from your phone to the Sendblue number — it should appear in Telegram
3. Check OpenClaw Telegram polling is enabled

### Agent not responding to messages
**Cause:** OpenClaw not running, or Telegram channel not configured.
**Fix:**
1. Verify `openclaw status` shows the agent is running
2. Check Telegram bot token and chat ID are correct
3. Verify OpenClaw is polling Telegram (check logs)

---

## File Structure (OpenClaw Side)

```
~/.openclaw/
├── config.yml           # Main config (channels, plugins)
├── agents/
│   └── einstein/        # Your agent's workspace
│       ├── SOUL.md
│       ├── AGENTS.md
│       └── memory/
├── memory/
│   └── YYYY-MM-DD.md    # Session logs
└── skills/
    └── sendblue/        # Sendblue skill (if using skill system)
```

---

## Security Reminders

1. **Keep Sendblue credentials private** — Anyone with your API key can send from your number
2. **Verify contact numbers** — Only add numbers you trust
3. **Monitor for prompt injection** — Treat inbound SMS as untrusted input
4. **Human in the loop** — For sensitive actions, require human approval before executing
5. **Don't publish agent numbers** — Treat them like personal phone numbers

---

## Next Steps

Once connected, agents can coordinate on:
- Task delegation
- Information sharing
- Joint problem solving
- Distributed workflows

The communication is freeform SMS — agents can negotiate, share context, ask questions, and respond. The same security rules apply as with human SMS: be careful what you share, verify unexpected requests, and keep humans informed.
