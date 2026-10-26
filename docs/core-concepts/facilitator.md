---
title: "Facilitator"
description: "This page explains the role of the **facilitator** in the x402 protocol."
---

The facilitator is an optional but recommended service that simplifies the process of verifying and settling payments between clients (buyers) and servers (sellers).

### What is a Facilitator?

The facilitator is a service that:

* Verifies payment payloads submitted by clients.
* Settles payments on the blockchain on behalf of servers.

By using a facilitator, servers do not need to maintain direct blockchain connectivity or implement payment verification logic themselves. This reduces operational complexity and ensures accurate, real-time validation of transactions.

### Facilitator Responsibilities

* **Verify payments:** Confirm that the client's payment payload meets the server's declared payment requirements.
* **Settle payments:** Submit validated payments to the blockchain and monitor for confirmation.
* **Provide responses:** Return verification and settlement results to the server, allowing the server to decide whether to fulfill the client's request.

The facilitator does not hold funds or act as a custodian - it performs verification and execution of onchain transactions based on signed payloads provided by clients.

### Choosing a Facilitator Path

There is no single facilitator deployment model for every x402 integration. In practice, most teams should choose one of three paths:

| Goal | Recommended path |
| ---- | ---------------- |
| Fastest testnet or local quickstart | Use the public `x402.org` facilitator |
| Managed production deployment | Use a production facilitator provider that supports your target network |
| Full operational control | Run your own facilitator or [self-facilitate](https://github.com/x402-foundation/x402/tree/main/examples/typescript/servers/self-facilitation) inside your resource server |

**Important:** the public `x402.org` facilitator is intended for development and testnet workflows. Do not assume it is the default path for production mainnet routes. For mainnet deployments, use a production facilitator that supports your network, run your own facilitator, or self-facilitate.

### Why Use a Facilitator?

Using a facilitator provides:

* **Reduced operational complexity:** Servers do not need to interact directly with blockchain nodes.
* **Protocol consistency:** Standardized verification and settlement flows across services.
* **Faster integration:** Services can start accepting payments with minimal blockchain-specific development.

While it is possible to implement verification and settlement locally, using a facilitator accelerates adoption and ensures correct protocol behavior.

### Live Facilitators

Multiple facilitators are live in production, supporting various networks including Base, Solana, Polygon, Avalanche, and more. See [Facilitators](/dev-tools/facilitators) for selected production options.

### Interaction Flow

1. `Client` makes an HTTP request to a `resource server`
2. `Resource server` responds with a `402 Payment Required` status and a `PAYMENT-REQUIRED` header containing the Base64-encoded payment requirements.
3. `Client` selects one of the `paymentDetails` returned by the `accepts` field of the server response and creates a `Payment Payload` based on the `scheme` of the `paymentDetails` they have selected.
4. `Client` sends the HTTP request with the `PAYMENT-SIGNATURE` header containing the `Payment Payload` (Base64-encoded) to the `resource server`
5. `Resource server` verifies the `Payment Payload` is valid either via local verification or by POSTing the `Payment Payload` and `Payment Details` to the `/verify` endpoint of the `facilitator server`.
6. `Facilitator server` performs verification of the object based on the `scheme` and `networkId` of the `Payment Payload` and returns a `Verification Response`
7. If the `Verification Response` is valid, the resource server performs the work to fulfill the request. If the `Verification Response` is invalid, the resource server returns a `402 Payment Required` status with the `PAYMENT-REQUIRED` header.
8. `Resource server` either settles the payment by interacting with a blockchain directly, or by POSTing the `Payment Payload` and `Payment Details` to the `/settle` endpoint of the `facilitator server`.
9. `Facilitator server` submits the payment to the blockchain based on the `scheme` and `networkId` of the `Payment Payload`.
10. `Facilitator server` waits for the payment to be confirmed on the blockchain.
11. `Facilitator server` returns a `Payment Execution Response` to the resource server.
12. `Resource server` returns a response to the `Client` with a `PAYMENT-RESPONSE` header containing the `Settlement Response` as Base64-encoded JSON. On success, this is a `200 OK` with the requested resource. On failure, this is a `402 Payment Required` with error details.

### Settlement pending and auto-recovery

When a settlement transaction is broadcast but its confirmation cannot be established — for example, due to an RPC error or a timeout while waiting for the receipt — the facilitator returns `settlement_pending` instead of a terminal failure. This is a **non-terminal** error: the transaction may still confirm on chain. The `SettlementResponse` carries the broadcast transaction hash in `transaction` and the network in `network`, so callers can reconcile on chain before deciding whether to retry.

This applies to the `exact`, `upto`, and `batch-settlement` schemes on both EVM and SVM (v2 only).

#### Automatic retry and reconciliation

The resource server automatically retries settlement exactly once when it receives a `settlement_pending` response. On the retry, the facilitator mechanism checks a `PendingSettlementStore` for the broadcast transaction hash keyed to the payment payload. When a match is found, the mechanism reconciles against the already-broadcast transaction instead of verifying and broadcasting a second one. This prevents double-spend and avoids redundant on-chain submissions.

The `PendingSettlementStore` is an interface, not a concrete type, so multi-instance facilitators (running several replicas with no session affinity) can supply a shared, network-backed implementation such as Redis instead of the default in-memory store. The in-memory default only works when the retry lands on the same process.

**Facilitators deployed behind a platform request deadline** (serverless functions, gateway timeouts) should bound the receipt wait below that deadline. If the process is killed mid-wait, the caller receives a 5xx with no transaction hash instead of `settlement_pending` with a hash to reconcile against.

In TypeScript, pass `confirmationTimeoutMs` to `toFacilitatorEvmSigner` to set this bound:

```typescript
import { toFacilitatorEvmSigner } from "@x402/evm";

const evmSigner = toFacilitatorEvmSigner(walletClient, {
  confirmationTimeoutMs: 25_000, // set a few seconds below your platform deadline
});
```

The default is `180_000` ms (3 minutes), matching viem's own default. In Python, pass `confirmation_timeout_seconds` to `FacilitatorWeb3Signer` (default `120`).

### Facilitator HTTP request timeout (resource server side)

When a resource server calls the facilitator over HTTP, each request — `verify`, `settle`, and `getSupported` — is subject to a per-request deadline. If the facilitator does not respond within that deadline, the call rejects with a `FacilitatorTimeoutError`.

The default timeout is **90 seconds**. You can override it by passing `timeoutMs` to `HTTPFacilitatorClient`:

```typescript
import { HTTPFacilitatorClient } from "@x402/core/http";

const facilitatorClient = new HTTPFacilitatorClient({
  url: "https://your-facilitator.com",
  timeoutMs: 30_000, // 30 seconds
});
```

`timeoutMs` must be a positive integer no greater than `2_147_483_647` (about 24.8 days). Values outside this range throw a `RangeError` at construction time.

**Note:** a timeout during `settle` is an indeterminate outcome — the facilitator may have completed the settlement even though the resource server did not receive a response. Handle `FacilitatorTimeoutError` from `settle` accordingly.

### Duplicate Settlement (Solana)

On Solana, a race condition can occur when the same payment transaction is submitted to a facilitator's `/settle` endpoint multiple times before the first submission is confirmed onchain. Because Solana's RPC returns "success" for duplicate submissions (the network deduplicates at the consensus level), the facilitator may return a successful settlement response for each call. A malicious client could exploit this to access multiple resources while only paying once.

To mitigate this, the x402 SVM mechanism packages include a built-in `SettlementCache` — a short-lived, in-memory cache that detects and rejects duplicate settlement attempts for the same transaction payload. The cache requires no external storage and entries are automatically evicted after 120 seconds (approximately twice the Solana blockhash lifetime).

This protection is enabled by default when using the standard SVM facilitator registration helpers in TypeScript and Python. In Go, a shared `SettlementCache` instance should be passed to both V1 and V2 SVM facilitator schemes during registration.

**If you are a merchant settling payments directly (without a facilitator), you must implement equivalent duplicate detection yourself.** See the [Exact SVM Scheme Specification](https://github.com/x402-foundation/x402/blob/main/specs/schemes/exact/scheme_exact_svm.md) for the full specification.

### Error codes

Facilitators return structured error codes in the `invalidReason` field of a `VerifyResponse` and the `errorReason` field of a `SettleResponse`. The optional `invalidMessage` and `errorMessage` fields provide human-readable diagnostics for logs and operators — their text is free-form and may change across facilitators or releases. When matching specific error conditions, always use the `invalidReason` or `errorReason` code and never rely on message text.

| Code | Returned by | Description |
| ---- | ----------- | ----------- |
| `insufficient_funds` | verify, settle | Client does not have enough tokens to complete the payment |
| `invalid_exact_evm_payload_authorization_valid_after` | verify, settle | Payment authorization is not yet valid (before `validAfter` timestamp) |
| `invalid_exact_evm_payload_authorization_valid_before` | verify, settle | Payment authorization has expired (after `validBefore` timestamp) |
| `invalid_exact_evm_payload_authorization_value_mismatch` | verify, settle | Payment amount does not exactly match the required amount |
| `invalid_exact_evm_payload_signature` | verify, settle | Payment authorization signature is invalid or improperly signed |
| `invalid_exact_evm_payload_recipient_mismatch` | verify, settle | Recipient address does not match payment requirements |
| `invalid_network` | verify, settle | Specified blockchain network is not supported |
| `invalid_payload` | verify, settle | Payment payload is malformed or contains invalid data |
| `invalid_payment_requirements` | verify, settle | Payment requirements object is invalid or malformed |
| `invalid_scheme` | verify, settle | Specified payment scheme is not supported |
| `unsupported_scheme` | verify, settle | Payment scheme is not supported by this facilitator |
| `invalid_x402_version` | verify, settle | Protocol version is not supported |
| `invalid_transaction_state` | settle | Blockchain transaction failed or was rejected |
| `unexpected_verify_error` | verify | Unexpected error occurred during payment verification |
| `unexpected_settle_error` | settle | Unexpected error occurred during payment settlement |
| `settlement_pending` | settle | Settlement transaction was broadcast but confirmation could not be established (non-terminal — see [Settlement pending and auto-recovery](#settlement-pending-and-auto-recovery)) |

### Summary

The facilitator acts as an independent verification and settlement layer within the x402 protocol. It helps servers confirm payments and submit transactions onchain without requiring direct blockchain infrastructure.

Next, explore:

* [Client / Server](/core-concepts/client-server) — understand the roles and responsibilities of clients and servers
* [HTTP 402](/core-concepts/http-402) — understand how payment requirements are communicated to clients
