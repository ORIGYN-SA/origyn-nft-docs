---
icon: layer-group
metaLinks:
  alternates:
    - >-
      https://app.gitbook.com/s/yE16Xb3IemPxJWydtPOj/getting-started/collections-and-certificates
---

# Collections & Certificates

## Collections

A collection is an ORIGYN NFT canister that holds your certificates. Each collection is created from a [template](templates.md) and is managed by the Minting Studio. The Minting Studio handles canister creation, cycle management, and infrastructure and you only focus on defining your template and minting certificates.

### Collection Lifecycle

When you create a collection, it progresses through these states:

```
Queued → Created → Installed → TemplateUploaded → Ready for minting
```

| Status               | Description                                                            |
| -------------------- | ---------------------------------------------------------------------- |
| **Queued**           | Request received, waiting to be processed                              |
| **Created**          | Canister created on the Internet Computer                              |
| **Installed**        | ORIGYN NFT WASM code installed on the canister                         |
| **TemplateUploaded** | Template metadata uploaded to the collection(ready for minting)        |
| **Failed**           | An attempt failed (see error reason). The system **retries** automatically |

**Failure path:** `Failed` does not mean you have lost the collection or that a refund is on its way. Creation is retried automatically about once a minute. Only once the retry limit is exhausted is a reimbursement requested:

```
Failed → (retries) → ReimbursingQueued → Reimbursed
```

Reimbursement is delivered by a background job, so it is never instant. If the payout itself cannot be completed, the status becomes `QuarantinedReimbursement` with a reason, and requires manual resolution.

***

### Creating a Collection

Before creating a collection, you need:

1. A registered template (see [Templates](templates.md))
2. An approved OGY spend allowance (see [Getting Started](getting-started.md))

{% hint style="danger" %}
**This spends 15,000 OGY.** Test it sends a real production request.
{% endhint %}

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/create_collection" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

**Using dfx instead**

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai create_collection '(record {
  categories = vec {};
  name = "My Gold Bar Collection";
  description = "Certified gold bar certificates with full provenance tracking";
  symbol = "GBC";
  template_id = 1
})'
```

**Returns:** `collection_id` (nat), use this to monitor the collection's status.

**Parameters:**

| Field         | Type     | Description                                                                                                        |
| ------------- | -------- | ------------------------------------------------------------------------------------------------------------------ |
| `categories`  | vec text | Category names to attach. Required (not optional); pass `vec {}` for none. Each name must already exist in the taxonomy. |
| `name`        | text     | Collection display name                                                                                            |
| `description` | text     | Description of the collection                                                                                      |
| `symbol`      | text     | Short symbol (e.g., "GBC")                                                                                         |
| `template_id` | nat      | ID of a previously registered template                                                                             |

The collection creation process typically completes in under a minute. Monitor progress with `get_collection_info`.

***

### Querying Collections

#### Get Collection Info (by ID or Canister)

```bash
# By collection ID
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai get_collection_info \
  '(variant { CollectionId = 1 })'

# By canister ID (once the collection is created)
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai get_collection_info \
  '(variant { CanisterId = principal "<collection_canister_id>" })'
```

**Returns:** `opt CollectionInfo`, or `null` if no collection matches the id or canister you passed. The record contains:

| Field           | Description                                                  |
| --------------- | ------------------------------------------------------------ |
| `collection_id` | Unique collection identifier                                 |
| `status`        | Current lifecycle status                                     |
| `canister_id`   | The ORIGYN NFT canister principal (available after creation) |
| `metadata`      | Collection name, symbol, description, template\_id, and categories |
| `owner`         | Principal of the collection creator                          |
| `ogy_charged`   | OGY tokens charged for creation                              |
| `created_at`    | Creation timestamp                                           |

#### List Your Collections

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/owners/{principal}/collections" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

**Using dfx instead**

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai get_collections_by_owner '(record {
  owner = principal "<your_principal>";
  pagination = record { offset = opt 0; limit = opt 10 }
})'
```

**Returns:** `CollectionsResult` with a vector of `CollectionInfo` and `total_count`.

***

## Certificates (NFTs)

Certificates are the individual ORIGYN NFTs within a collection. Each certificate contains metadata structured according to the collection's template. They implement the ICRC-7 standard and can be transferred, approved, and queried using standard ICRC-7/ICRC-37 methods (see [ICRC-37 / ICRC-7](../technical-reference/icrc37-icrc7.md)).

> **Looking for JSON metadata or paginated listings?** The endpoints in this section return raw ICRC-7 metadata at the collection canister level. For ready-to-render JSON plus collection info (name, symbol, logo) in a single response, see [Querying NFTs](#querying-nfts) at the bottom of this page.

### Viewing Certificate Details

Fetch the metadata for one or more certificates:

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai get_nft_details '(record {
  canister_id = principal "<collection_canister_id>";
  token_ids = vec { 1; 2; 3 }
})'
```

**Returns:** A vector of `NftDetails`. Burned tokens are dropped, so the vector can be **shorter than `token_ids`** — match entries by `token_id`, never by position. Each entry contains:

| Field      | Description                                                                                                |
| ---------- | ---------------------------------------------------------------------------------------------------------- |
| `token_id` | The certificate's token ID within the collection                                                           |
| `owner`    | Current owner (principal + optional subaccount)                                                            |
| `metadata` | Raw ICRC-7 metadata map, `opt vec record { text; ICRC3Value }`. **Not** a JSON string. Keys follow [Producing your mint JSON](minting.md#producing-your-mint-json-from-a-template). For a ready-to-parse JSON string, use `get_nft` |

### Listing Certificates in a Collection

Paginate through all certificates in a collection:

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai get_collection_nfts '(record {
  canister_id = principal "<collection_canister_id>";
  pagination = record { offset = null; limit = opt 20 }
})'
```

**Returns:** A vector of token IDs (nat). The HTTP equivalent is live here:

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/collections/{canister_id}/token-ids" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

{% hint style="warning" %}
`offset` behaves differently here than on the other paginated calls. For this endpoint it is a **cursor**, not a skip count: pass the last token ID from the previous page to get the next set, and `null` for the first page.
{% endhint %}

For the complete end-to-end flow from template design to certificate viewing, see [How It Works](../core-concepts/how-it-works.md).

***

## Querying NFTs

Two different things live in this section, and the difference matters right now.

{% hint style="danger" %}
### The indexer-backed queries are being retired on 31 August 2026

`get_nfts_by_owner`, `get_nfts_by_holder`, `get_past_nfts_by_holder`, `get_holders_by_collection`, `get_account_stats` and `get_collection_stats` are **deprecated** and will stop being supported after **31 August 2026**.

They are replaced by the HTTP indexer API, which is faster, paginated consistently, sortable, and needs no Internet Computer client at all. Migrate now; see the mapping table below.

**`get_nft` is not affected** and is staying.
{% endhint %}

### Migrating to the HTTP API

Every deprecated call has a direct HTTP equivalent under `https://gateway.origyn.com/v1/nft/production/`. These endpoints are public and need no API key.

| Deprecated canister call | HTTP replacement |
| ------------------------ | ---------------- |
| `get_nfts_by_owner` | `GET /owners/{principal}/nfts` |
| `get_nfts_by_holder` | `GET /accounts/{principal}/nfts` |
| `get_past_nfts_by_holder` | `GET /accounts/{principal}/past-nfts` |
| `get_holders_by_collection` | `GET /collections/{canister_id}/holders` |
| `get_account_stats` | `GET /accounts/{principal}/stats` |
| `get_collection_stats` | `GET /collections/{canister_id}/stats` |

Each replacement is live below. They are public, so you can call any of them right here.

### `get_nfts_by_owner` → `/owners/{principal}/nfts`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/owners/{principal}/nfts" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### `get_nfts_by_holder` → `/accounts/{principal}/nfts`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/accounts/{principal}/nfts" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### `get_past_nfts_by_holder` → `/accounts/{principal}/past-nfts`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/accounts/{principal}/past-nfts" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### `get_holders_by_collection` → `/collections/{canister_id}/holders`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/collections/{canister_id}/holders" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### `get_account_stats` → `/accounts/{principal}/stats`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/accounts/{principal}/stats" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### `get_collection_stats` → `/collections/{canister_id}/stats`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/collections/{canister_id}/stats" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

Four differences to plan for when you port:

* **`owners` and `accounts` are not the same thing.** Under `/owners/{principal}/…` the principal is a collection **creator**; under `/accounts/{principal}/…` it is a current **holder**. Map each call to the right family using the table above rather than by name.
* **Timestamps are milliseconds**, not nanoseconds, and the field names say so (`minted_at_ms`, `released_at_ms`).
* **Pagination is `?limit=` and `?offset=`**, with `limit` defaulting to 50 and capped at 100. Out-of-range values are rejected with a 400 rather than clamped.
* **Principals carry no subaccount** on the HTTP surface; they are plain strings.

The four **list** endpoints additionally support `?sort=` and `?order=`, which the canister calls never had. The two `/stats` endpoints take no query parameters.

{% hint style="info" %}
The full, always-current schema for every HTTP endpoint is published at [gateway.origyn.com/docs](https://gateway.origyn.com/docs/).
{% endhint %}

### Pagination model (deprecated calls)

Every deprecated list endpoint takes the same pagination arguments and returns the same envelope.

```
type PaginationArgs = record {
  offset : opt nat64;
  limit  : opt nat64;
};
```

| Field    | Default | Notes                       |
| -------- | ------- | --------------------------- |
| `offset` | `0`     | Number of items to skip     |
| `limit`  | `100`   | Capped at `100` server-side |

Responses use the same envelope:

```
type PaginatedResponse<T> = record {
  items       : vec T;
  total       : nat64;
  collections : vec NftCollectionInfo;
};
```

`collections` contains one entry per unique collection referenced by `items`. Each `NftCollectionInfo` has `canister_id`, `name`, `symbol`, `description`, `logo`, sourced from the ICRC-7 well-known keys on the collection canister (`icrc7:name`, `icrc7:symbol`, etc.).

### `get_nft` (single certificate)

{% hint style="success" %}
**Not deprecated.** `get_nft` is staying.
{% endhint %}

Fetch one certificate plus its collection-level info.

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai get_nft '(record {
  collection = principal "<collection_canister_id>";
  token_id   = 1 : nat
})'
```

Unlike the deprecated calls in this section, `get_nft` does **not** read from the indexer. It queries the collection canister directly, which is precisely why it is being kept: a certificate is readable the moment it is minted, with no indexing delay.

**Returns:** `opt NftDetailView`. `null` means the token does not exist on the collection canister, or the call to it failed. It never means "not indexed yet".

| Field               | Description                                        |
| ------------------- | -------------------------------------------------- |
| `nft.collection`    | Collection canister principal                      |
| `nft.token_id`      | Token ID within the collection                     |
| `nft.owner`         | Current owner (`opt Account`)                      |
| `nft.metadata_json` | The JSON metadata string the NFT was minted with   |
| `collection_info`   | Name, symbol, description, logo for the collection |

{% hint style="info" %}
If you need just-minted certificates over HTTP, use the live read `GET /collections/{canister_id}/nfts/{token_id}`, which also goes straight to the collection canister. The similarly named `GET /collections/{canister_id}/tokens/{token_id}` is served from the index and lags slightly behind a mint.
{% endhint %}


### `get_nfts_by_owner` (collection-owner view)

{% hint style="danger" %}
**Deprecated, retiring 31 August 2026.** Use `GET /owners/{principal}/nfts` instead. See [Migrating to the HTTP API](#migrating-to-the-http-api).
{% endhint %}

Lists NFTs from the perspective of a **collection-owner principal**. The response unions two sources, deduplicated by `(collection, token_id)`:

1. NFTs minted under collections owned by `owner` (status `Minted` if still held by `owner`, `Transferred` if held by someone else).
2. NFTs currently held by `owner` from collections they do **not** own (status `Received`).

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai get_nfts_by_owner '(record {
  owner      = principal "<owner_principal>";
  collection = null;
  statuses   = null;
  pagination = record { offset = opt 0; limit = opt 50 }
})'
```

| Arg          | Type                | Description                                          |
| ------------ | ------------------- | ---------------------------------------------------- |
| `owner`      | `principal`         | Collection-owner principal                           |
| `collection` | `opt principal`     | Filter to a single collection canister               |
| `statuses`   | `opt vec NftStatus` | Filter by `Minted`, `Transferred`, and/or `Received` |
| `pagination` | `PaginationArgs`    | See above                                            |
| `categories` | `opt vec text`      | Restrict to collections carrying any of these category names |

**Returns:** `PaginatedResponse<NftViewWithStatus>`. Each item has the standard `NftView` fields plus `status`.

**`NftStatus`:**

| Variant       | Meaning                                                       |
| ------------- | ------------------------------------------------------------- |
| `Minted`      | From `owner`'s collection, currently held by `owner`.         |
| `Transferred` | From `owner`'s collection, currently held by another account. |
| `Received`    | Not from `owner`'s collection but currently held by `owner`.  |

### `get_nfts_by_holder` (holder-account view)

{% hint style="danger" %}
**Deprecated, retiring 31 August 2026.** Use `GET /accounts/{principal}/nfts` instead. See [Migrating to the HTTP API](#migrating-to-the-http-api).
{% endhint %}

Lists NFTs currently held by a specific `Account`, regardless of who minted them.

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai get_nfts_by_holder '(record {
  holder     = record { owner = principal "<holder_principal>"; subaccount = null };
  collection = null;
  pagination = record { offset = opt 0; limit = opt 50 }
})'
```

**Returns:** `PaginatedResponse<NftView>`. `timestamp_nanos` on each item is the `acquired_at` timestamp.

### `get_past_nfts_by_holder` (history)

{% hint style="danger" %}
**Deprecated, retiring 31 August 2026.** Use `GET /accounts/{principal}/past-nfts` instead. See [Migrating to the HTTP API](#migrating-to-the-http-api).
{% endhint %}

Lists NFTs that `holder` previously owned and no longer holds. **Only holdings released by a transfer are returned.** Burned tokens are excluded, because a burn destroys the metadata on the collection canister and the entry would carry no useful payload.

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai get_past_nfts_by_holder '(record {
  holder     = record { owner = principal "<holder_principal>"; subaccount = null };
  pagination = record { offset = opt 0; limit = opt 50 }
})'
```

**Returns:** `PaginatedResponse<PastNftView>`. Each item:

| Field            | Description                                                            |
| ---------------- | ---------------------------------------------------------------------- |
| `acquired_at`    | Nanosecond timestamp when `holder` acquired the token                  |
| `released_at`    | Nanosecond timestamp when `holder` lost the token                      |
| `release_reason` | Always `variant { Transfer : record { to : Account } }` here. The type also has a `Burn` arm, but burns are filtered out of this response |
| `current_owner`  | Whoever holds the token now (`null` if burned)                         |
| `metadata_json`  | JSON metadata string                                                   |

Use this for an owner's transfer history view. It will not show burns.

### `get_holders_by_collection`

{% hint style="danger" %}
**Deprecated, retiring 31 August 2026.** Use `GET /collections/{canister_id}/holders` instead. See [Migrating to the HTTP API](#migrating-to-the-http-api).
{% endhint %}

Lists current holders of every token in a collection.

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai get_holders_by_collection '(record {
  collection = principal "<collection_canister_id>";
  pagination = record { offset = opt 0; limit = opt 100 }
})'
```

**Returns:** `PaginatedResponse<NftView>`.

### `get_account_stats`

{% hint style="danger" %}
**Deprecated, retiring 31 August 2026.** Use `GET /accounts/{principal}/stats` instead. See [Migrating to the HTTP API](#migrating-to-the-http-api).
{% endhint %}

Aggregate stats for an account across every indexed collection.

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai get_account_stats '(record {
  account = record { owner = principal "<account_principal>"; subaccount = null }
})'
```

| Field                  | Description                                                                  |
| ---------------------- | ---------------------------------------------------------------------------- |
| `total_owned`          | Total tokens currently owned across all collections                          |
| `per_collection`       | `vec record { principal; nat64 }`: owned count per collection                |
| `distinct_collections` | Number of distinct collections in which the account holds at least one token |
| `first_activity_nanos` | First time this account was observed (`opt nat64`)                           |
| `last_activity_nanos`  | Most recent activity (`opt nat64`)                                           |

### `get_collection_stats`

{% hint style="danger" %}
**Deprecated, retiring 31 August 2026.** Use `GET /collections/{canister_id}/stats` instead. See [Migrating to the HTTP API](#migrating-to-the-http-api).
{% endhint %}

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai get_collection_stats '(record {
  collection = principal "<collection_canister_id>"
})'
```

| Field                | Description                                                                        |
| -------------------- | ---------------------------------------------------------------------------------- |
| `distinct_holders`   | Number of distinct accounts holding at least one token in this collection          |
| `total_tokens`       | Total tokens of this collection currently tracked by the indexer                   |
| `next_indexed_block` | First ICRC-3 block id the indexer has not yet processed (use as a freshness probe) |
