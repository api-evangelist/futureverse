---
name: futureverse-subscribe-to-asset-events
description: >-
  Register a webhook endpoint and subscribe to Asset Register events so an application is notified
  in real time when assets are linked or unlinked on a collection. Use when an agent or service must
  react to register state changes rather than poll for them.
api: Futureverse Asset Register API
endpoint: https://ar-api.futureverse.app/graphql
operations:
  - Mutation.createWebhookEndpoint
  - Mutation.createWebhookSubscription
  - Mutation.updateWebhookEndpoint
  - Mutation.deleteWebhookSubscription
  - Mutation.deleteWebhookEndpoint
  - Query.webhookEndpoints
  - Query.webhookSubscriptions
generated: '2026-08-16'
method: generated
source: >-
  Grounded in graphql/futureverse-asset-register.graphql (live introspection, 2026-08-16) and
  https://docs.therootnetwork.com/asset-register/guides/subscriptions.
---

# Subscribe to Asset Register events

## What this does

The Asset Register is event driven. Instead of polling `assetTree`, register an HTTPS endpoint and
subscribe it to the event types and collections you care about. Deliveries arrive as POSTs carrying
the **full transaction** that contained the event.

## Authentication

Every operation on this page is admin functionality. You need a **SIWE** (Sign-In With Ethereum)
token in the `Authorization` header — generate one at
`https://ar-docs.futureverse.app/siwe-generator/` and see
`authentication/futureverse-authentication.yml`.

## Steps

### 1. Create the endpoint

```graphql
mutation CreateWebhookEndpoint($input: CreateWebhookEndpointInput!) {
  createWebhookEndpoint(input: $input) {
    webhookEndpoint { id webhookId subscriber url apiKey retries createdAt }
  }
}
```

```json
{ "input": { "url": "https://example.com/webhook", "retries": 5 } }
```

Keep two things from the response:

- `webhookId` — required to create any subscription.
- `apiKey` — the value the Subscription Service will send in the `API_KEY` header on every
  delivery. **Store it as a secret.** It is your only means of verifying the request came from
  Futureverse, and it cannot be rotated in place — rotating means recreating the endpoint.

`retries` is capped at **20**.

### 2. Subscribe

```graphql
mutation CreateWebhookSubscription($input: CreateWebhookSubscriptionInput!) {
  createWebhookSubscription(input: $input) {
    webhookSubscription { id subscriptionId webhookId type actions collectionId createdAt }
  }
}
```

```json
{
  "input": {
    "type": "asset-link",
    "actions": ["create", "delete"],
    "collectionId": "1:evm:0x6bca6de2dbdc4e0d41f7273011785ea16ba47182",
    "webhookId": "<from step 1>"
  }
}
```

`asset-link` is the only event type Futureverse documents. `WebhookSubscription.type` is a plain
`String` in the schema rather than an enum, so the complete set of subscribable types cannot be
enumerated from the API either — see `asyncapi/futureverse-asset-register-events.yml`.

### 3. Handle the delivery

```json
{
  "status": "SUCCESS",
  "signature": "0x00",
  "message": "",
  "events": [
    {
      "type": "asset-link",
      "action": "delete",
      "args": ["equipWith_asmBrain", "did:fv-asset:1:evm:0x6bca…:1000", "did:fv-asset:1:evm:0x1ea6…:200"],
      "collectionId": "1:evm:0x6bca…"
    }
  ]
}
```

Verify the `API_KEY` header before processing. Note what is **not** there: no HMAC signature over
the body and no timestamp, so you cannot detect a replayed delivery. If replay matters, deduplicate
on the transaction identity yourself.

### 4. Control retries with your status code

| You return | Futureverse does |
|---|---|
| 2xx | Acknowledged, no retry |
| 409, 429, 5xx | Retries, up to your `retries` budget |
| 1xx, 3xx, 4xx (not 429) | Never retries — the event is dropped |

Send `Retry-After` on a 429 to control backoff. The service takes the more conservative of your
value and its own policy; a **negative** `Retry-After` suppresses the retry entirely.

### 5. Audit and tear down

```graphql
query { webhookEndpoints { edges { node { webhookId url retries } } } }
query { webhookSubscriptions { edges { node { subscriptionId type actions collectionId } } } }
```

Use `deleteWebhookSubscription` then `deleteWebhookEndpoint` to remove them.

## Gotcha worth planning around

A 4xx that is not 429 is terminal. A deploy that briefly returns 404 or 401 on your webhook path
will silently lose every event delivered during that window, and there is **no replay API** to
recover them.
