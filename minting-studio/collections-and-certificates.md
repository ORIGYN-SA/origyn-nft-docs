---
icon: layer-group
metaLinks:
  alternates:
    - https://app.gitbook.com/s/yE16Xb3IemPxJWydtPOj/getting-started/collections-and-certificates
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
| **Failed**           | Something went wrong (see error reason). Automatic refund is initiated |

**Failure path:** If collection creation fails at any stage, the system automatically reimburses your 15,000 OGY fee:

```
Failed → ReimbursingQueued → Reimbursed
```

In rare cases where automatic reimbursement encounters an issue, the status becomes `QuarantinedReimbursement` and requires manual resolution.

---

### Creating a Collection

Before creating a collection, you need:

1. A registered template (see [Templates](templates.md))
2. An approved OGY spend allowance (see [Getting Started](getting-started.md))

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai create_collection '(record {
  name = "My Gold Bar Collection";
  description = "Certified gold bar certificates with full provenance tracking";
  symbol = "GBC";
  template_id = 1
})'
```

**Returns:** `collection_id` (nat), use this to monitor the collection's status.

**Parameters:**

| Field         | Type | Description                            |
| ------------- | ---- | -------------------------------------- |
| `name`        | text | Collection display name                |
| `description` | text | Description of the collection          |
| `symbol`      | text | Short symbol (e.g., "GBC")             |
| `template_id` | nat  | ID of a previously registered template |

The collection creation process typically completes in under a minute. Monitor progress with `get_collection_info`.

---

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

**Returns:** `CollectionInfo` record with:

| Field           | Description                                                  |
| --------------- | ------------------------------------------------------------ |
| `collection_id` | Unique collection identifier                                 |
| `status`        | Current lifecycle status                                     |
| `canister_id`   | The ORIGYN NFT canister principal (available after creation) |
| `metadata`      | Collection name, symbol, description, and template_id        |
| `owner`         | Principal of the collection creator                          |
| `ogy_charged`   | OGY tokens charged for creation                              |
| `created_at`    | Creation timestamp                                           |

#### List Your Collections

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai get_collections_by_owner '(record {
  owner = principal "<your_principal>";
  pagination = record { offset = opt 0; limit = opt 10 }
})'
```

**Returns:** `CollectionsResult` with a vector of `CollectionInfo` and `total_count`.

---

## Certificates (NFTs)

Certificates are the individual ORIGYN NFTs within a collection. Each certificate contains metadata structured according to the collection's template. They implement the ICRC-7 standard and can be transferred, approved, and queried using standard ICRC-7/ICRC-37 methods (see [ICRC-37 / ICRC-7](../getting-started/icrc37-icrc7.md)).

> **Looking for JSON metadata or paginated listings?** The endpoints in this section return raw ICRC-7 metadata at the collection canister level. For ready-to-render JSON plus collection info (name, symbol, logo) in a single response, see [Querying NFTs (indexer-backed)](#querying-nfts-indexer-backed) at the bottom of this page.

### Viewing Certificate Details

Fetch the metadata for one or more certificates:

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai get_nft_details '(record {
  canister_id = principal "<collection_canister_id>";
  token_ids = vec { 1; 2; 3 }
})'
```

**Returns:** A vector of `NftDetails`, each containing:

| Field      | Description                                                                                                |
| ---------- | ---------------------------------------------------------------------------------------------------------- |
| `token_id` | The certificate's token ID within the collection                                                           |
| `owner`    | Current owner (principal + optional subaccount)                                                            |
| `metadata` | Key-value pairs following ICRC3 format (see [Minting - Metadata Format](minting.md#metadata-format-icrc3)) |

### Listing Certificates in a Collection

Paginate through all certificates in a collection:

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai get_collection_nfts '(record {
  canister_id = principal "<collection_canister_id>";
  prev = null;
  take = opt 20
})'
```

**Returns:** A vector of token IDs (nat).

Use `prev` for pagination pass the last token ID from the previous page to get the next set.

For the complete end-to-end flow from template design to certificate viewing, see [How It Works](../getting-started/how-it-works.md).

---

## Querying NFTs (indexer-backed)

The Minting Studio runs an indexer that follows the ICRC-3 transaction stream of every collection it has provisioned. The endpoints below are served from that index. Compared to the lower-level `get_nft_details` / `get_collection_nfts` calls above, they return:

- `metadata_json: text` for each NFT, ready to parse and render (see [Producing your mint JSON](minting.md#producing-your-mint-json-from-a-template) for the shape).
- `collections` info (name, symbol, description, logo) bundled into every paginated response, so you do not need a separate ICRC-7 round trip.

### Pagination model

Every list endpoint takes the same pagination arguments and returns the same envelope.

```
type PaginationArgs = record {
  offset : opt nat64;
  limit  : opt nat64;
};
```

| Field    | Default | Notes                                  |
| -------- | ------- | -------------------------------------- |
| `offset` | `0`     | Number of items to skip                |
| `limit`  | `100`   | Capped at `100` server-side            |

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

Fetch one certificate plus its collection-level info.

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai get_nft '(record {
  collection = principal "<collection_canister_id>";
  token_id   = 1 : nat
})'
```

**Returns:** `opt NftDetailView`. `null` if the token does not exist or has not been indexed yet.

| Field                   | Description                                              |
| ----------------------- | -------------------------------------------------------- |
| `nft.collection`        | Collection canister principal                            |
| `nft.token_id`          | Token ID within the collection                           |
| `nft.owner`             | Current owner (`opt Account`)                            |
| `nft.metadata_json`     | The JSON metadata string the NFT was minted with         |
| `collection_info`       | Name, symbol, description, logo for the collection       |

### `get_nfts_by_owner` (collection-owner view)

Lists NFTs from the perspective of a **collection-owner principal**. The response unions two sources, deduplicated by `(collection, token_id)`:

1. NFTs minted under collections owned by `owner` (status `Minted` if still held by `owner`, `Transferred` if held by someone else).
2. NFTs currently held by `owner` from collections they do **not** own (status `Received`).

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai get_nfts_by_owner '(record {
  owner      = principal "<owner_principal>";
  collection = null;
  statuses   = null;
  pagination = record { offset = opt 0 : opt nat64; limit = opt 50 : opt nat64 }
})'
```

| Arg          | Type                       | Description                                            |
| ------------ | -------------------------- | ------------------------------------------------------ |
| `owner`      | `principal`                | Collection-owner principal                             |
| `collection` | `opt principal`            | Filter to a single collection canister                 |
| `statuses`   | `opt vec NftStatus`        | Filter by `Minted`, `Transferred`, and/or `Received`   |
| `pagination` | `PaginationArgs`           | See above                                              |

**Returns:** `PaginatedResponse<NftViewWithStatus>`. Each item has the standard `NftView` fields plus `status`.

**`NftStatus`:**

| Variant       | Meaning                                                                          |
| ------------- | -------------------------------------------------------------------------------- |
| `Minted`      | From `owner`'s collection, currently held by `owner`.                            |
| `Transferred` | From `owner`'s collection, currently held by another account.                    |
| `Received`    | Not from `owner`'s collection but currently held by `owner`.                     |

### `get_nfts_by_holder` (holder-account view)

Lists NFTs currently held by a specific `Account`, regardless of who minted them.

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai get_nfts_by_holder '(record {
  holder     = record { owner = principal "<holder_principal>"; subaccount = null };
  collection = null;
  pagination = record { offset = opt 0 : opt nat64; limit = opt 50 : opt nat64 }
})'
```

**Returns:** `PaginatedResponse<NftView>`. `timestamp_nanos` on each item is the `acquired_at` timestamp.

### `get_past_nfts_by_holder` (history)

Lists NFTs that `holder` previously owned and no longer holds (transferred away or burned).

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai get_past_nfts_by_holder '(record {
  holder     = record { owner = principal "<holder_principal>"; subaccount = null };
  pagination = record { offset = opt 0 : opt nat64; limit = opt 50 : opt nat64 }
})'
```

**Returns:** `PaginatedResponse<PastNftView>`. Each item:

| Field            | Description                                                                  |
| ---------------- | ---------------------------------------------------------------------------- |
| `acquired_at`    | Nanosecond timestamp when `holder` acquired the token                        |
| `released_at`    | Nanosecond timestamp when `holder` lost the token                            |
| `release_reason` | `variant { Transfer = record { to : Account } }` or `variant { Burn }`       |
| `current_owner`  | Whoever holds the token now (`null` if burned)                               |
| `metadata_json`  | JSON metadata string                                                         |

Use this for an owner's burn / transfer history view.

### `get_holders_by_collection`

Lists current holders of every token in a collection.

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai get_holders_by_collection '(record {
  collection = principal "<collection_canister_id>";
  pagination = record { offset = opt 0 : opt nat64; limit = opt 100 : opt nat64 }
})'
```

**Returns:** `PaginatedResponse<NftView>`.

### `get_account_stats`

Aggregate stats for an account across every indexed collection.

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai get_account_stats '(record {
  account = record { owner = principal "<account_principal>"; subaccount = null }
})'
```

| Field                  | Description                                                                              |
| ---------------------- | ---------------------------------------------------------------------------------------- |
| `total_owned`          | Total tokens currently owned across all collections                                      |
| `per_collection`       | `vec record { principal; nat64 }`: owned count per collection                            |
| `distinct_collections` | Number of distinct collections in which the account holds at least one token             |
| `first_activity_nanos` | First time this account was observed (`opt nat64`)                                       |
| `last_activity_nanos`  | Most recent activity (`opt nat64`)                                                       |

### `get_collection_stats`

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai get_collection_stats '(record {
  collection = principal "<collection_canister_id>"
})'
```

| Field                | Description                                                                          |
| -------------------- | ------------------------------------------------------------------------------------ |
| `distinct_holders`   | Number of distinct accounts holding at least one token in this collection            |
| `total_tokens`       | Total tokens of this collection currently tracked by the indexer                     |
| `next_indexed_block` | First ICRC-3 block id the indexer has not yet processed (use as a freshness probe)   |
