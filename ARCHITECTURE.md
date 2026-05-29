# LexCore MVP — System Architecture

> trace_id: GROK-20260529-UKR-LEX-MVP-DECISION
> sync_state: SYNCING
> Last updated: 2026-05-29 | Bakhmach, UA

---

## Parallel Integration Architecture

```
+------------------+     Webhook      +------------------+
|    ClickUp       |----------------->|      Make        |
| (Event Source)   |<-----------------|  (Orchestrator)  |
+------------------+   Status Update  +------------------+
        |                                      |
        | custom fields:                       | Scenarios:
        | external_id                          | ingest
        | source_system                        | normalize
        | sync_state                           | execute
        | trace_id                             | notify
        | poj_trace_hash                       |
        | pqc_compliance_level                 |
        v                                      v
+------------------+              +------------------+
|  Backend APIs    |              | Document         |
|  (Biz Logic)     |              | Constructor      |
|  - Validation    |              | (KUPA AL)        |
|  - Audit Trail   |              | - ClauseBank     |
|  - Case Manager  |              | - 5 Templates    |
+------------------+              +------------------+
        |                                      |
        v                                      v
+------------------+              +------------------+
|  Cloudflare      |              |   PoJ Stack      |
|  (Edge Security) |              |  - RFC3161 TSA   |
|  - WAF           |              |  - Hash Chain    |
|  - mTLS          |              |  - Polygon Anchor|
|  - Rate Limiting |              +------------------+
|  - Logging       |
+------------------+
```

---

## Component Specifications

### 1. ClickUp — Event Source & Contract Store

**Role:** Central source of truth for task states and custom field contracts.

**Custom Fields:**

```yaml
external_id:
  type: text
  format: "LEX-MVP-{YYYYMMDD}-{seq}"
  example: "LEX-MVP-20260529-001"

source_system:
  type: dropdown
  values: [Grok, Make, Manual, API, LexCore]

sync_state:
  type: dropdown
  values: [PENDING, SYNCING, SUCCESS, FAILED, ARCHIVED]
  default: PENDING

trace_id:
  type: text
  format: "GROK-{YYYYMMDD}-{PROJECT}-{ACTION}"

poj_trace_hash:
  type: text
  format: "sha256:{hex64}"

pqc_compliance_level:
  type: dropdown
  values: [NONE, BASIC, ML-KEM-768, ML-DSA, FULL]
  default: NONE
```

**Sync State Machine:**
```
PENDING --> SYNCING --> SUCCESS
                   --> FAILED --> PENDING (retry)
SUCCESS --> ARCHIVED
```

---

### 2. Make — Orchestrator

**Active Scenario:** `LexCore_Ukraine_Strat_Integration_v0.2`

**Pipeline Stages:**

```
[INGEST]
  Input: PDF tables / Google Sheets / Webhook
  Action: Parse rows, extract fields
  Output: Normalized JSON payload

[NORMALIZE]
  Input: Raw parsed data
  Action: Validate schema, map to ClickUp fields
  Output: ClickUp task payload

[EXECUTE]
  Input: Normalized payload
  Actions:
    - Create/update ClickUp task
    - Trigger Document Constructor
    - Call PoJ Stack (hash + timestamp)
    - Update sync_state = SYNCING
  Output: Task ID + poj_trace_hash

[NOTIFY]
  Input: Execution result
  Actions:
    - Slack notification
    - Email summary
    - Update sync_state = SUCCESS / FAILED
  Output: Confirmation log
```

**Error Handling:**
- On FAILED: retry x3 with exponential backoff
- After 3 retries: set sync_state = FAILED, notify admin
- All errors logged to immutable audit trail

---

### 3. KUPA AL + Document Constructor

**AI Pipeline:**
```
User Request / Case Trigger
  -> KUPA AL (Prompt Manager)
  -> AI Draft Generation
  -> ClauseBank lookup (PQC clauses, MFO Shield clauses)
  -> Draft assembly
  -> human-in-the-loop review (Lawyer / Senior Partner)
  -> Final document
  -> PoJ anchoring
```

**PQC Integration:**
- Default signing: ML-DSA (CRYSTALS-Dilithium, NIST FIPS 204)
- Default encryption: ML-KEM-768 (CRYSTALS-Kyber, NIST FIPS 203)
- Applied to ALL new document templates

---

### 4. PoJ Stack — Proof of Justice

**Immutability Layers:**

```
Layer 1: Hash Generation
  Algorithm: SHA-256
  Input: document_content + timestamp + case_id
  Output: sha256_hash

Layer 2: RFC3161 Timestamping
  TSA: trusted timestamp authority
  Input: sha256_hash
  Output: rfc3161_token (DER encoded)
  Stored: ClickUp poj_trace_hash field

Layer 3: Polygon Blockchain Anchoring
  Network: Polygon PoS (mainnet)
  Input: sha256_hash
  Method: calldata in tx
  Output: polygon_tx_hash
  Stored: ClickUp trace_id field

Layer 4: Immutable Log Entry
  Type: append-only hash-chain
  Fields: timestamp, hash, rfc3161_token, polygon_tx, actor, action
  Integrity: each entry hashes previous entry (chain)
```

**Verification Flow:**
```
Verifier receives: document + poj_trace_hash + polygon_tx_hash
  1. Compute SHA-256 of document
  2. Verify RFC3161 token against hash
  3. Check Polygon tx calldata matches hash
  4. Cross-reference immutable log
  Result: VERIFIED / TAMPERED
```

---

### 5. Cloudflare — Edge Security

**Security Layers:**

```yaml
mTLS:
  enforced: true
  scope: all API endpoints
  cert_rotation: 90 days

WAF:
  ruleset: OWASP Core Rule Set
  custom_rules:
    - block SQL injection
    - block path traversal
    - rate limit legal doc generation

Rate Limiting:
  document_generation: 100 req/min per IP
  api_calls: 1000 req/min per token
  webhook_ingest: 500 req/min

Logging:
  destination: Cloudflare Logpush -> S3
  retention: 365 days
  format: JSON structured
  fields: [timestamp, ip, path, status, trace_id]
```

---

## Make Automation — Table Parsing Script

**Input Sources:**
- Google Sheets (via Sheets API)
- PDF tables (via PDF parser module)
- CSV uploads

**Field Mapping:**

```json
{
  "source_row": {
    "task_name": "-> ClickUp task title",
    "priority": "-> ClickUp priority",
    "assignee": "-> ClickUp assignee",
    "due_date": "-> ClickUp due_date",
    "category": "-> ClickUp tag",
    "external_ref": "-> custom: external_id",
    "system": "-> custom: source_system"
  }
}
```

**Auto-generated on task creation:**
- `sync_state` = PENDING
- `trace_id` = auto-generated
- `pqc_compliance_level` = NONE (upgradeable)

---

## Security Architecture Summary

| Layer | Technology | Purpose |
|-------|-----------|--------|
| Transport | mTLS (Cloudflare) | Mutual auth all API calls |
| Application | WAF (Cloudflare) | Block injection attacks |
| Document Signing | ML-DSA (FIPS 204) | PQC digital signatures |
| Document Encryption | ML-KEM-768 (FIPS 203) | PQC key encapsulation |
| Timestamping | RFC3161 TSA | Trusted time proof |
| Blockchain | Polygon PoS | Public immutable anchor |
| Audit | Hash-chain log | Tamper-evident trail |

---

*LexCore MVP Architecture v0.1 | 2026-05-29 | Bakhmach, UA*
