# Agent-to-Agent Verification API

> Sendblue backend endpoint that breaks the circular contact-verification dependency for agent-to-agent SMS.

---

## Problem

Sendblue requires bidirectional contact verification before agents can SMS each other:

```
Agent A adds Agent B's number → pending
Agent B must text Agent A's number to verify
```

For **human-to-agent** this works fine. For **agent-to-agent**, neither agent can text first without already being verified. Circular dependency.

## Solution

A backend verification endpoint that validates agent identity through a shared secret — no SMS round-trip required.

---

## Architecture

```
┌─────────────┐                           ┌─────────────┐
│  Agent A    │                           │  Agent B    │
│  (OpenClaw) │                           │  (OpenClaw) │
└──────┬──────┘                           └──────┬──────┘
       │                                         │
       │  1. /agent/verify-start                 │
       │────────────────────────────────────────►│
       │                                         │
       │                                         │  2. /agent/verify-start
       │◄────────────────────────────────────────│
       │                                         │
       │  3. Exchange secret hashes              │
       │     (via any trusted channel)           │
       │                                         │
       │  4. /agent/verify-complete              │
       │────────────────────────────────────────►│
       │                                         │
       │  5. /agent/verify-complete              │
       │◄────────────────────────────────────────│
       │                                         │
       │  ✓ Both agents verified                │
```

---

## Endpoints

### POST `/api/v2/agent/verify-start`

Initiates verification with another agent. Both agents call this independently.

**Request:**
```json
{
  "number": "+15551112222",
  "api_key_id": "sb_abc123",
  "api_secret_key": "sk_def456",
  "target_number": "+15553334444"
}
```

**Response (200):**
```json
{
  "status": "ok",
  "verification_id": "vrf_7f3e9a2c",
  "secret_hash": "a3f9e2c1b8d4f6e0a9c3d7e2b5f8a1c4d9e6f3b8a2c5d7e9f1b4a3c8d6e2f9b",
  "expires_at": "2026-03-27T11:05:00Z"
}
```

**What happens:**
- Sendblue records that `+15551112222` initiated verification with `+15553334444`
- A `secret_hash` (SHA-256 of a Sendblue-generated secret) is returned
- Agent must share this `secret_hash` with the other agent via any trusted channel
- Both agents will submit their secret hashes in `verify-complete`

---

### POST `/api/v2/agent/verify-complete`

Completes verification once both agents have each other's secret hashes.

**Request:**
```json
{
  "number": "+15551112222",
  "api_key_id": "sb_abc123",
  "api_secret_key": "sk_def456",
  "verification_id": "vrf_7f3e9a2c",
  "their_secret_hash": "b4c8d2e1f6a3b9c7d5e8f1a3b7c2d6e9f4a8b1c3d5e7f9a2b4c6d8e1f3a7b"
}
```

**Response (200):**
```json
{
  "status": "ok",
  "verified": true,
  "contact": {
    "number": "+15553334444",
    "verified_at": "2026-03-27T10:55:00Z"
  }
}
```

**What happens:**
- Sendblue looks up the pending verification for `verification_id`
- Compares the submitted `their_secret_hash` against the hash stored for the target agent
- If both sides match, marks `+15553334444` as a verified contact for `+15551112222`
- Agent can now send SMS to that number

**Failure (400):**
```json
{
  "status": "error",
  "error": "secret_mismatch",
  "message": "Submitted secret hash does not match target agent's secret"
}
```

---

## Flow: Full Example

### Step 1: Exchange Secret Hashes (Out-of-Band)

The only step requiring coordination between agents. Any trusted channel works:

- Agents could exchange hashes via a shared database both have access to
- A simple approach: each agent writes its hash to a shared file/API endpoint
- For Nikita's Sendblue customers: the Sendblue dashboard could host a temporary shared workspace where two agents enter their hashes

**Simpler approach for shared Sendblue organization:**

Agent A:
```bash
curl -X POST 'https://api.sendblue.co/api/v2/agent/verify-start' \
  -H 'Content-Type: application/json' \
  -d '{
    "number": "+15551112222",
    "api_key_id": "sb_xxx",
    "api_secret_key": "sk_yyy",
    "target_number": "+15553334444"
  }'
# Returns: { "secret_hash": "abc123...", "verification_id": "vrf_xxx" }
```

Agent A shares `abc123` + `vrf_xxx` with Agent B (via shared Sendblue dashboard, Signal, email, etc.)

Agent B does the same:
```bash
curl -X POST 'https://api.sendblue.co/api/v2/agent/verify-start' \
  ...
# Returns: { "secret_hash": "def456...", "verification_id": "vrf_yyy" }
```

Agent B shares `def456` + `vrf_yyy` with Agent A.

### Step 2: Complete Verification

Agent A:
```bash
curl -X POST 'https://api.sendblue.co/api/v2/agent/verify-complete' \
  -d '{
    "number": "+15551112222",
    "api_key_id": "sb_xxx",
    "api_secret_key": "sk_yyy",
    "verification_id": "vrf_xxx",
    "their_secret_hash": "def456"
  }'
# Verified: Agent A can now send to Agent B
```

Agent B:
```bash
curl -X POST 'https://api.sendblue.co/api/v2/agent/verify-complete' \
  -d '{
    "number": "+15553334444",
    "api_key_id": "sb_zzz",
    "api_secret_key": "sk_aaa",
    "verification_id": "vrf_yyy",
    "their_secret_hash": "abc123"
  }'
# Verified: Agent B can now send to Agent A
```

**Note:** Calls can happen in any order or concurrently. Verification is per-direction — Agent A being verified to send to Agent B doesn't automatically make Agent B verified to send to Agent A, though typically both complete it.

---

## Security Considerations

### Shared Secret Hash — Not the Secret Itself

The API only exchanges **hashed** secrets. The raw secret is generated by Sendblue and sent to Agent A; Agent A never transmits it. This prevents interception attacks.

### Secret Expiry

Secrets expire after 15 minutes. If `verify-complete` isn't called in time, the agent must restart with a new `verify-start`.

### API Key Authentication

Both endpoints require the calling agent's API credentials. This ties verification to the Sendblue account — an attacker without valid API credentials can't initiate or complete verification.

### Rate Limiting

`verify-start` is rate-limited to 5 requests per minute per account to prevent enumeration attacks.

---

## Error Codes

| Code | Meaning |
|------|---------|
| `verification_not_found` | `verification_id` is invalid or expired |
| `secret_mismatch` | Submitted hash doesn't match target agent's hash |
| `target_not_found` | `target_number` is not a registered Sendblue number |
| `already_verified` | Contact is already verified |
| `invalid_credentials` | API key or secret is wrong |
| `rate_limited` | Too many requests |

---

## Open Questions for Nikita

1. **Secret exchange:** What's the simplest trusted channel for agents to exchange hashes? A shared Sendblue dashboard UI? A temporary rendezvous endpoint?
2. **Bidirectional by default:** Should completing verification for one direction automatically verify the reverse? (Most agent pairs want mutual verification)
3. **Revocation:** If an agent's secret is compromised, can it call `/verify-revoke` to invalidate and re-verify?
4. **Agent registry:** Should Sendblue maintain a list of registered agent numbers so agents can look each other up by ID rather than phone number?
