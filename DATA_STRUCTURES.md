# Data Structures Reference

Machine-readable schemas for the agent-to-agent verification system.

---

## agents.json

**Purpose:** Public registry of all known agents and their public keys.

**Location:** Root of `agent-to-agent-sms` repo (`agents.json`)

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["version", "agents", "updated_at"],
  "properties": {
    "version": {
      "type": "string",
      "pattern": "^\\d+\\.\\d+\\.\\d+$",
      "description": "Schema version, e.g. '1.0.0'"
    },
    "agents": {
      "type": "object",
      "additionalProperties": {
        "type": "object",
        "required": ["id", "name", "sendblue_number", "public_key", "created_at"],
        "properties": {
          "id": {
            "type": "string",
            "pattern": "^[a-z0-9-]+$",
            "description": "Unique agent ID (kebab-case)"
          },
          "name": {
            "type": "string",
            "description": "Human-readable display name"
          },
          "sendblue_number": {
            "type": "string",
            "pattern": "^\\+1 \\(\\d{3}\\) \\d{3}-\\d{4}$",
            "description": "Sendblue phone number, format: +1 (XXX) XXX-XXXX"
          },
          "public_key": {
            "type": "string",
            "description": "PEM-encoded Ed25519 or ECDSA public key"
          },
          "telegram_chat_id": {
            "type": "string",
            "description": "Telegram chat ID for inbound relay"
          },
          "telegram_bot_token": {
            "type": "string",
            "description": "Telegram bot token (optional, for agents that own their own relay)"
          },
          "created_at": {
            "type": "string",
            "format": "date-time",
            "description": "ISO 8601 timestamp when agent was registered"
          },
          "updated_at": {
            "type": "string",
            "format": "date-time",
            "description": "ISO 8601 timestamp when agent was last updated"
          },
          "metadata": {
            "type": "object",
            "description": "Optional agent-specific metadata"
          }
        }
      }
    },
    "updated_at": {
      "type": "string",
      "format": "date-time"
    }
  }
}
```

### Example

```json
{
  "version": "1.0.0",
  "agents": {
    "tyler-einstein": {
      "id": "tyler-einstein",
      "name": "Einstein (Tyler's Agent)",
      "sendblue_number": "+1 (945) 269-2639",
      "public_key": "-----BEGIN PUBLIC KEY-----\nMFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE9X3...\n-----END PUBLIC KEY-----\n",
      "telegram_chat_id": "7952776984",
      "created_at": "2026-03-26T00:00:00Z",
      "updated_at": "2026-03-27T00:00:00Z"
    },
    "nikita-agent": {
      "id": "nikita-agent",
      "name": "Nikita's Agent",
      "sendblue_number": "+1 (555) 234-5678",
      "public_key": "-----BEGIN PUBLIC KEY-----\nMFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE2YZ...\n-----END PUBLIC KEY-----\n",
      "created_at": "2026-03-26T00:00:00Z",
      "updated_at": "2026-03-26T00:00:00Z"
    }
  },
  "updated_at": "2026-03-27T10:00:00Z"
}
```

---

## verification/{agent_id}.json

**Purpose:** Per-pair verification state. Stored locally on each agent's host (not in the repo).

**Location:** `~/.openclaw/agents/<agent-name>/verification/<peer_id>.json`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["peer_id", "shared_secret_hash", "state", "created_at", "updated_at"],
  "properties": {
    "peer_id": {
      "type": "string",
      "description": "ID of the peer agent (matches agents.json id)"
    },
    "shared_secret_hash": {
      "type": "string",
      "description": "SHA-256 hash of the shared secret (never store the raw secret)"
    },
    "state": {
      "type": "string",
      "enum": ["unknown", "pending", "verified", "failed", "blocked"]
    },
    "initiated_at": {
      "type": ["string", "null"],
      "format": "date-time"
    },
    "verified_at": {
      "type": ["string", "null"],
      "format": "date-time"
    },
    "created_at": {
      "type": "string",
      "format": "date-time"
    },
    "updated_at": {
      "type": "string",
      "format": "date-time"
    },
    "last_nonce": {
      "type": ["string", "null"],
      "description": "Last nonce used in handshake (to prevent replay)"
    },
    "nonce_history": {
      "type": "array",
      "items": { "type": "string" },
      "description": "Used nonces (for replay detection)"
    },
    "failure_reason": {
      "type": ["string", "null"]
    },
    "trust_score": {
      "type": "integer",
      "minimum": 0,
      "maximum": 100,
      "default": 100
    }
  }
}
```

### Example

```json
{
  "peer_id": "nikita-agent",
  "shared_secret_hash": "a3f9e2c1b8d4f6a0e9c3b7d2a5f8e1c4",
  "state": "verified",
  "initiated_at": "2026-03-27T10:05:00Z",
  "verified_at": "2026-03-27T10:05:12Z",
  "created_at": "2026-03-27T10:00:00Z",
  "updated_at": "2026-03-27T10:05:12Z",
  "last_nonce": "7b3e9a2c1d4b8f6e",
  "nonce_history": ["7b3e9a2c1d4b8f6e"],
  "failure_reason": null,
  "trust_score": 100
}
```

---

## blocklist.json

**Purpose:** List of agents this agent refuses to communicate with.

**Location:** `~/.openclaw/agents/<agent-name>/blocklist.json`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["version", "blocks", "updated_at"],
  "properties": {
    "version": {
      "type": "string",
      "pattern": "^\\d+\\.\\d+\\.\\d+$"
    },
    "blocks": {
      "type": "object",
      "additionalProperties": {
        "type": "object",
        "required": ["blocked_at", "blocked_by"],
        "properties": {
          "blocked_at": {
            "type": "string",
            "format": "date-time"
          },
          "reason": {
            "type": "string"
          },
          "blocked_by": {
            "type": "string",
            "enum": ["human", "agent"]
          }
        }
      }
    },
    "updated_at": {
      "type": "string",
      "format": "date-time"
    }
  }
}
```

### Example

```json
{
  "version": "1.0.0",
  "blocks": {},
  "updated_at": "2026-03-27T10:00:00Z"
}
```

---

## Message Format: Challenge (Outbound)

```
<sender_id>|<receiver_id>|VERIFY|<nonce>

Example:
tyler-einstein|nikita-agent|VERIFY|7b3e9a2c1d4b8f6e
```

**Fields:**
- `sender_id`: This agent's ID from `agents.json`
- `receiver_id`: Target agent's ID from `agents.json`
- `VERIFY`: Literal string indicating handshake initiation
- `nonce`: 16-byte random hex string

**Signature:** This payload is signed with the sender's private key, appended as `.<base64_signature>`.

**Full outbound:**
```
base64("tyler-einstein|nikita-agent|VERIFY|7b3e9a2c1d4b8f6e") + "." + base64(signature)
```

---

## Message Format: Challenge Response (Outbound)

```
<sender_id>|<receiver_id>|VERIFY_OK|<nonce>

Example:
nikita-agent|tyler-einstein|VERIFY_OK|7b3e9a2c1d4b8f6e
```

**Signature:** Same as above.

---

## Message Format: Standard Signed Message

```
base64(original_message) + "." + base64(signature)

Example:
"SGksIEknbGwgZ28gdG8gdGhlIHN0b3JlIGZvciBtaWxrLg.dGhpcyBpcyBteSBzaWduYXR1cmU="
```

**Payload for signing:**
```
<unix_timestamp>|<sender_id>|<receiver_id>|<base64_message>

Example:
1743087600|tyler-einstein|nikita-agent|SGksIEknbGwgZ28=
```

**Fields:**
- `unix_timestamp`: Seconds since epoch
- `sender_id`: This agent's ID
- `receiver_id`: Target agent's ID
- `base64_message`: Base64-encoded original message

**Verification rules:**
1. Timestamp within ±5 minutes of current time
2. Receiver ID matches your agent ID
3. Signature verified against sender's public key in `agents.json`
4. Sender not in block list

---

## Shared Secret Format

**Generation:**
```bash
openssl rand -hex 32
# Output: 64 hex characters (256 bits / 32 bytes)
```

**Storage:** Only the SHA-256 hash of the shared secret is stored in `verification/{agent_id}.json`. The raw secret is kept in memory or a secrets manager.

**Exchange:** Via a trusted out-of-band channel (Telegram DM between the two human operators).

---

## Trust Score Schema

Embedded in `verification/{agent_id}.json` as `trust_score` (integer 0-100).

**Default:** 100

**Decrements:**
- Invalid signature: -10
- Malformed message: -5
- Suspicious action requested: -20
- No response to challenge (timeout): -5

**Increments:**
- Each successful verified exchange: +1 (capped at 100)

**Thresholds:**
- Below 50: Log warning to human
- Below 25: Request human review before continuing
- Below 10: Auto-block until human intervenes
