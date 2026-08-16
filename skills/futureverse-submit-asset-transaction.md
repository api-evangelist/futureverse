---
name: futureverse-submit-asset-transaction
description: >-
  Create or delete a link between two Futureverse assets by building, signing and submitting an
  ARTM (Asset Rights Token Metadata) transaction to the Asset Register. Use when an agent must
  change register state — equip, unequip, attach provenance, or link an off-chain asset.
api: Futureverse Asset Register API
endpoint: https://ar-api.futureverse.app/graphql
operations:
  - Query.getNonceForChainAddress
  - Mutation.submitTransaction
  - Mutation.assetMutation
  - Query.transaction
  - Query.getTransaction
generated: '2026-08-16'
method: generated
source: >-
  Grounded in graphql/futureverse-asset-register.graphql (live introspection, 2026-08-16),
  https://docs.therootnetwork.com/asset-register/guides/asset-register-transaction,
  /asset-register/guides/asset-link-operations and /asset-register/guides/error-messages.
---

# Submit an Asset Register transaction

## What this does

Asset Register state changes are not authenticated with a token. They are **signed messages**. You
build a human-readable ARTM message, the asset owner's wallet signs it, and you submit the signature
with the message. The register verifies ownership from the signature.

## Preconditions

- The signing wallet must own the asset. A mismatch fails with `INVALID_ASSET_OWNER` (code 6).
- Use `@futureverse/artm` to build the message. Hand-assembling it is the most common cause of
  `INVALID_ARTM` (code 5).
- An asset may have **one** parent only, and each path under a parent must be unique.

## Steps

### 1. Fetch the current nonce — every time

```graphql
query Nonce($input: NonceInput!) {
  getNonceForChainAddress(input: $input)
}
```

The nonce is the replay guard. Reusing one fails with `INVALID_NONCE` (code 2). There is **no
idempotency key** on this API: if a submit times out, you cannot safely resubmit the same message,
because the nonce may already have advanced. Re-read the nonce and re-check state with
`Query.transaction` before retrying.

### 2. Build and sign the ARTM

The message follows the Ethereum message standard and is deliberately readable by the person
signing it. Sign it with the owner's wallet.

### 3. Submit

```graphql
mutation Submit($input: SubmitTransactionInput!) {
  submitTransaction(input: $input) { __typename }
}
```

For asset operations expressed as a mutation rather than a raw transaction:

```graphql
mutation AssetOp($input: AssetMutationInput!) {
  assetMutation(input: $input) {
    __typename
    ... on AssetOperationResultFailure { errors { message } }
  }
}
```

`AssetMutationInput.transaction` is an `AssetTransactionMessage`.

### 4. Confirm

```graphql
query Tx($hash: TransactionHash!) {
  transaction(hash: $hash) {
    status
    events { type action args }
    error { code message }
  }
}
```

`TransactionError.code` is the one place the Asset Register gives you a stable machine-readable
failure code. Branch on it.

## Failure codes worth handling explicitly

| Code | Meaning | What to do |
|---|---|---|
| `INVALID_NONCE` (2) | Nonce reused or malformed | Re-fetch with `getNonceForChainAddress`, rebuild, resign |
| `INVALID_ASSET_OWNER` (6) | Signer does not own the asset | Verify on-chain ownership before signing |
| `ASSET_ALREADY_LINKED` (14) | Asset is already linked | Delete the existing link first |
| `PARENT_PATH_IN_USE` (12) | Another child occupies that path | Choose a different path |
| `INVALID_ASSET_PARENT` (8) | Asset would get a second parent | Unlink before relinking |
| `FAIL_ARTM_VALIDATION` (1) | Signature unverifiable | Re-sign; do not modify the message after signing |

Full catalog: `errors/futureverse-problem-types.yml`.

## Operational warning

This writes to infrastructure whose operating company entered receivership on 2025-09-30 and
liquidation on 2025-12-16. There is no SLA, no status page and no deprecation policy —
see `lifecycle/futureverse-lifecycle.yml`. Treat write paths as best-effort and always confirm
with `Query.transaction` rather than assuming success.
