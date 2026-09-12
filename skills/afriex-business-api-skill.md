---
name: Afriex
description: Use when building payment integrations, managing cross-border transactions, processing customer payouts, handling deposits, or setting up real-time webhook notifications. Agents should reach for this skill when working with payment APIs, customer management, transaction processing, or integrating multi-currency payment rails.
metadata:
    mintlify-proj: afriex
    version: "1.0"
---

# Afriex Business API Skill

## Product Summary

The Afriex Business API enables cross-border payments, customer management, and real-time transaction processing across Africa and global markets. Use it to create customers, set up payment methods (bank accounts, mobile money, SWIFT, crypto), process deposits and withdrawals, and receive webhook notifications. The API supports 15+ payment channels and 50+ currencies.

**Key files and endpoints:**
- Base URLs: `https://sandbox.api.afriex.com` (staging) or `https://api.afriex.com` (production)
- Authentication: `x-api-key` header with API key from dashboard
- Core endpoints: `/api/v1/customer`, `/api/v1/payment-method`, `/api/v1/transaction`, `/api/v1/webhook`
- SDK: `@afriex/sdk` (TypeScript, includes retry logic and webhook verification)
- MCP Server: `https://mcp.afriex.com/mcp` for AI assistant integration

## When to Use

Reach for this skill when:
- Building payment flows: creating customers, adding payment methods, processing transactions
- Handling cross-border payments: converting currencies, routing through multiple payment rails
- Managing deposits and withdrawals: pulling funds from customer accounts or sending payouts
- Setting up real-time notifications: configuring webhooks for transaction status updates
- Testing payment integrations: using sandbox environment with simulated outcomes
- Integrating with AI tools: using MCP server to expose Afriex API to Claude, Cursor, or other MCP clients
- Debugging payment failures: interpreting AFX_* failure codes and retry logic

## Quick Reference

### Authentication & Environments

| Item | Value |
|------|-------|
| Header | `x-api-key: YOUR_API_KEY` |
| Staging Base URL | `https://sandbox.api.afriex.com` |
| Production Base URL | `https://api.afriex.com` |
| API Version Header | `x-api-version: 2026-05-18` (optional, defaults to this) |
| Webhook Signature Header | `x-webhook-signature` (RSA-SHA256, base64) |

### Core Resource IDs

| Resource | ID Field | Used In |
|----------|----------|---------|
| Customer | `customerId` | All customer operations, transactions |
| Payment Method | `paymentMethodId` | Transactions as `sourceId` (DEPOSIT) or `destinationId` (WITHDRAW) |
| Transaction | `transactionId` | Status checks, authorization, webhooks |

### Payment Channels

| Channel | Type | Common Use |
|---------|------|------------|
| BANK_ACCOUNT | WITHDRAW | Payout to local banks (Nigeria GTBank, Kenya KCB) |
| MOBILE_MONEY | DEPOSIT & WITHDRAW | M-Pesa, MTN MoMo, Airtel Money |
| SWIFT | WITHDRAW | International wire transfers to US, Europe |
| UPI | WITHDRAW | India payouts |
| INTERAC | DEPOSIT & WITHDRAW | Canada collections and payouts |
| CRYPTO | DEPOSIT | USDC/USDT deposits |
| VIRTUAL_BANK_ACCOUNT | DEPOSIT | Receive collections (production only) |
| POOL_ACCOUNT | DEPOSIT | Shared collection account (production only) |

### Transaction Types

| Type | Source | Destination | Use Case |
|------|--------|-------------|----------|
| WITHDRAW | Business wallet (implicit) | Payment method (destinationId) | Send money to customer |
| DEPOSIT | Payment method (sourceId) | Business wallet (implicit) | Collect money from customer |
| SWAP | Business wallet | Business wallet | Convert currency within wallet |

### Transaction Statuses

| Status | Meaning | Action |
|--------|---------|--------|
| PENDING | Awaiting processing | Monitor for updates |
| PROCESSING | In flight | Wait for terminal status |
| SUCCESS | Completed | Transaction finished |
| FAILED | Could not process | Check `meta.failureReason.code` |
| CUSTOMER_ACTION_REQUIRED | Needs OTP/authorization | Call authorize endpoint |
| IN_REVIEW, CHECKER_APPROVAL_REQUIRED | Internal review | Non-terminal, keep listening |

### HTTP Status Codes

| Code | Meaning |
|------|---------|
| 200 | Success |
| 201 | Created |
| 400 | Bad request (check payload) |
| 401 | Unauthorized (invalid/missing API key or insufficient permissions) |
| 404 | Resource not found |
| 409 | Conflict (customer email/phone already exists) |
| 429 | Rate limit exceeded |
| 503 | Service unavailable (retry with backoff) |

## Decision Guidance

### When to Use REST vs SDK

| Scenario | Use |
|----------|-----|
| TypeScript/Node.js project | SDK (`@afriex/sdk`) — type-safe, built-in retries, webhook verification |
| Other languages (Python, Go, Java) | REST API with HTTP client |
| Endpoints not in SDK (SME registration, media upload, settlement advice) | REST API directly |
| AI assistant integration | MCP server at `https://mcp.afriex.com/mcp` |

### When to Use Staging vs Production

| Scenario | Environment |
|----------|-------------|
| Development, testing, integration | Staging (`sandbox.api.afriex.com`) |
| Testing transaction outcomes (auto-settle in 5-6 min) | Staging only |
| Live transactions, real money | Production (`api.afriex.com`) |
| Never mix | Keep separate API keys per environment |

### When to Use WITHDRAW vs DEPOSIT

| Scenario | Type | Payment Method |
|----------|------|-----------------|
| Send payout to customer's bank/mobile | WITHDRAW | destinationId (recipient's account) |
| Collect payment from customer | DEPOSIT | sourceId (customer's account) |
| Convert USD to NGN in your wallet | SWAP | None (wallet-to-wallet) |

### When to Use Polling vs Webhooks

| Scenario | Approach |
|----------|----------|
| Real-time status updates, production | Webhooks (configure in dashboard, allowlist IPs) |
| Testing, low-volume, simple flows | Polling with GET `/api/v1/transaction/{id}` |
| Avoid polling in production | Use webhooks instead (more reliable, lower latency) |

## Workflow

### 1. Set Up Authentication

1. Log in to [Afriex Dashboard](https://business.afriex.com)
2. Navigate to **Settings > API Keys** (or **Developer > API Keys**)
3. Toggle to **Test** mode for staging, **Live** for production
4. Generate a new API key and store securely
5. Never commit API keys to version control; use environment variables

### 2. Create a Customer

1. Call `POST /api/v1/customer` with `fullName`, `email`, `phone`, `countryCode`
2. Store the returned `customerId` — you'll need it for all transactions
3. Optionally update KYC info later with `PATCH /api/v1/customer/{customerId}/kyc`

### 3. Create a Payment Method

1. Call `POST /api/v1/payment-method` with customer ID, channel, and account details
2. Specify `type: WITHDRAW` (for payouts) or `type: DEPOSIT` (for collections)
3. Store the returned `paymentMethodId`
4. For bank accounts, ensure `accountName` uses Latin or Chinese characters only

### 4. Process a Transaction

1. Call `POST /api/v1/transaction` with:
   - `customerId`, `type` (WITHDRAW/DEPOSIT/SWAP)
   - `sourceAmount`, `sourceCurrency`, `destinationAmount`, `destinationCurrency`
   - `destinationId` (for WITHDRAW) or `sourceId` (for DEPOSIT)
   - `meta.idempotencyKey` (unique key to prevent duplicates)
   - `meta.reference` (your internal reference)
2. In staging, transaction auto-settles in ~5-6 minutes (or 30 seconds with `SIMULATE_INSTANT`)
3. Poll `GET /api/v1/transaction/{transactionId}` or listen for webhook

### 5. Handle Webhooks

1. Configure webhook URL in dashboard (**Developers > Webhooks**)
2. Allowlist Afriex IPs: Staging `34.234.189.210`, Production `34.197.33.100`
3. Verify `x-webhook-signature` header using RSA-SHA256 and public key from dashboard
4. Always verify against raw request body (not parsed JSON)
5. Return `200` quickly; do slow work after acknowledging
6. Afriex retries failed deliveries up to 12 times with exponential backoff

### 6. Test in Sandbox

1. Use staging environment and Test mode API keys
2. Create test customers and payment methods
3. Control transaction outcomes via `meta.reference`:
   - Include `fail` → transaction settles as FAILED
   - Include `SIMULATE_INSTANT` → settles in 30 seconds
   - Include `SIMULATE_OTP` → requires OTP authorization
4. Call `POST /api/v1/org/balance/topup` to credit test wallet (sandbox only)
5. Verify webhook handling with `POST /api/v1/webhook/trigger` (sandbox only)

## Common Gotchas

- **API key permissions**: A 401 error may mean missing permissions, not a bad key. Check dashboard **Developer > API Keys** for required permissions.
- **Environment mismatch**: Staging and production use different API keys and base URLs. Never use production keys in staging.
- **Webhook signature verification**: Always verify against raw request body bytes, not parsed JSON. Use the correct public key for your environment (staging and production keys differ).
- **Payment method type mismatch**: Create WITHDRAW methods for payouts, DEPOSIT for collections. Using the wrong type causes transaction failures.
- **Account name validation**: Bank account names must use Latin (A-Z, accented) or Chinese characters. Arabic, Cyrillic, or other scripts are rejected.
- **Idempotency keys**: Use unique `meta.idempotencyKey` for critical operations to prevent duplicate transactions on retry.
- **Phone number format**: Phone numbers must be in E.164 format (e.g., `+2348192837465`). Mismatch with country code causes rejection.
- **Email/phone uniqueness**: Creating a customer with duplicate email or phone returns 409 with existing `customerId` in response. Reuse the existing customer instead of creating a new one.
- **Transaction status polling**: Don't treat intermediate statuses (PROCESSING, CUSTOMER_ACTION_REQUIRED) as final. Always branch on the `status` field, not the event type.
- **OTP authorization**: When `meta.otpRequired` is true, call `POST /api/v1/transaction/{id}/authorize` with OTP `123456` in sandbox.
- **Failure reason codes**: Branch on `meta.failureReason.code` (AFX_* codes), not the message. Codes are stable; messages may change.
- **Rate limiting**: Implement exponential backoff for retries. Cache exchange rates and use webhooks instead of polling.
- **Webhook retries**: Afriex retries up to 12 times. Return 200 quickly; slow processing should happen after acknowledgment.
- **Staging auto-settle**: Transactions in sandbox settle automatically in 5-6 minutes. Use `SIMULATE_INSTANT` to speed up testing.

## Verification Checklist

Before submitting work with Afriex API:

- [ ] API key is stored in environment variables, not hardcoded
- [ ] Using correct base URL for environment (staging vs production)
- [ ] Customer created with valid `fullName`, `email`, `phone`, `countryCode`
- [ ] Payment method created with correct `type` (WITHDRAW or DEPOSIT)
- [ ] Transaction includes `meta.idempotencyKey` for critical operations
- [ ] Phone numbers in E.164 format (e.g., `+2348192837465`)
- [ ] Account names use only Latin or Chinese characters (no Arabic/Cyrillic)
- [ ] Webhook signature verified against raw request body with correct public key
- [ ] Webhook endpoint returns 200 quickly after verification
- [ ] Webhook IPs allowlisted on firewall (staging: 34.234.189.210, production: 34.197.33.100)
- [ ] Error handling branches on `meta.failureReason.code`, not message
- [ ] Transaction status checks branch on `status` field, not event type
- [ ] OTP authorization handled when `meta.otpRequired` is true
- [ ] Retry logic uses exponential backoff for 503 and 429 responses
- [ ] Tested in staging before going to production
- [ ] Sandbox test outcomes controlled via `meta.reference` (fail, SIMULATE_INSTANT, SIMULATE_OTP)

## Resources

**Comprehensive navigation:** See [llms.txt](https://docs.afriex.com/llms.txt) for a complete page-by-page listing of all documentation.

**Critical pages:**
- [API Reference Introduction](https://docs.afriex.com/api-reference/introduction) — Authentication, versioning, core resources
- [Quickstart Guide](https://docs.afriex.com/quickstart) — First API call in 5 minutes
- [Connecting the Pieces](https://docs.afriex.com/guides/connecting-the-pieces) — How customers, payment methods, and transactions link together
- [Webhook Documentation](https://docs.afriex.com/api-reference/endpoint/webhooks/introduction) — Setup, security, signature verification
- [SDK Documentation](https://docs.afriex.com/sdk/introduction) — TypeScript SDK, configuration, retries
- [MCP Server](https://docs.afriex.com/mcp/introduction) — AI assistant integration
- [Integration Guide](https://docs.afriex.com/development) — Best practices, error handling, testing, idempotency

---

> For additional documentation and navigation, see: https://docs.afriex.com/llms.txt