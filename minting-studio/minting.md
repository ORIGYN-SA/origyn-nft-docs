---
icon: hammer
metaLinks:
  alternates:
    - https://app.gitbook.com/s/yE16Xb3IemPxJWydtPOj/getting-started/minting
---

# Minting

Minting is the process of creating certificates (ORIGYN NFTs) within a collection. This guide covers the complete minting flow using the Minting Studio API.

### Prerequisites

* A collection in **TemplateUploaded** status (see [Collections & Certificates](collections-and-certificates.md))
* OGY tokens in your wallet for minting fees
* Certificate data ready (text fields, images, documents)

All commands below use the Minting Studio canister ID `uasjq-dyaaa-aaaas-qdwka-cai`.

***

## Step 1: Estimate Costs

Before minting, you can estimate the total cost:

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai estimate_mint_cost '(record {
  collection_canister_id = principal "<your_collection_canister_id>";
  num_mints = 10 : nat64;
  total_file_size_bytes = 5000000 : nat
})'
```

**Returns:** A `MintCostEstimate` with:

| Field | Description |
|-------|-------------|
| `total_ogy_e8s` | Total cost in OGY (e8s precision, divide by 100,000,000 for OGY) |
| `total_usd_e8s` | Total cost in USD equivalent |
| `ogy_usd_price_e8s` | Current OGY/USD exchange rate used |
| `breakdown.base_fee_usd_e8s` | Base fee component |
| `breakdown.storage_fee_usd_e8s` | Storage fee component (based on file sizes) |

**Errors:**
* `OgyPriceNotAvailable` — The OGY price oracle is temporarily unavailable. Try again shortly.
* `MintPricingNotConfigured` — Minting pricing has not been configured for this canister.

***

## Step 2: Initialize Mint Request

Create a mint request to reserve capacity and lock in the fee:

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai initialize_mint '(record {
  collection_canister_id = principal "<your_collection_canister_id>";
  num_mints = 10 : nat64;
  total_file_size_bytes = 5000000 : nat
})'
```

**Returns:** `mint_request_id` (nat64) — save this, you will need it for all subsequent steps.

This call will transfer OGY tokens from your wallet to cover the minting fee. Ensure you have approved the Minting Studio canister to spend from your OGY balance (via `icrc2_approve` as shown in [Getting Started](getting-started.md)).

**Errors:**
* `CollectionNotReady` — The collection is not in TemplateUploaded status.
* `CallerNotCollectionOwner` — You are not the owner of this collection.
* `InvalidNumMints` — num_mints must be greater than 0.
* `TransferFromError` — Insufficient OGY balance or approval.

***

## Step 3: Upload Files

If your certificates include images, documents, or other files, upload them before minting. File upload is a three-step process:

#### A. Initialize Upload

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai proxy_init_upload '(record {
  mint_request_id = <your_mint_request_id> : nat64;
  file_path = "certificate_image.png";
  file_size = 500000 : nat64;
  file_hash = "<sha256_hash_of_file>";
  chunk_size = null
})'
```

#### B. Store Chunks

For files larger than 2 MB, split them into chunks. Each chunk is uploaded separately:

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai proxy_store_chunk '(record {
  mint_request_id = <your_mint_request_id> : nat64;
  file_path = "certificate_image.png";
  chunk_id = 0 : nat;
  chunk_data = blob "...binary_data..."
})'
```

Repeat for each chunk, incrementing `chunk_id`.

#### C. Finalize Upload

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai proxy_finalize_upload '(record {
  mint_request_id = <your_mint_request_id> : nat64;
  file_path = "certificate_image.png"
})'
```

**Returns:** The public URL of the uploaded file (e.g., `https://<canister_id>.raw.icp0.io/certificate_image.png`).

**Errors:**
* `ByteLimitExceeded` — Total uploaded bytes exceed the `total_file_size_bytes` specified in the mint request.
* `Unauthorized` — You are not the owner of this mint request.

***

## Step 4: Mint Certificates

With files uploaded, mint the certificates by providing metadata for each one:

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai mint_nfts '(record {
  mint_request_id = <your_mint_request_id> : nat64;
  mint_items = vec {
    record {
      token_owner = record {
        owner = principal "<recipient_principal>";
        subaccount = null
      };
      metadata = vec {
        record { "name"; variant { Text = "Gold Bar #001" } };
        record { "serial_number"; variant { Text = "GB-2026-001" } };
        record { "weight"; variant { Text = "1 oz" } };
        record { "image"; variant { Text = "https://<canister>.raw.icp0.io/certificate_image.png" } };
      };
      memo = null
    }
  }
})'
```

**Returns:** A vector of minted token IDs (nat).

You can mint in batches — call `mint_nfts` multiple times with the same `mint_request_id` until you reach the `num_mints` limit.

**Errors:**
* `MintLimitExceeded` — You have already minted the maximum number of tokens for this request.
* `TooManyItems` — Too many items in a single call. Reduce the batch size.
* `NoItemsProvided` — The `mint_items` vector is empty.
* `MintRequestNotActive` — The mint request has been refunded or is no longer active.

***

## Step 5: Check Minting Status

Monitor the progress of your mint request:

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai get_mint_request '(<your_mint_request_id> : nat64)'
```

**Returns:** `MintRequestInfo` with:

| Field | Description |
|-------|-------------|
| `status` | Current status (see below) |
| `minted_count` | Number of tokens minted so far |
| `num_mints` | Total number of mints requested |
| `bytes_uploaded` | Total bytes uploaded |
| `allocated_bytes` | Total bytes allocated |
| `uploaded_files` | List of uploaded files with paths and URLs |
| `ogy_charged` | OGY tokens charged for this request |

**Mint Request Statuses:**

| Status | Description |
|--------|-------------|
| `Initialized` | Request created, ready for uploads and minting |
| `Completed` | All tokens have been minted |
| `RefundRequested` | A refund has been requested |
| `Refunded` | OGY tokens have been refunded |
| `RefundFailed` | Refund attempt failed (see reason) |

***

## Requesting a Refund

If you need to cancel a mint request before it is completed, you can request a refund of the OGY tokens:

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai request_mint_refund '(record {
  mint_request_id = <your_mint_request_id> : nat64
})'
```

**Note:** Refunds are only available for mint requests that have not been fully completed. Tokens already minted will remain, and the refund covers the unused portion.

**Errors:**
* `NotInRefundableState` — The request is already completed or refunded.
* `CreditsAlreadyUsed` — All minting credits have been used.

***

## Metadata Format (ICRC3)

Certificate metadata is stored as key-value pairs following the ICRC3 standard. Each pair consists of a text key and an `ICRC3Value`:

```
vec {
  record { "field_name"; variant { Text = "value" } };
  record { "numeric_field"; variant { Nat = 42 } };
  record { "image_field"; variant { Blob = blob "..." } };
  record { "gallery"; variant { Array = vec { variant { Text = "url1" }; variant { Text = "url2" } } } };
}
```

**Supported value types:**

| Type | Candid | Use Case |
|------|--------|----------|
| `Text` | `variant { Text = "..." }` | String values (names, descriptions, URLs) |
| `Nat` | `variant { Nat = 42 }` | Unsigned integers |
| `Int` | `variant { Int = -1 }` | Signed integers |
| `Blob` | `variant { Blob = blob "..." }` | Binary data (embedded images) |
| `Array` | `variant { Array = vec {...} }` | Lists of values (galleries, multi-select) |
| `Map` | `variant { Map = vec { record {...} } }` | Nested key-value pairs (localized content) |

### Multi-Language Metadata

For multi-language certificates, store localized values using the `Map` type:

```
record { "product_name"; variant {
  Map = vec {
    record { "en"; variant { Text = "Gold Bar" } };
    record { "fr"; variant { Text = "Lingot d'or" } };
    record { "it"; variant { Text = "Lingotto d'oro" } }
  }
} }
```
