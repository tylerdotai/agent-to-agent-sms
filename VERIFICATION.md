# Agent-to-Agent Verification Protocol

> How two agents establish trusted communication without a human in the loop.

---

## The Problem

Sendblue requires bidirectional contact verification before agents can SMS each other:

```
Agent A adds Agent B as contact → status: pending
Agent B must text Agent A to verify → status: verified
```

For **human-to-agent** this works fine. For **agent-to-agent**, neither agent can text first without already being verified. Circular dependency.

The solution: a **shared secret handshake** that proves both agents are authorized, without requiring an SMS round-trip.

---

## The Protocol: Shared Secret + Backend Verify

1. Each agent has a **Sendblue API key** tied to their phone number
2. Both agents call **`/agent/verify-start`** — Sendblue generates a per-agent secret and returns its hash
3. Agents **exchange hashes** via any trusted channel (email, Signal, shared dashboard, etc.)
4. Both agents call **`/agent/verify-complete`** with the hash they received
5. Sendblue validates both hashes match and marks contacts verified

---

## Step-by-Step

### Step 1: Generate Your Secret Hash

```
POST /api/v2/agent/verify-start
{
  "number": "+15551112222",
  "api_key_id": "sb_xxx",
  "api_secret_key": "sk_yyy",
  "target_number": "+15553334444"
}

→ {
  "verification_id": "vrf_abc123",
  "secret_hash": "a3f9e2c1b8d4f6e0a9c3d7e2b5f8a1c4...",
  "expires_at": "2026-03-27T11:05:00Z"
}
```

### Step 2: Exchange Hashes

Agent A shares `a3f9e2c1...` + `vrf_abc123` with Agent B (any trusted channel).

Agent B does the same, gets their own `secret_hash` and `verification_id`.

### Step 3: Complete Verification

```
POST /api/v2/agent/verify-complete
{
  "number": "+15551112222",
  "api_key_id": "sb_xxx",
  "api_secret_key": "sk_yyy",
  "verification_id": "vrf_abc123",
  "their_secret_hash": "b4c8d2e1f6a3b9c7..."
}

→ {
  "status": "ok",
  "verified": true,
  "contact": { "number": "+15553334444", "verified_at": "..." }
}
```

Agent B does the same with their `verification_id` and Agent A's hash.

**Result:** Both agents can now SMS each other.

---

## Verification States

| State | Meaning |
|-------|---------|
| `unknown` | No record of this agent |
| `pending` | `verify-start` called, waiting for `verify-complete` |
| `verified` | Both sides validated — SMS enabled |

---

## Trust Scoring

Agents may track per-pair trust scores:

```json
{
  "trust_scores": {
    "+15553334444": {
      "score": 100,
      "last_interaction": "2026-03-27T10:55:00Z",
      "message_count": 47,
      "failed_signature_count": 0
    }
  }
}
```

Trust scores decrease when:
- Malformed messages arrive
- Suspicious actions are requested

---

## Threat Model

### What This Solves

- **Circular verification** — no SMS round-trip required
- **Unauthorized contact** — only agents with valid API credentials and a shared secret can establish verification

### What This Doesn't Solve

- ** Compromised API key** — rotate immediately if leaked
- **Man-in-the-middle on the hash exchange** — use a trusted channel for the hash exchange
- **Sendblue internal compromise** — this is a Sendblue infrastructure trust question

---

## File Structure

```
agent-to-agent-sms/
├── README.md
├── ARCHITECTURE.md
├── SETUP.md
├── SECURITY.md
├── VERIFICATION.md          ← High-level protocol
├── VERIFICATION_API.md       ← Concrete API spec (for Sendblue)
└── IMPLEMENTATION.md         ← Agent-side implementation guide
```
