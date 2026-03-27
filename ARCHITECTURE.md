# Architecture Deep Dive

## System Overview

The agent-to-agent SMS system has four layers:

```
┌─────────────────────────────────────────────────────────────┐
│                     MESSAGE LAYER                           │
│                                                             │
│   Agent A ◄────── SMS ──────► Agent B                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    CHANNEL LAYER                            │
│                                                             │
│   Sendblue CLI ────────────────────────────────────────►   │
│   (outbound)            SMS Network            Receive SMS  │
│                                                             │
│   Inbound Relay ◄────── Webhook ──────── Sendblue API       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   SENDblue LAYER                           │
│                                                             │
│   Sendblue Account A           Sendblue Account B          │
│   Phone Number A               Phone Number B              │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    PHYSICAL LAYER                          │
│                                                             │
│   Cellular Network ◄─────────────────► Cellular Network     │
└─────────────────────────────────────────────────────────────┘
```

---

## Component Roles

### Agent

The reasoning layer. An agent:
- Maintains conversation context and memory
- Decides what to say and when
- Executes tools to send SMS
- Receives messages via an inbound relay

Each agent instance is independent. They don't share memory, credentials, runtime environment, or tool definitions. Communication is purely message-based.

### Sendblue

SMS/iMessage gateway. Each account is separate.

Key concepts:
- **Phone Number** — The agent's address for SMS
- **Contact** — A verified number the agent can send to
- **Outbound** — `POST /api/send-message` via CLI or API
- **Inbound** — Sendblue calls a webhook; an inbound relay delivers to the agent

### Inbound Relay

Sendblue can't push directly to agent infrastructure (agents typically run behind NAT or on residential connections without a public IP). An inbound relay bridges that gap.

**How it works:**
1. SMS arrives at the agent's Sendblue number
2. Sendblue calls a webhook URL
3. The relay (Telegram bot, Cloudflare Tunnel, etc.) receives the call
4. The agent polls or receives the relay input
5. The agent processes and responds

**Common relay options:**

| Relay | Pros | Cons |
|-------|------|------|
| Telegram bot | Free, globally accessible | Requires Telegram account |
| Cloudflare Tunnel | Permanent URL, free tier | More complex setup |
| ngrok | Quick to set up | Session-based URLs expire |

### Verification API

Breaks the circular contact verification problem. Both agents call `POST /api/v2/agent/verify-start` and `POST /api/v2/agent/verify-complete` with shared secret hashes. Sendblue validates and marks contacts verified — no SMS round-trip needed.

See [VERIFICATION_API.md](./VERIFICATION_API.md) for the full specification.

---

## The Complete Round Trip

```
Agent A wants to coordinate with Agent B:

[OUTBOUND]
1. Agent A composes a message
2. Agent A calls: sendblue send <B_number> <message>
3. Sendblue delivers SMS from A_number to B_number

[INBOUND]
4. SMS arrives at B_number
5. Sendblue calls webhook → relay → Agent B
6. Agent B processes the message

[OUTBOUND]
7. Agent B composes response
8. Agent B calls: sendblue send <A_number> <response>
9. Sendblue delivers SMS from B_number to A_number

[INBOUND]
10. SMS arrives at A_number
11. Relayed to Agent A
```

Typical latency: 3-15 seconds depending on relay and polling interval.

---

## Number Exchange and Discovery

In the current system, phone numbers are shared out-of-band:
- Agent A's operator shares the number with Agent B's operator via any trusted channel
- Agents exchange their Sendblue numbers before attempting verification

This is intentionally manual. There's no agent directory or discovery protocol built in. The operators act as the trust layer.

---

## Contact Verification Flow

Sendblue requires bidirectional contact verification before SMS delivery. The old approach required an SMS round-trip — which creates a circular dependency for two agents.

**The new approach (via Verification API):**

```
Agent A                                          Sendblue                    Agent B
   │                                                  │                           │
   │  POST /agent/verify-start                         │                           │
   │  { number, api_key, target: B_number }             │                           │
   │────────────────────────────────────────────────►│                           │
   │                                                  │                           │
   │  ←── { verification_id, secret_hash }             │                           │
   │                                                  │                           │
   │  [A shares secret_hash with B out-of-band]       │                           │
   │                                                  │                           │
   │                                                  │  POST /agent/verify-start │
   │                                                  │◄─────────────────────────│
   │                                                  │                           │
   │                                                  │  ←── { verification_id,   │
   │                                                  │        secret_hash }      │
   │                                                  │                           │
   │                                                  │  [B shares secret_hash    │
   │                                                  │   with A out-of-band]     │
   │                                                  │                           │
   │  POST /agent/verify-complete                      │                           │
   │  { verification_id, their_secret_hash }            │                           │
   │────────────────────────────────────────────────►│                           │
   │                                                  │                           │
   │  ✓ Contact verified                               │                           │
   │                                                  │  POST /agent/verify-complete
   │                                                  │◄─────────────────────────│
   │                                                  │                           │
   │                                                  │  ✓ Contact verified       │
   │                                                  │                           │
   │  Now A can send to B                              │  Now B can send to A       │
```

---

## Why Not Direct Webhook to the Agent?

Sendblue's inbound SMS delivery requires a publicly accessible HTTPS endpoint. Most agent deployments:
- Run on a local machine or home server without a public IP
- Behind NAT or a residential connection
- Without a TLS certificate

An inbound relay solves this by using a publicly accessible service (Telegram, Cloudflare Tunnel) as an intermediary. The relay doesn't have the circular verification problem because it's not sending SMS — it's just delivering the text content to wherever the agent is polling.

---

## Message Format

**SMS constraints:**
- 160 characters per segment
- Longer messages are concatenated (carrier-dependent)
- Unicode supported but reduces effective length

**For agent coordination, keep messages short and structured.** Break complex coordination into discrete messages rather than sending one massive SMS.

Agents may adopt a simple structured format:
```
task:delegate {"task_id": "123", "description": "process invoice"}
status:query
response:task_complete {"task_id": "123", "result": "ok"}
```

---

## Failure Modes

| Failure | Cause | Mitigation |
|---------|-------|------------|
| Send fails: contact not verified | Verification not completed | Call verification API |
| Send fails: 400 | Invalid number or API issue | Check E.164 format |
| No inbound received | Relay misconfigured | Check webhook URL, relay polling |
| Message lost | Relay webhook missed | Most relays store recent messages; poll to catch up |
| Circular loop | Two agents loop | Agents track state; operators monitor |

---

## Scalability

This architecture works well for **two agents talking to each other**. It doesn't require shared infrastructure.

Scaling to more agents requires:
- A directory or registry for agent phone numbers
- Per-pair contact verification
- Per-pair Sendblue account management (or a multi-number account)

For larger systems, consider a central message broker, structured message protocols, and a proper agent registry.
