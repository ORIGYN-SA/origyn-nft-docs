---
icon: layer-group
metaLinks:
  alternates:
    - https://app.gitbook.com/s/yE16Xb3IemPxJWydtPOj/getting-started/collections-and-certificates
---

# Collections & Certificates

## Collections

A collection is an ORIGYN NFT canister that holds your certificates. Each collection is created from a [template](templates.md) and is managed by the Minting Studio. The Minting Studio handles canister creation, cycle management, and infrastructure — you focus on defining your template and minting certificates.

### Collection Lifecycle

When you create a collection, it progresses through these states:

```
Queued → Created → Installed → TemplateUploaded → Ready for minting
```

| Status | Description |
|--------|-------------|
| **Queued** | Request received, waiting to be processed |
| **Created** | Canister created on the Internet Computer |
| **Installed** | ORIGYN NFT WASM code installed on the canister |
| **TemplateUploaded** | Template metadata uploaded to the collection — ready for minting |
| **Failed** | Something went wrong (see error reason). Automatic refund is initiated |

**Failure path:** If collection creation fails at any stage, the system automatically reimburses your 15,000 OGY fee:

```
Failed → ReimbursingQueued → Reimbursed
```

In rare cases where automatic reimbursement encounters an issue, the status becomes `QuarantinedReimbursement` and requires manual resolution.

***

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

**Returns:** `collection_id` (nat) — use this to monitor the collection's status.

**Parameters:**

| Field | Type | Description |
|-------|------|-------------|
| `name` | text | Collection display name |
| `description` | text | Description of the collection |
| `symbol` | text | Short symbol (e.g., "GBC") |
| `template_id` | nat | ID of a previously registered template |

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

**Returns:** `CollectionInfo` record with:

| Field | Description |
|-------|-------------|
| `collection_id` | Unique collection identifier |
| `status` | Current lifecycle status |
| `canister_id` | The ORIGYN NFT canister principal (available after creation) |
| `metadata` | Collection name, symbol, description, and template_id |
| `owner` | Principal of the collection creator |
| `ogy_charged` | OGY tokens charged for creation |
| `created_at` | Creation timestamp |

#### List Your Collections

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai get_collections_by_owner '(record {
  owner = principal "<your_principal>";
  pagination = record { offset = opt 0; limit = opt 10 }
})'
```

**Returns:** `CollectionsResult` with a vector of `CollectionInfo` and `total_count`.

***

## Certificates (NFTs)

Certificates are the individual ORIGYN NFTs within a collection. Each certificate contains metadata structured according to the collection's template. They implement the ICRC-7 standard and can be transferred, approved, and queried using standard ICRC-7/ICRC-37 methods (see [ICRC-37 / ICRC-7](../getting-started/icrc37-icrc7.md)).

### Viewing Certificate Details

Fetch the metadata for one or more certificates:

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai get_nft_details '(record {
  canister_id = principal "<collection_canister_id>";
  token_ids = vec { 1; 2; 3 }
})'
```

**Returns:** A vector of `NftDetails`, each containing:

| Field | Description |
|-------|-------------|
| `token_id` | The certificate's token ID within the collection |
| `owner` | Current owner (principal + optional subaccount) |
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

Use `prev` for pagination — pass the last token ID from the previous page to get the next set.

For the complete end-to-end flow from template design to certificate viewing, see [How It Works](../getting-started/how-it-works.md).
