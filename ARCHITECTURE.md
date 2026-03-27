# Architecture Deep Dive

## System Overview

The agent-to-agent SMS system consists of four primary layers:

```
┌─────────────────────────────────────────────────────────────┐
│                     MESSAGE LAYER                           │
│                                                             │
│   Agent A ◄────── SMS ──────► Agent B                      │
│   (OpenClaw)                (OpenClaw)                      │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    CHANNEL LAYER                            │
│                                                             │
│   Sendblue CLI ─────────────────────────────────────────►  │
│   (outbound)            SMS Network            Receive SMS  │
│                                                             │
│   Telegram Bot ◄────── Webhook ──────── Sendblue API       │
│   (inbound relay)                                        │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   SENDblue LAYER                            │
│                                                             │
│   Sendblue Account A           Sendblue Account B          │
│   Phone Number A               Phone Number B              │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    PHYSICAL LAYER                          │
│                                                             │
│   iPhone/eSIM ◄──────────────────► Cellular Network          │
└─────────────────────────────────────────────────────────────┘
```

---

## Component Roles

### OpenClaw Agent

The agent is the reasoning layer. It:
- Maintains conversation context and memory
- Decides what to say and when to say it
- Executes tools (Sendblue CLI for outbound)
- Receives messages via Telegram (or other inbound channel)

Each agent instance is independent. They don't share:
- Memory or conversation history
- Tool definitions or capabilities
- Credentials or API keys
- Runtime environment

**Communication between agents is purely message-based.**

### Sendblue

Sendblue provides phone numbers that can send and receive SMS and iMessage. Each account is completely separate.

Key Sendblue concepts:
- **Phone Number** — The identifier agents use to address each other
- **Contact** — A phone number that has been verified to receive messages
- **Outbound** — Sending via `sendblue send <number> <message>`
- **Inbound** — Receiving requires a relay (Telegram webhook is the tested approach)

**Important:** Sendblue's outbound CLI works independently of inbound. An agent can send without receiving, and vice versa.

### Telegram Bot Relay (Inbound SMS)

Sendblue doesn't natively push inbound SMS to OpenClaw. The tested relay:

```
Incoming SMS → Sendblue API → Telegram Bot Webhook → OpenClaw → Agent
```

In practice:
1. When SMS arrives at Agent B's Sendblue number, Sendblue calls a webhook
2. The webhook forwards the message to a Telegram bot (controlled by Agent B's OpenClaw)
3. OpenClaw reads the Telegram message
4. The agent processes it and responds

This feels like: `sms → telegram → openclaw → agent`

The reverse (outbound):
```
Agent → OpenClaw → Sendblue CLI → Sendblue API → SMS → Other Agent
```

### The Complete Round Trip

```
Agent A wants to coordinate with Agent B:

[OUTBOUND]
1. Agent A composes message
2. Agent A calls:  sendblue send <B_number> <message>
3. Sendblue API receives the request
4. Sendblue sends SMS from A_number to B_number

[INBOUND (Relay)]
5. SMS arrives at B_number
6. Sendblue forwards to Telegram webhook
7. Telegram bot receives the message
8. OpenClaw polls Telegram (or receives via webhook)
9. Agent B sees the message and processes it

[OUTBOUND]
10. Agent B composes response
11. Agent B calls: sendblue send <A_number> <response>
12. Sendblue delivers SMS from B_number to A_number

[INBOUND (Relay)]
13. SMS arrives at A_number
14. Relayed via Telegram to OpenClaw
15. Agent A receives the response
```

Total latency is dominated by relay efficiency — typically 2-10 seconds end to end.

---

## Number Assignment and Discovery

In the current setup, phone numbers are shared out-of-band:
- Agent A's human tells Agent B's human "my agent's number is X"
- They coordinate via a trusted channel (in this case, the humans texted each other directly)

**This is intentionally manual.** There's no directory service, no agent discovery protocol, no automated handshake. The humans act as the trust layer — Tyler texted Robert directly to exchange the numbers.

Future improvements could include:
- Encrypted number exchange via a shared secret
- Agent discovery protocol via a known rendezvous point
- Verified agent credentials published to a registry

For now, human-mediated number exchange is the simplest trust model.

---

## Contact Verification Flow

Sendblue enforces bidirectional verification:

```
Agent A                    Sendblue                     Agent B
   │                           │                            │
   │  add-contact B_number      │                            │
   │──────────────────────────►│                            │
   │                           │                            │
   │                    [status: pending]                   │
   │                           │                            │
   │                           │    ←── B texts A_number ─── │
   │                           │                            │
   │                           │  verify contact            │
   │                           │──────────────────────────► │
   │                           │                            │
   │  [status: verified]       │                            │
   │◄──────────────────────────│                            │
   │                           │                            │
   │  Now A can send to B       │                            │
```

This prevents spam and unauthorized inbound — an agent can't receive messages from a number that hasn't intentionally added and verified them.

---

## Why Not Just Use the Sendblue API Directly?

Sendblue does have an API for both sending and receiving. So why use Telegram as a relay?

**The inbound SMS problem:**

Sendblue's inbound SMS delivery requires a persistent webhook endpoint. For receiving SMS at a phone number, you need:
1. A publicly accessible URL (not behind NAT, no firewall blocking port 80/443)
2. TLS certificate (or SMS providers won't deliver)
3. A process that can receive the webhook and route it to OpenClaw

For most home lab setups (including the one this project uses), the OpenClaw host is on a residential connection without a public IP. The Telegram bot workaround solves this by using Telegram as the publicly accessible relay — OpenClaw polls Telegram instead of receiving webhooks directly.

**Alternatives considered:**
- **ngrok / Cloudflare Tunnel** — Would work, but adds another dependency
- **Direct webhook to OpenClaw** — Requires public IP, TLS, always-on host
- **Email relay** — Works but higher latency, less reliable

Telegram bot is the simplest approach for most setups.

---

## Message Format and Limits

**SMS constraints:**
- 160 characters per segment (single SMS)
- Longer messages are concatenated (depends on carrier)
- Unicode and emojis supported but reduce effective length

**For agent communication, keep messages short and structured:**
```
[OUTBOUND]
Agent A: "Status check: are you receiving? Respond YES/NO."
Agent B: "YES. Ready to coordinate."
Agent A: "Good. Starting task delegation..."
[Full task description]
```

The agent should break complex coordination into discrete messages rather than sending a single massive SMS.

---

## Failure Modes

### Sendblue API Failure
- Outbound fails → agent retries with exponential backoff
- After N retries, agent reports failure to its human

### Telegram Relay Failure
- Webhook missed → message lost (no retry)
- Mitigation: Telegram stores recent messages; agent can poll to catch missed messages

### Contact Not Verified
- Agent tries to send → "Send failed: contact not verified"
- Agent notifies its human to complete verification

### Circular Messages (Agent Chat Loop)
- Two agents could theoretically enter a loop (A texts B, B responds, A responds...)
- Mitigation: agents should track conversation state and know when to stop
- For now, this is a human-governed problem — humans monitor and intervene if needed

---

## Scalability Considerations

This architecture doesn't scale to many agents easily — each new agent pair requires:
- Exchanging phone numbers (manual or via directory)
- Mutual contact verification
- Per-pair Sendblue account management

For a larger multi-agent system, you'd want:
- A central message broker or agent registry
- Structured message protocols (JSON payloads, not freeform SMS)
- Authentication and authorization for agent-to-agent requests

But for **two agents talking to each other**, this setup is lightweight, independent, and requires zero shared infrastructure.

---

## Future Improvements

1. **WebSocket-based relay** — Replace Telegram with a persistent WebSocket connection to Sendblue
2. **Structured message protocol** — JSON payloads with type hints, request IDs, and response routing
3. **Agent discovery** — A shared registry where agents can look up each other's numbers
4. **End-to-end encryption** — Encrypt message content so only the receiving agent can read it
5. **Delivery receipts** — Know when a message was delivered vs. failed
6. **Multi-agent groups** — Like a group chat, but for agents
