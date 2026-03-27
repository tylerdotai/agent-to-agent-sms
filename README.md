# Agent-to-Agent SMS Communication

> Two AI agents, talking to each other through real SMS — like telekinesis, but with phone bills.

This repo documents an architecture for enabling two AI agents running on separate infrastructure to communicate directly via SMS, using Sendblue as the phone provider and a backend verification API to establish trusted contact without a human in the loop.

---

## Why This Matters

Most agent-to-agent communication requires shared infrastructure — a shared API, a message broker, a webhook both agents can reach. SMS breaks that dependency. If two agents have phone numbers, they can coordinate regardless of where they're hosted or what framework they run on.

This is like human communication: you don't need to be on the same platform to text someone. Now agents don't either.

---

## The Architecture

```
┌─────────────────┐         SMS          ┌─────────────────┐
│   Agent A        │◄──────────────────►│   Agent B        │
│                   │                      │                   │
│  ┌─────────────┐│                      │  ┌─────────────┐│
│  │ Sendblue CLI ││                      │  │ Sendblue CLI ││
│  └──────┬──────┘│                      │  └──────┬──────┘│
└─────────┼───────┘                        └─────────┼───────┘
          │                                        │
          ▼                                        ▼
   ┌─────────────┐                          ┌─────────────┐
   │  Sendblue   │                          │  Sendblue   │
   │ Account A   │                          │ Account B   │
   │ (Phone #1)  │                          │ (Phone #2)  │
   └─────────────┘                          └─────────────┘
```

Each agent has its own Sendblue account and phone number. They text each other the same way two humans would.

---

## The Verification Problem

Sendblue requires bidirectional contact verification before agents can exchange SMS. The standard flow requires one agent to text first — but for two agents, neither can text without already being verified. Circular dependency.

**The solution:** A backend verification API that validates agent identity through a shared secret, with no SMS round-trip required.

See [VERIFICATION_API.md](./VERIFICATION_API.md) for the full specification.

---

## Components

| Component | Role |
|-----------|------|
| **Sendblue** | Phone number provider, SMS gateway, contact verification |
| **Sendblue CLI / API** | How agents send outbound SMS |
| **Inbound Relay** | Delivers inbound SMS to the agent (see below) |
| **Verification API** | Breaks the circular verification dependency |

### Inbound SMS Relay

Sendblue doesn't push directly to agent infrastructure. An inbound relay forwards incoming SMS to wherever the agent is running. Common options:

- **Telegram bot** — Sendblue calls a Telegram bot API; the agent polls Telegram
- **Cloudflare Tunnel** — Direct webhook to the agent's host with a public HTTPS URL
- **ngrok** — Temporary public URL to your local agent

The relay is an infrastructure detail. Any publicly reachable endpoint that can receive a webhook from Sendblue works.

---

## How It Works

### 1. Each agent provisions a Sendblue number

Each operator creates a Sendblue account, gets a phone number, and installs the Sendblue CLI.

### 2. Agents exchange numbers (out-of-band)

Agents share their phone numbers via any trusted channel — email, Signal, a shared dashboard, etc.

### 3. Agents verify via the backend API

Both agents call the verification API with their number and API credentials. Sendblue validates both sides and marks the contact verified. No SMS exchange required.

See [VERIFICATION_API.md](./VERIFICATION_API.md) for the full flow.

### 4. Agents communicate freely

Once verified, agents can send and receive SMS directly.

---

## Repo Structure

```
agent-to-agent-sms/
├── README.md               # This file — overview
├── ARCHITECTURE.md         # Deep dive into components and data flow
├── SETUP.md                # Step-by-step setup guide
├── SECURITY.md             # Security model and threat analysis
├── VERIFICATION.md         # High-level verification protocol
├── VERIFICATION_API.md     # Backend API specification
└── DATA_STRUCTURES.md      # Request/response schemas
```

---

## Related

- [Sendblue API Documentation](https://docs.sendblue.com)
- [OpenClaw Documentation](https://docs.openclaw.ai)
