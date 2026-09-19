---
icon: hammer
metaLinks:
  alternates:
    - https://app.gitbook.com/s/yE16Xb3IemPxJWydtPOj/getting-started/minting
---

# Minting

Minting creates certificates inside a collection. Every example below is shown over HTTP first, with the equivalent `dfx` call beside it.

**Before you start**

* A collection in `TemplateUploaded` status (see [Collections & Certificates](collections-and-certificates.md)).
* The Owner, Admin or Minter role in the collection's [organization](organizations.md).
* An OGY allowance approved by the organization's billing principal, which pays the fees. See [Pricing](../core-concepts/pricing.md).

Mint requests are shared: any colleague with minting rights can upload to, mint from, close or refund a request you opened.

The whole flow is five calls:

```
1. estimate            what it will cost          free
2. initialize_mint     reserve and pay            charges OGY
3. upload files        init → chunks → finalize   per file
4. mint_json_nfts      create the certificates    up to 50 per call
5. close_mint_request  settle and get the rest back
```

Set these once and the examples below run as written:

```bash
export ORIGYN_API_KEY="sk_live_..."
export API="https://gateway.origyn.com/gateway/v1/nft/production"
export COLLECTION="<collection_canister_id>"
```

---

## Step 1: Estimate the cost

Free, and charges nothing. `total_bytes` is the total size of the files you will upload, as a decimal string.

{% tabs %}
{% tab title="HTTP" %}
```bash
curl "$API/estimate?num_mints=10&total_bytes=5000000" \
  -H "Authorization: Bearer $ORIGYN_API_KEY"
```

```json
{
  "total_ogy_e8s": "66663099337",
  "total_usd_e8s": "92015991",
  "ogy_usd_price_e8s": "140792",
  "breakdown": { "base_fee_usd_e8s": "10000000", "storage_fee_usd_e8s": "82015991" }
}
```

That is 666.63 OGY for ten certificates and 5 MB of files, about $0.92 at the rate in `ogy_usd_price_e8s`.
{% endtab %}

{% tab title="dfx" %}
```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai estimate_mint_cost '(record {
  num_mints = 10 : nat64;
  total_file_size_bytes = 5000000 : nat
})'
```
{% endtab %}
{% endtabs %}

All amounts are e8s, so divide by 100,000,000. See [Pricing](../core-concepts/pricing.md) for what drives the number, and [Minting Private Content](../private-content/minting.md#1-reserve-room-for-encryption) if the batch includes private files.

If the OGY price oracle is briefly unavailable you get `OgyPriceNotAvailable`; retry shortly.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/estimate" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

---

## Step 2: Reserve capacity and pay

This charges OGY to the organization's billing principal and reserves two things: how many certificates you may mint, and how many bytes you may upload. Both are refunded in Step 5 to the extent you do not use them.

{% tabs %}
{% tab title="HTTP" %}
```bash
curl -X POST "$API/initialize_mint" \
  -H "Authorization: Bearer $ORIGYN_API_KEY" \
  -H "Idempotency-Key: $(uuidgen)" \
  -H "Content-Type: application/json" \
  -d "{
        \"collection_canister_id\": \"$COLLECTION\",
        \"num_mints\": 10,
        \"total_file_size_bytes\": \"5000000\"
      }"
```

```json
{ "mint_request_id": "77", "status": "Initialized" }
```

`total_file_size_bytes` is a **decimal string**, not a number. `Idempotency-Key` is required; see [Paid Requests](../rest-api/paid-requests.md).
{% endtab %}

{% tab title="dfx" %}
```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai initialize_mint '(record {
  collection_canister_id = principal "<collection_canister_id>";
  num_mints = 10 : nat64;
  total_file_size_bytes = 5000000 : nat
})'
```
{% endtab %}
{% endtabs %}

Save the id, every later call needs it:

```bash
export MINT_REQUEST_ID=77
```

Size the byte reservation generously. Uploads stop the moment you exceed it, and the unused part comes back when you settle. `num_mints = 0` is valid and opens an upload-only session.

**Errors:** `CollectionNotReady` (the collection is not `TemplateUploaded`), `CallerNotCollectionOwner` (no minting rights in the organization, or it is suspended; `403 not_owner` over HTTP), `TransferFromError` (the billing principal lacks balance or approval).

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/initialize_mint" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

---

## Step 3: Upload files

Three calls per file: declare it, send the bytes, finalize. A chunk can be at most **1 MiB**, and a file at most **100 MiB**.

Private files use different endpoints; see [Minting Private Content](../private-content/minting.md).

### A. Declare the file

{% tabs %}
{% tab title="HTTP" %}
```bash
curl -X POST "$API/init_upload" \
  -H "Authorization: Bearer $ORIGYN_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{
        \"mint_request_id\": $MINT_REQUEST_ID,
        \"file_path\": \"gold_bar_001.png\",
        \"file_size\": 500000,
        \"file_hash\": \"$(shasum -a 256 gold_bar_001.png | cut -d' ' -f1)\"
      }"
```

```json
{ "file_path": "gold_bar_001.png", "chunk_size": 1048576 }
```

`file_hash` (SHA-256, hex) is required over HTTP and is checked when you finalize.
{% endtab %}

{% tab title="dfx" %}
```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai proxy_init_upload '(record {
  mint_request_id = 77 : nat64;
  file_path = "gold_bar_001.png";
  file_size = 500000 : nat64;
  file_hash = opt "<sha256_hex>";
  chunk_size = null
})'
```

Over `dfx` the hash is optional: pass `null` to skip the check.
{% endtab %}
{% endtabs %}

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/init_upload" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### B. Send the bytes

The chunk goes in the request body as raw bytes. Everything else goes in the query string.

{% tabs %}
{% tab title="HTTP" %}
```bash
split -b 1048576 -a 3 -d gold_bar_001.png chunk_

n=0
for chunk in chunk_*; do
  curl -X POST "$API/store_chunk?mint_request_id=$MINT_REQUEST_ID&file_path=gold_bar_001.png&chunk_id=$n" \
    -H "Authorization: Bearer $ORIGYN_API_KEY" \
    -H "Content-Type: application/octet-stream" \
    --data-binary "@$chunk"
  n=$((n + 1))
done
```

A file under 1 MiB is a single chunk with `chunk_id=0`.
{% endtab %}

{% tab title="dfx" %}
```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai proxy_store_chunk '(record {
  mint_request_id = 77 : nat64;
  file_path = "gold_bar_001.png";
  chunk_id = 0 : nat;
  chunk_data = blob "...binary_data..."
})'
```
{% endtab %}
{% endtabs %}

{% hint style="warning" %}
**Only re-send a chunk that returned an error.** Bytes are counted per successful call, not per `chunk_id`, so re-sending a chunk that already worked charges you for it twice.
{% endhint %}

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/store_chunk" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### C. Finalize

{% tabs %}
{% tab title="HTTP" %}
```bash
curl -X POST "$API/finalize_upload" \
  -H "Authorization: Bearer $ORIGYN_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{ \"mint_request_id\": $MINT_REQUEST_ID, \"file_path\": \"gold_bar_001.png\" }"
```

```json
{ "file_url": "https://<collection_canister_id>.raw.icp0.io/77/gold_bar_001.png" }
```
{% endtab %}

{% tab title="dfx" %}
```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai proxy_finalize_upload '(record {
  mint_request_id = 77 : nat64;
  file_path = "gold_bar_001.png"
})'
```
{% endtab %}
{% endtabs %}

Keep `file_url`. It is what you put in `path` when the certificate references the file, and uploads are namespaced by mint request, so a bare filename will not render.

Files are served by the collection's storage canister: the URL redirects (`307`) to it, so follow redirects. Range requests are answered with `206`, at most 2 MiB per response.

**Errors:** `ByteLimitExceeded` (past the bytes you reserved), `Unauthorized` (no minting rights, or the organization is suspended).

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/finalize_upload" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

---

## Step 4: Mint the certificates

Each item mints one certificate from a JSON string validated against the collection's template. Up to **50 items** per call, and you can call it repeatedly with the same `mint_request_id` until `num_mints` runs out.

{% tabs %}
{% tab title="HTTP" %}
```bash
curl -X POST "$API/mint_json_nfts" \
  -H "Authorization: Bearer $ORIGYN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
        "mint_request_id": 77,
        "items": [
          {
            "owner": { "principal": "<recipient_principal>" },
            "json_metadata": "{\"name\":\"Gold Bar #001\",\"data\":{\"serial_number\":\"GB-2026-001\"}}",
            "public_content": [
              { "name": "certificate_image", "file_path": "gold_bar_001.png" }
            ]
          }
        ]
      }'
```

```json
{ "token_ids": ["1"] }
```
{% endtab %}

{% tab title="dfx" %}
```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai mint_json_nfts '(record {
  mint_request_id = 77 : nat64;
  mint_items = vec {
    record {
      token_owner = record { owner = principal "<recipient_principal>"; subaccount = null };
      json_metadata = "<the JSON string, see below>";
      memo = null
    }
  }
})'
```

The field names differ from HTTP: `mint_items` and `token_owner` here, `items` and `owner` over HTTP.
{% endtab %}
{% endtabs %}

Over HTTP keep each request under about **2 MB** in total, which is the body limit. That is roughly 40 items, whatever the per-item limit allows. A bigger body is rejected as a plain `413` with no JSON error.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/mint_json_nfts" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### Attaching uploaded files

`public_content` attaches files from this mint request to the certificate, as `name` and `file_path` pairs. `file_path` is the name you gave `init_upload`; both `gold_bar_001.png` and `77/gold_bar_001.png` resolve to the same file, and `GET /mint_requests/{id}` lists the exact stored paths.

This is separate from the file references inside `data`, which point at a file by URL. Most certificates set both, pointing at the same upload.

**Errors:** `MintRequestNotFound`, `Unauthorized`, `MintRequestNotActive` (settled or refunded), `UnauthorizedFile { file_path }` (`403 file_not_uploaded`: the file was not uploaded to this collection), `MintLimitExceeded`, `NoItemsProvided`, `TooManyItems` (over 50), `JsonTooLarge` (an item over 50 KiB), `BrokenJsonMetadata`, `InvalidMetadata` (see [validation](#what-validation-enforces)), `MintError`.

---

## Writing the certificate JSON

`json_metadata` is a JSON object with a fixed envelope: display keys at the top level, template fields under `data`.

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

Stringify it and pass it as `json_metadata`.

### Value shapes inside `data`

| Field kind | Shape | Example |
| ---------- | ----- | ------- |
| Plain text | a bare JSON string | `"serial_number": "GB-2026-001"` |
| Localized text | an object with `content` mapping language codes to strings | `"name": { "content": { "en": "Gold Bar", "fr": "Lingot d'or" } }` |
| File-bearing (`image`, `video`, `document`, `signature`) | an array of file references | `"certificate_image": [{ "id": "img_1", "path": "https://..." }]` |

Two rules catch people out. A plain text field is a bare string, so `{ "content": "GB-2026-001" }` does not count as a value for a required field. And `path` is the **`file_url` that finalize returned**, not the `file_path` you sent, because uploads are namespaced by mint request.

### Reserved top-level keys

* **`template`** pins the [template version](templates.md#how-a-certificate-pins-its-version) the certificate is validated against. Leave it out and the current version is recorded for you.
* **`private`** holds encrypted [private content](../private-content/overview.md) and is written only by the gateway. Sending your own is refused with `InvalidMetadata`.

### Reserved field IDs

The standard viewer looks for these ids when it renders a certificate:

| Purpose | Field IDs (priority order) |
| ------- | -------------------------- |
| Certificate title | `name`, `company_name`, `certificate_title` |
| Certificate image | `certificate_image` (file reference) or `stamp_upload` (string URL) |
| Description | `description`, `short_description` |
| Company logo (header) | `company_logo` |
| Issuer ("Certified by") | `certified_by` |

### What validation enforces

Validation is lenient. Extra keys and unknown field types pass. A certificate is rejected with `InvalidMetadata` only when a `required` field has no non-empty value, when the `template` pin names another template or a version that does not exist, or when the JSON carries its own top-level `private` key. Fields marked `private` are checked by the gateway instead, when you [mint private content](../private-content/minting.md#validation-rules).

So a certificate can mint and still render incompletely. Check your first one in the viewer before minting the rest.

---

## Step 5: Check status, then settle

{% tabs %}
{% tab title="HTTP" %}
```bash
curl "$API/mint_requests/$MINT_REQUEST_ID" \
  -H "Authorization: Bearer $ORIGYN_API_KEY"
```
{% endtab %}

{% tab title="dfx" %}
```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai get_mint_request '(77 : nat64)'
```
{% endtab %}
{% endtabs %}

The response reports `status`, `minted_count` against `num_mints`, `bytes_uploaded` against `allocated_bytes`, `uploaded_files`, `ogy_charged`, and the member who opened the request. `GET /mint_requests` lists only the requests you opened; `get_mint_requests_by_org` on the canister lists the whole organization's.

| Status | Meaning |
| ------ | ------- |
| `Initialized` | Open for uploads and minting. Minting everything leaves it here. |
| `Completed` | Settled, with nothing left to refund |
| `RefundRequested` | Settled, refund queued |
| `Refunded` | Refund paid |
| `RefundFailed` | Refund attempt failed, see the reason |

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/mint_requests/{id}" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### Settling

**Minting everything does not close the request.** Settling is when money moves: what you used is burned, and the unused part of both reservations goes back to the billing principal, minus the ledger transfer fee. Residues at or below that fee are burned instead of paid out.

There are two ways to end a request, and which one applies depends on whether you have used it at all.

| Situation | Call | Result |
| --------- | ---- | ------ |
| You minted or uploaded anything | `close_mint_request` | Unused capacity and unused storage refunded |
| You touched nothing at all | `request_mint_refund` | The whole amount back, all or nothing |
| You forgot | nothing | The hourly sweep settles anything idle for 24 hours, on the same terms as `close_mint_request` |

{% tabs %}
{% tab title="HTTP" %}
```bash
curl -X POST "$API/close_mint_request" \
  -H "Authorization: Bearer $ORIGYN_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{ \"mint_request_id\": $MINT_REQUEST_ID }"
```

Answers `202`; the refund is paid asynchronously.
{% endtab %}

{% tab title="dfx" %}
```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai close_mint_request '(record {
  mint_request_id = 77 : nat64
})'
```
{% endtab %}
{% endtabs %}

{% hint style="warning" %}
`request_mint_refund` works only while `minted_count` and `bytes_uploaded` are both zero. One token or one byte makes it permanently unavailable (`CreditsAlreadyUsed`), and `close_mint_request` becomes your only route to a refund.
{% endhint %}

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/close_mint_request" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/request_mint_refund" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

---

## A whole batch, end to end

This mints one certificate per image in a folder, in batches of 40, and settles at the end. It is the five steps above in one script.

```bash
#!/usr/bin/env bash
set -euo pipefail

API="https://gateway.origyn.com/gateway/v1/nft/production"
COLLECTION="<collection_canister_id>"
RECIPIENT="<recipient_principal>"
IMAGES=(images/*.png)

auth=(-H "Authorization: Bearer $ORIGYN_API_KEY")
json=(-H "Content-Type: application/json")

# 1. Reserve: one mint per image, plus the bytes they take.
total_bytes=$(du -cb "${IMAGES[@]}" | tail -1 | cut -f1)
mint_request_id=$(curl -s -X POST "$API/initialize_mint" "${auth[@]}" "${json[@]}" \
  -H "Idempotency-Key: batch-$(date +%Y%m%d)-1" \
  -d "{\"collection_canister_id\":\"$COLLECTION\",\"num_mints\":${#IMAGES[@]},\"total_file_size_bytes\":\"$total_bytes\"}" \
  | jq -r .mint_request_id)

# 2. Upload each image: declare, send (one chunk per MiB), finalize.
declare -A url_of
for image in "${IMAGES[@]}"; do
  name=$(basename "$image")
  size=$(wc -c < "$image")
  hash=$(shasum -a 256 "$image" | cut -d' ' -f1)

  curl -s -X POST "$API/init_upload" "${auth[@]}" "${json[@]}" \
    -d "{\"mint_request_id\":$mint_request_id,\"file_path\":\"$name\",\"file_size\":$size,\"file_hash\":\"$hash\"}" > /dev/null

  split -b 1048576 -a 3 -d "$image" "/tmp/$name.chunk_"
  n=0
  for chunk in "/tmp/$name.chunk_"*; do
    curl -s -X POST "$API/store_chunk?mint_request_id=$mint_request_id&file_path=$name&chunk_id=$n" \
      "${auth[@]}" -H "Content-Type: application/octet-stream" --data-binary "@$chunk" > /dev/null
    n=$((n + 1))
  done
  rm -f "/tmp/$name.chunk_"*

  url_of[$name]=$(curl -s -X POST "$API/finalize_upload" "${auth[@]}" "${json[@]}" \
    -d "{\"mint_request_id\":$mint_request_id,\"file_path\":\"$name\"}" | jq -r .file_url)
done

# 3. Mint in batches of 40, keeping each request under the 2 MB body limit.
batch=()
mint_batch() {
  [ ${#batch[@]} -eq 0 ] && return
  printf '%s\n' "${batch[@]}" | jq -s \
    --argjson id "$mint_request_id" '{mint_request_id: $id, items: .}' \
    | curl -s -X POST "$API/mint_json_nfts" "${auth[@]}" "${json[@]}" -d @- | jq -r '.token_ids[]'
  batch=()
}

for image in "${IMAGES[@]}"; do
  name=$(basename "$image")
  serial="${name%.*}"
  metadata=$(jq -nc --arg serial "$serial" --arg url "${url_of[$name]}" \
    '{name: $serial, image: $url, data: {serial_number: $serial, certificate_image: [{id: "img_1", path: $url}]}}')
  batch+=("$(jq -nc --arg owner "$RECIPIENT" --arg meta "$metadata" --arg name "$name" \
    '{owner: {principal: $owner}, json_metadata: $meta, public_content: [{name: "certificate_image", file_path: $name}]}')")
  [ ${#batch[@]} -eq 40 ] && mint_batch
done
mint_batch

# 4. Settle: refunds the capacity and storage you did not use.
curl -s -X POST "$API/close_mint_request" "${auth[@]}" "${json[@]}" \
  -d "{\"mint_request_id\":$mint_request_id}"
```

Reuse the same `Idempotency-Key` if you retry the reservation, and a new one only for a genuinely new batch. See [Paid Requests](../rest-api/paid-requests.md).

---

## Burning certificates

The **token owner** can destroy a certificate by calling the collection canister directly. This cannot be undone.

```bash
dfx canister call <collection_canister_id> burn_nft '(1 : nat)' --network ic
```

**Errors:** `NotTokenOwner`, `TokenDoesNotExist`, `ConcurrentManagementCall` (another management call is in flight; retry). Burns are recorded in the collection's ICRC-3 history as `7burn` transactions.

## `mint_nfts` is gone

The old `mint_nfts` method is a rejecting stub: every call fails, telling you to use `mint_json_nfts`. It remains in the interface only so generated bindings keep compiling. Certificates are now stored as a single JSON entry, and multi-language values live in the mint JSON above.
