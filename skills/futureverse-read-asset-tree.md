---
name: futureverse-read-asset-tree
description: >-
  Resolve a Futureverse asset and everything linked to it into a single JSON-LD asset tree, using
  the Asset Register GraphQL API. Use when an agent needs to know what an NFT/SFT is, what it is
  composed of, and what is equipped to it, across chains.
api: Futureverse Asset Register API
endpoint: https://ar-api.futureverse.app/graphql
staging_endpoint: https://ar-api.futureverse.cloud/graphql
operations:
  - Query.assetTree
  - Query.getAssetTree
  - Query.asset
  - Query.assets
generated: '2026-08-16'
method: generated
source: >-
  Grounded in graphql/futureverse-asset-register.graphql (live introspection of
  https://ar-api.futureverse.app/graphql, 2026-08-16) and
  https://docs.therootnetwork.com/asset-register/guides/view-asset-tree.
---

# Read a Futureverse asset tree

## What this does

The Asset Register stores relationships between assets — an accessory equipped to a character, a
provenance record attached to a collectible, an off-chain asset linked to an on-chain one. The
**asset tree** is the resolved composition: one call returns the asset plus everything linked
beneath it, as a JSON-LD document.

## Before you start

- **No credentials are needed for reads.** Introspection and queries on
  `https://ar-api.futureverse.app/graphql` answered anonymously on 2026-08-16.
- Identify the asset by its Futureverse DID:
  `did:fv-asset:<chainId>:<chainType>:<contractAddress>:<tokenId>` —
  e.g. `did:fv-asset:1:evm:0x6bca6de2dbdc4e0d41f7273011785ea16ba47182:1000`.
- Collection IDs are the same string without the token: `1:evm:0x6bca…`.

## Steps

### 1. Find the asset if you only have an owner address

```graphql
query OwnedAssets($addresses: [ChainAddress!], $first: Float) {
  assets(addresses: $addresses, first: $first) {
    edges { node { tokenId collectionId assetType } }
    pageInfo { hasNextPage endCursor }
  }
}
```

`assets` is a Relay connection. Page with `first`/`after`, never by re-querying with a larger
`first`. Optional narrowing arguments that exist on this field: `collectionIds`, `schemaId`,
`chainId`, `chainType`, `filter` (an `AssetFilter` with `hasFilters`, `eqFilters`, `search`),
`sort`, and `removeDuplicates`.

### 2. Read one asset directly

```graphql
query OneAsset($tokenId: String!, $collectionId: CollectionId!) {
  asset(tokenId: $tokenId, collectionId: $collectionId) {
    tokenId
    assetType
    metadata { properties attributes }
    schema { namespace }
    collection { name }
  }
}
```

### 3. Resolve the tree

```graphql
query Tree($input: AssetTreeInput!) {
  assetTree(input: $input) { data }
}
```

`data` is JSON-LD. Use `getAssetTree(input:)` instead when you want the typed
`AssetTreeResponse` union, which returns either `AssetTreeResultSuccess { data }` or
`AssetTreeResultFailure { errors { message } }`.

### 4. Read the inventory projection when you care about equip relationships

```graphql
query Inventory($owner: ChainAddress!, $first: Float) {
  inventoryAssets(owner: $owner, first: $first) {
    edges { node { tokenId collectionId parentLink { parent { tokenId } } } }
    pageInfo { hasNextPage endCursor }
  }
}
```

## Handling failure

- **HTTP status is not the signal.** The Asset Register returns HTTP 200 on failure and encodes the
  outcome in a typed union. Always request `__typename` and branch on it — never on the status code.
- `AssetTreeResultFailure.errors[].message` is free text. Only `TransactionError` and
  `AssetRegisterErrorExtensions` carry machine-readable codes, so do not build control flow on
  message strings from other error objects.
- See `errors/futureverse-problem-types.yml` for the full catalog.

## What you cannot do here

There is no MCP tool for any of this. `mcp/futureverse-tool-crosswalk.yml` records the measurement:
zero of the 57 Asset Register GraphQL fields are exposed through the MCP server. An agent must speak
GraphQL directly.
