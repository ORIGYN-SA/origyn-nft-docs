---
icon: hammer
metaLinks:
  alternates:
    - https://app.gitbook.com/s/yE16Xb3IemPxJWydtPOj/getting-started/minting
---

# Minting

Minting is the process of creating certificates (ORIGYN NFTs) within a collection. This guide covers the complete minting flow using the Minting Studio API.

### Prerequisites

- A collection in **TemplateUploaded** status (see [Collections & Certificates](collections-and-certificates.md))
- OGY tokens in your wallet for minting fees
- Certificate data ready (text fields, images, documents)

Unless noted otherwise, the `dfx` commands below use the Minting Studio canister ID `uasjq-dyaaa-aaaas-qdwka-cai`. The one exception is `burn_nft`, which is called on the collection canister directly.

{% hint style="info" %}
Every step here is also available over HTTP with an API key. See the [REST API Overview](../rest-api/overview.md).
{% endhint %}

---

## Step 1: Estimate Costs

Before minting, you can estimate the total cost:

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai estimate_mint_cost '(record {
  num_mints = 10 : nat64;
  total_file_size_bytes = 5000000 : nat
})'
```

**Returns:** A `MintCostEstimate` with:

| Field                           | Description                                                      |
| ------------------------------- | ---------------------------------------------------------------- |
| `total_ogy_e8s`                 | Total cost in OGY (e8s precision, divide by 100,000,000 for OGY) |
| `total_usd_e8s`                 | Total cost in USD equivalent                                     |
| `ogy_usd_price_e8s`             | Current OGY/USD exchange rate used                               |
| `breakdown.base_fee_usd_e8s`    | Base fee component                                               |
| `breakdown.storage_fee_usd_e8s` | Storage fee component (based on file sizes)                      |

**Errors:**

- `OgyPriceNotAvailable`:The OGY price oracle is temporarily unavailable. Try again shortly.
- `MintPricingNotConfigured`:Minting pricing has not been configured for this canister.

---

## Step 2: Initialize Mint Request

Create a mint request to reserve capacity and lock in the fee:

{% hint style="danger" %}
**This charges OGY.** Test it sends a real production request.
{% endhint %}

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/initialize_mint" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

**Using dfx instead**

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai initialize_mint '(record {
  collection_canister_id = principal "<your_collection_canister_id>";
  num_mints = 10 : nat64;
  total_file_size_bytes = 5000000 : nat
})'
```

Over REST, `total_file_size_bytes` is a **decimal string**, not a number.

**Returns:** `mint_request_id` (nat64). Save this, you will need it for all subsequent steps.

This call will transfer OGY tokens from your wallet to cover the minting fee. Ensure you have approved the Minting Studio canister to spend from your OGY balance (via `icrc2_approve` as shown in [Getting Started](getting-started.md)).

**Errors:**

- `CollectionNotReady`: The collection is not in TemplateUploaded status.
- `CallerNotCollectionOwner`: You are not the owner of this collection.
- `TransferFromError`: Insufficient OGY balance or approval.

{% hint style="info" %}
`num_mints = 0` is valid. It opens a **storage-only session**, which is how you reserve upload capacity without reserving any mints.
{% endhint %}

---

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

**Returns:** the public URL of the uploaded file. Upload paths are namespaced by the mint request, and the host comes from the collection canister, so the URL has the form:

```
https://<collection_canister_id>.raw.icp0.io/<mint_request_id>/certificate_image.png
```

Keep this returned URL. It is the value you put in `path` when you reference the file from your mint JSON.

**Errors:**

- `ByteLimitExceeded`: Total uploaded bytes exceed the `total_file_size_bytes` specified in the mint request.
- `Unauthorized`: You are not the owner of this mint request.

#### Over REST

The same three steps. Chunk bytes are sent as the raw request body.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/init_upload" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/store_chunk" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/finalize_upload" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

`finalize_upload` returns `{ "file_url": "..." }`. Chunk ids start at 0.

{% hint style="warning" %}
**Only re-send a chunk that returned an error.** Uploaded bytes are counted per successful `store_chunk` call, not per `chunk_id`, so re-sending a chunk that already succeeded counts its bytes twice against the storage you paid for at `initialize_mint`.
{% endhint %}

---

## Step 4: Mint Certificates

With files uploaded, mint the certificates by providing a JSON metadata string for each one. The Minting Studio validates the JSON server-side against the collection's template before minting.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/mint_json_nfts" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

**Using dfx instead**

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai mint_json_nfts '(record {
  mint_request_id = <your_mint_request_id> : nat64;
  mint_items = vec {
    record {
      token_owner = record {
        owner = principal "<recipient_principal>";
        subaccount = null
      };
      json_metadata = "<json string, see Producing your mint JSON below>";
      memo = null
    }
  }
})'
```

**Returns:** A vector of minted token IDs (nat), one per `mint_items` entry in order.

You can mint in batches or call `mint_json_nfts` multiple times with the same `mint_request_id` until you reach the `num_mints` limit.

**Errors (`MintJsonNftsError`):**

- `MintRequestNotFound`: No mint request exists for the given ID.
- `Unauthorized`: You are not the owner of this mint request.
- `MintRequestNotActive`: The mint request has been refunded or is no longer active.
- `MintLimitExceeded { allowed, already_minted, requested }`: This batch would exceed the request's `num_mints` cap.
- `NoItemsProvided`: The `mint_items` vector is empty.
- `TooManyItems { max }`: Batch is larger than the per-call limit.
- `JsonTooLarge { index, max, got }`: The `json_metadata` for the item at `index` exceeds the per-item byte cap.
- `BrokenJsonMetadata`: The `json_metadata` string is not valid JSON.
- `InvalidMetadata`: The JSON parsed but failed validation against the template (missing required field, wrong shape for a field, etc.).
- `MintError(text)`: Underlying mint call to the collection canister failed.

---

## Producing your mint JSON from a template

The `json_metadata` you pass to `mint_json_nfts` is a JSON object with a fixed outer envelope. Template field values go inside a `data` object; a handful of display keys sit at the top level.

```json
{
  "name":         "Gold Bar #001",
  "image":        "https://<collection_canister_id>.raw.icp0.io/<mint_request_id>/gold_bar_001.png",
  "description":  "1oz certified gold bar",
  "minted_at":    "2026-08-01T10:30:00Z",
  "certified_by": "ORIGYN",
  "data": {
    "<template_field_id>": <value>,
    ...
  }
}
```

The top-level keys drive how the certificate is displayed. Everything declared in your template belongs under `data`, keyed by field `id`.

### Value shapes inside `data`

Each field takes one of exactly three shapes.

| Field kind | Shape | Example |
| ---------- | ----- | ------- |
| Plain text | a bare JSON string | `"serial_number": "GB-2026-001"` |
| Localized text | an object with a `content` key mapping language codes to strings | `"name": { "content": { "en": "Gold Bar", "fr": "Lingot d'or" } }` |
| File-bearing (`image`, `video`, `document`, `signature`) | an array of file references | `"certificate_image": [{ "id": "img_1", "path": "https://..." }]` |

{% hint style="warning" %}
A plain string field is written as a **bare string**, not as `{ "content": "..." }`. Under `data`, a `content` key must map to an object of language codes; `{ "content": "GB-2026-001" }` is not accepted as a value for a required field.
{% endhint %}

{% hint style="warning" %}
`path` in a file reference is the **URL returned by `finalize_upload`**, not the `file_path` you passed in. Uploads are namespaced server-side by mint request, and the viewer uses this value directly as the media source, so a bare filename renders as a broken image.
{% endhint %}

### What validation actually enforces

Validation is deliberately lenient, which is worth knowing so you are not surprised in either direction:

* Extra keys the template does not declare are **ignored**, not rejected.
* Unknown field types in the template do not cause an error.
* The **only** failure mode is a field marked `required: true` (and not `immutable: true`) that has no non-empty value. That returns `InvalidMetadata`.

So a payload can be accepted and still render incompletely. Treat the template as the contract and check your output in the viewer.

### Reserved field IDs

To have the certificate render correctly in the standard ORIGYN viewer, use these IDs for fields that serve display roles:

| Purpose                  | Field IDs (priority order)                          |
| ------------------------ | --------------------------------------------------- |
| Certificate title        | `name`, `company_name`, `certificate_title`         |
| Certificate image        | `certificate_image` (file reference) or `stamp_upload` (string URL) |
| Description              | `description`, `short_description`                  |
| Company logo (header)    | `company_logo`                                      |
| Issuer ("Certified by")  | `certified_by`                                      |

### Worked example

Given a template with `name` (localized), `serial_number` (plain text), `certificate_image` (image) and `certified_by` (plain text), and assuming `finalize_upload` returned `https://abcde-fqaaa-aaaam-xyzab-cai.raw.icp0.io/77/gold_bar_001.png`:

```json
{
  "name": "Gold Bar #001",
  "image": "https://abcde-fqaaa-aaaam-xyzab-cai.raw.icp0.io/77/gold_bar_001.png",
  "description": "1oz certified gold bar",
  "certified_by": "ORIGYN",
  "data": {
    "name": { "content": { "en": "Gold Bar #001", "fr": "Lingot d'or #001" } },
    "serial_number": "GB-2026-001",
    "certificate_image": [
      { "id": "img_1", "path": "https://abcde-fqaaa-aaaam-xyzab-cai.raw.icp0.io/77/gold_bar_001.png" }
    ],
    "certified_by": "ORIGYN"
  }
}
```

Stringify that JSON and pass it as `json_metadata`.

---

## Step 5: Check Minting Status

Monitor the progress of your mint request:

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai get_mint_request '(<your_mint_request_id> : nat64)'
```

**Returns:** `MintRequestInfo` with:

| Field             | Description                                |
| ----------------- | ------------------------------------------ |
| `status`          | Current status (see below)                 |
| `minted_count`    | Number of tokens minted so far             |
| `num_mints`       | Total number of mints requested            |
| `bytes_uploaded`  | Total bytes uploaded                       |
| `allocated_bytes` | Total bytes allocated                      |
| `uploaded_files`  | List of uploaded files with paths and URLs |
| `ogy_charged`     | OGY tokens charged for this request        |

**Mint Request Statuses:**

| Status            | Description                                                                                  |
| ----------------- | -------------------------------------------------------------------------------------------- |
| `Initialized`     | Request created, ready for uploads and minting. **Minting every token leaves it here**        |
| `Completed`       | The request has been **settled** with nothing left to refund                                  |
| `RefundRequested` | Settled with a refund owed; payout is queued                                                  |
| `Refunded`        | OGY tokens have been refunded                                                                 |
| `RefundFailed`    | Refund attempt failed (see reason)                                                            |

{% hint style="warning" %}
**Minting all of your tokens does not close the request.** It stays `Initialized` until it is settled, either because you call `close_mint_request` or because it sits idle for 24 hours and the hourly sweep settles it for you.

Settlement is also when money moves: the OGY you actually used is burned and the unused portion of **both** reservations (unminted capacity and un-uploaded storage) is refunded, minus the ledger transfer fee.
{% endhint %}

---

## Closing a request and getting money back

There are two ways to end a mint request, and picking the wrong one is the most common way to lose access to a refund. **Which one applies depends entirely on whether you have used the request at all.**

### `close_mint_request` (the usual one)

Settles a request you have used. The OGY you consumed is burned, and the unused portion of **both** reservations is refunded: capacity you did not mint, and storage you did not upload.

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai close_mint_request '(record {
  mint_request_id = <your_mint_request_id> : nat64
})'
```

Use this whenever you have minted or uploaded anything, including a partially completed batch. The refund arrives asynchronously and is reduced by the ledger transfer fee; amounts at or below that fee are burned rather than paid out.

If you forget, the hourly sweep settles any request left idle for 24 hours on exactly the same terms.

### `request_mint_refund` (untouched requests only)

Refunds a request you have **not used at all**, all or nothing.

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai request_mint_refund '(record {
  mint_request_id = <your_mint_request_id> : nat64
})'
```

{% hint style="warning" %}
This works only while `minted_count` and `bytes_uploaded` are both zero. Minting a single token or uploading a single byte makes it permanently unavailable and it returns `CreditsAlreadyUsed`. At that point `close_mint_request` is your route to a refund.
{% endhint %}

**Errors:**

- `NotInRefundableState`: The request is already settled or refunded.
- `CreditsAlreadyUsed`: Something has been minted or uploaded. Use `close_mint_request` instead.

---

## Burning Certificates

To permanently destroy a certificate, the **token owner** can call `burn_nft` directly on the collection canister. This is an irreversible operation.

```bash
dfx canister call <collection_canister_id> burn_nft '(1 : nat)' --network ic
```

The argument is the `token_id` of the certificate to burn.

**Returns:** Empty `Ok` on success.

**Errors:**

- `NotTokenOwner`: Only the current owner of the token can burn it.
- `TokenDoesNotExist`: The specified token ID does not exist in this collection.
- `ConcurrentManagementCall`: Another management operation is in progress. Retry the command.

Burn events are recorded in the collection's ICRC-3 transaction history as `7burn` transactions.

---

## Deprecated: `mint_nfts`

{% hint style="danger" %}
**`mint_nfts` no longer mints anything.** It was reduced to a rejecting stub in release 1.6.0 and returns an error on every call:

```
mint_nfts is deprecated and no longer mints.
Use mint_json_nfts; certificates are stored as a single JSON_DATA entry.
```

Use [`mint_json_nfts`](#step-4-mint-certificates) instead. There is no migration window, because there is no working version of this endpoint to migrate away from.
{% endhint %}

The endpoint still appears in the canister interface so that the surface and its shared types stay stable, but no call path reaches a mint.

**What this means if you have older code or documentation:**

* The ICRC-3 structured metadata format that `mint_nfts` accepted (the `Text` / `Nat` / `Int` / `Blob` / `Array` / `Map` variant tree) is no longer a supported way to describe a certificate. Every certificate is now stored as a single `JSON_DATA` text entry.
* The read-side converter for that format was removed in the same release, so nothing produces or consumes it any more.
* Multi-language values are still fully supported; they are expressed in the mint JSON instead. See [Producing your mint JSON from a template](#producing-your-mint-json-from-a-template).

If you were building a metadata map by hand, replace it with a JSON object and send it through `mint_json_nfts`, which additionally validates your metadata against the collection's template before minting.
