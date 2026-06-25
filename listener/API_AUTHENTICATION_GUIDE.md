# NotifyChain — API Authentication Guide

This guide details the authentication and authorization mechanisms used by the NotifyChain listener service, including REST API key authentication, rate limiting tiering, template write auditing, and webhook signature verification.

---

## 1. Authentication Flows

NotifyChain uses two different authentication methods depending on the communication flow:

### REST API Calls (Incoming to Listener)
For clients querying the events feed, registering scheduled notifications, or managing templates, requests are authenticated using HTTP headers.

```
Client Request
   │
   ├── X-API-Key: <key>  ───┐
   │                        ├──► Resolve Actor & Rate Limit Tier ──► Proceed
   ├── Bearer <token>    ───┘
   │
   └── No Header         ──────► Fallback to IP‑based Rate Limit Tier
```

### Webhook Deliveries (Incoming from Webhook Source)
For webhooks delivered to the listener's `/api/webhooks` endpoint, authentication uses HMAC‑SHA256 signatures of the raw request payload to ensure sender authenticity and integrity.

```
Webhook Sender (e.g. Stripe, Stellar Indexer)
   │
   ├── Computes HMAC-SHA256 of raw body using shared secret
   └── Sends POST /api/webhooks with:
        - X-Webhook-Signature: sha256=<hex_digest>
        - X-Webhook-Key-Id: <signing_key_id>
```

---

## 2. Configuration & Key Management

API Keys, Bearer Tokens, and Webhook Secrets are configured on the listener service via environment variables in the `.env` file.

### 1. REST API Keys & Custom Rate Limits
Generic users are rate-limited based on their IP address. To grant elevated rate limits or identify individual clients, configure overrides in `RATE_LIMIT_CLIENT_OVERRIDES` as a JSON object:

```bash
# Define clients, their maximum requests, and sliding window size (ms)
RATE_LIMIT_CLIENT_OVERRIDES='{"admin-key-123":{"maxRequests":100,"windowMs":60000},"editor-token":{"maxRequests":500,"windowMs":60000}}'
```

### 2. Webhook Verification Keys
Register Webhook signing secrets in `WEBHOOK_SECRETS` as a JSON array of objects:

```bash
# Register unique signing keys and their secrets
WEBHOOK_SECRETS='[{"id":"key-prod-1","secret":"whsec_very_long_production_secret_key_12345"},{"id":"key-dev-1","secret":"whsec_development_secret"}]'
```

---

## 3. Authorization Requirements & Scopes

### Rate Limiting Access
*   **IP-Based (Unauthenticated)**: Standard rate limiting (configured via `RATE_LIMIT_MAX_REQUESTS` and `RATE_LIMIT_WINDOW_MS`).
*   **Token-Based (Authenticated)**: Elevates the client to their specific rate limiting parameters as defined in the client overrides configuration.

### Write Operations & Audit Trails
Mutating routes—specifically templates management (such as `PUT /api/templates/:id`)—require identification.
*   The actor identity is parsed from either `X-API-Key` or `Authorization: Bearer <token>`.
*   If no identity header is found, the mutation is rejected with `400 Bad Request` (`TemplateValidationError: Actor is required for template updates`).
*   Successful updates log the actor (e.g. `api-key:admin-key-123` or `bearer:editor-token`) immutably in the `notification_template_audit_log` table.

---

## 4. Request Examples

### REST API Authentication

#### Example 1: Authenticating with X-API-Key
```bash
curl -X GET http://localhost:8787/api/events \
  -H "X-API-Key: admin-key-123"
```

#### Example 2: Authenticating with Bearer Token
```bash
curl -X PUT http://localhost:8787/api/templates/tmpl-001 \
  -H "Authorization: Bearer editor-token" \
  -H "Content-Type: application/json" \
  -d '{"name":"Updated Welcome Email","body":"Hello {{username}}, welcome to NotifyChain!"}'
```

### Webhook Signature Verification

#### Signature Generation Script (Node.js)
The sender signs the raw body of the webhook using HMAC-SHA256:

```javascript
const crypto = require('crypto');

const secret = 'whsec_very_long_production_secret_key_12345';
const payload = JSON.stringify({ event: 'TaskCreated', contractAddress: 'CCEMX6...' });

const signature = 'sha256=' + crypto
  .createHmac('sha256', secret)
  .update(payload, 'utf8')
  .digest('hex');

console.log('X-Webhook-Signature:', signature);
console.log('X-Webhook-Key-Id:', 'key-prod-1');
```

#### Delivering the Webhook (curl)
```bash
curl -X POST http://localhost:8787/api/webhooks \
  -H "Content-Type: application/json" \
  -H "X-Webhook-Signature: sha256=a88e630fb5e88d..." \
  -H "X-Webhook-Key-Id: key-prod-1" \
  -d '{"event": "TaskCreated", "contractAddress": "CCEMX6..."}'
```

---

## 5. Troubleshooting & Error Responses

### `401 Unauthorized` (Webhooks)
Indicates that webhook signature verification failed. Common causes:

| Error Payload | Cause | Resolution |
| :--- | :--- | :--- |
| `{"error":"Missing signature header"}` | `X-Webhook-Signature` header is absent | Include the signature header in the request. |
| `{"error":"Missing key-id header"}` | `X-Webhook-Key-Id` header is absent | Include the key ID used to sign the request. |
| `{"error":"Unknown key-id"}` | Key ID is not registered in the `WEBHOOK_SECRETS` config | Check the `.env` settings to verify the key ID matches what the server expected. |
| `{"error":"Invalid signature"}` | Computed signature did not match the header signature | Verify the secret is correct. Ensure you sign the **raw string/binary body**, and that no formatting transformations occur in transit. |

> [!WARNING]
> **Discrepancy Note**: Older documentation referenced the headers as `X-Signature` and `X-Key-Id`. The actual listener codebase expects `X-Webhook-Signature` and `X-Webhook-Key-Id`. Using the old headers will result in a `401 Unauthorized` response.

### `429 Too Many Requests`
Your client exceeded its request limit.
*   Check the response headers `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `Retry-After`.
*   To resolve this, verify your credentials are being passed correctly (so you are not being rate limited as a generic IP), or ask your system administrator to increase your client limits in `RATE_LIMIT_CLIENT_OVERRIDES`.
