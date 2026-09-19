---
icon: file-lock
---

# Minting Private Content

Minting a certificate with private content follows the normal [minting flow](../minting-studio/minting.md), with three additions: you reserve room for encryption, you upload private files through their own endpoints, and each certificate carries a `private` part next to its public JSON.

```
1. Estimate            GET  /estimate           (+ private_file_sizes)
2. Reserve and pay     POST /initialize_mint    (+ private_file_sizes)
3. Upload public files POST /init_upload, /store_chunk, /finalize_upload
4. Upload private files POST /private_uploads, .../chunks/{n}, .../finalize
5. Mint                POST /mint_json_nfts     (+ private part per item)
6. Settle              POST /close_mint_request
```

All paths are under `https://gateway.origyn.com/gateway/v1/nft/production/`, and every step is an HTTP call with your [API key](../rest-api/api-keys.md): private content works only through the HTTP API for now, and the canister's own `mint_json_nfts` refuses any certificate that carries a `private` block (see [Private Content](overview.md)). The collection must belong to an [organization](../minting-studio/organizations.md), and its template must mark the items you send privately as `"private": true` (see [Marking fields private](overview.md#marking-fields-private)).

## 1. Reserve room for encryption

Encryption makes every private file slightly larger: 16 bytes per 1,048,560-byte chunk. You pay storage for the encrypted size, so tell the gateway which files are private and it adds the overhead for you.

* Count each private file's **plaintext** size in the total, exactly like a public file.
* List those same plaintext sizes in `private_file_sizes`.

```bash
curl "https://gateway.origyn.com/gateway/v1/nft/production/estimate?num_mints=1&total_bytes=3355443&private_file_sizes=3355443" \
  -H "Authorization: Bearer $ORIGYN_API_KEY"
```

The estimate answers with an extra `encryption_overhead_bytes`.

```bash
curl -X POST https://gateway.origyn.com/gateway/v1/nft/production/initialize_mint \
  -H "Authorization: Bearer $ORIGYN_API_KEY" \
  -H "Idempotency-Key: $IDEMPOTENCY_KEY" \
  -H "Content-Type: application/json" \
  -d '{
        "collection_canister_id": "<collection_canister_id>",
        "num_mints": 1,
        "total_file_size_bytes": "3355443",
        "private_file_sizes": [3355443]
      }'
```

**Leave out `private_file_sizes` and the reservation is too small.** The upload is then refused at its last chunk with `413 byte_limit_exceeded`, after the batch has already been paid for.

## 2. Upload a private file

Private files have their own three endpoints. You send **plaintext**; the gateway encrypts each chunk before it reaches the canister.

### Declare the upload

```bash
curl -X POST https://gateway.origyn.com/gateway/v1/nft/production/private_uploads \
  -H "Authorization: Bearer $ORIGYN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "mint_request_id": 77, "file_size": 3355443 }'
```

```json
{ "upload_id": "p/0c4e9b7a5d1f4e2a9b3c8d7e6f5a4b3c", "chunk_size": 1048560 }
```

`file_size` is the plaintext size. There is no file name and no hash here: the gateway assigns the stored name, and you give the real name and the hash when you mint.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/private_uploads" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### Send the chunks

Split the file into chunks of exactly `chunk_size` bytes (the last one may be shorter) and send each as the raw request body. Chunk indexes start at `0`, and `mint_request_id` goes in the query string.

```bash
split -b 1048560 -a 3 -d lab_report.pdf chunk_

n=0
for chunk in chunk_*; do
  curl -X POST "https://gateway.origyn.com/gateway/v1/nft/production/private_uploads/p/0c4e9b7a5d1f4e2a9b3c8d7e6f5a4b3c/chunks/$n?mint_request_id=77" \
    -H "Authorization: Bearer $ORIGYN_API_KEY" \
    -H "Content-Type: application/octet-stream" \
    --data-binary "@$chunk"
  n=$((n + 1))
done
```

You can paste the whole `upload_id` into the path, slash included: `/private_uploads/p/0c4e.../chunks/0` is the same URL.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/private_uploads/p/{upload_hex}/chunks/{n}" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### Finalize

```bash
curl -X POST https://gateway.origyn.com/gateway/v1/nft/production/private_uploads/p/0c4e9b7a5d1f4e2a9b3c8d7e6f5a4b3c/finalize \
  -H "Authorization: Bearer $ORIGYN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "mint_request_id": 77 }'
```

```json
{ "file_url": "https://<collection_canister_id>.raw.icp0.io/77/p/0c4e9b7a5d1f4e2a9b3c8d7e6f5a4b3c" }
```

The URL serves ciphertext. Keep the `upload_id`; that is how the mint refers to the file.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/private_uploads/p/{upload_hex}/finalize" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

The public upload endpoints do not take private files. `POST /init_upload` answers `400 private_flag_removed` if the body has a `private` key, and any `file_path` starting with `p/` answers `400 reserved_path`.

### Reusing a private file

A certificate may reference a private file uploaded in an **earlier mint request of the same collection**, for example the same warranty document on many certificates. Reference it by its path (`<mint_request_id>/p/<hex>`) or by its `file_url`. A reused file needs no new storage, so leave it out of `total_file_size_bytes` and `private_file_sizes`.

One mint call may reuse files from at most 20 other mint requests.

## 3. Mint with a private part

Each item of `POST /mint_json_nfts` may carry `private` **next to** `json_metadata`, never inside it.

```json
{
  "mint_request_id": 77,
  "items": [
    {
      "owner": { "principal": "<recipient_principal>" },
      "json_metadata": "{\"name\":\"Diamond #A17\",\"data\":{\"serial\":\"A17\"}}",
      "private": {
        "fields": {
          "purchase_price": { "content": { "en": "61 200 CHF" } }
        },
        "files": {
          "lab_report": [
            {
              "upload_id": "p/0c4e9b7a5d1f4e2a9b3c8d7e6f5a4b3c",
              "name": "Lab report.pdf",
              "size": 3355443,
              "sha256": "<sha256 of the plaintext, hex>"
            }
          ]
        },
        "readers": {
          "lab_report": ["insurer", "owner"]
        }
      }
    }
  ]
}
```

```json
{ "token_ids": ["1"] }
```

| Key | Content |
| --- | ------- |
| `fields` | Values of private items, keyed by item id, in the same [value shapes](../minting-studio/minting.md#value-shapes-inside-data) as `data` |
| `files` | Files of private items, keyed by item id. Every entry needs `upload_id`, `name` (the real file name), `size` (plaintext bytes) and `sha256` (hex of the plaintext) |
| `readers` | Optional. Per-certificate reader overrides, see below |

All three are optional. The gateway validates the part, encrypts it and writes it into the certificate. The Minting Studio canister never sees the plaintext.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/mint_json_nfts" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### Per-certificate readers

`readers` replaces the template's reader list for that item, on this certificate only. Use it when one certificate needs a different audience from the rest, for example a specific insurer.

{% hint style="danger" %}
**Overrides are fixed forever.** Nothing can change a certificate's `readers` after it is minted. Group names must be 1 to 32 characters of `a-z`, `0-9` and `-`, but whether the group exists is **not** checked, so a typo permanently locks that item to organization members only.
{% endhint %}

### Validation rules

The gateway checks every item, including items that send no private part. Each rule answers `400` with the `error` code shown.

| `error` | Cause |
| ------- | ----- |
| `private_in_data` | A private item's value appears in the public JSON, under `data` or as a top-level string. Private values must go only in `private`. |
| `missing_private` | A private item marked `required` has no value in `fields` and no file in `files`. An empty string counts as a value; only a missing or `null` value fails. |
| `not_private` | `fields`, `files` or `readers` names an item that is not private in the certificate's template version. |
| `private_not_allowed` | `json_metadata` has its own top-level `private` key. Only the gateway writes that key. |
| `invalid_reader` | A `readers` name is not a valid group id. |
| `not_a_private_upload` | An `upload_id` does not name a private upload of this collection. |
| `upload_not_finalized` | The referenced upload was never finalized. |
| `private_file_from_another_collection` | The file belongs to a different collection. |
| `too_many_reused_mint_requests` | Files are reused from more than 20 other mint requests. |

The certificate plus its private part must stay within the 50 KiB per-item limit, otherwise `413 json_too_large`.

**These checks run at mint time, after `initialize_mint` has charged you.** Validate your private parts against the template before you reserve and pay.

**Template versions and private items.** Whether an item is private comes from the template version the certificate is minted with (the newest, unless your JSON pins another with `"template": { "id": ..., "version": ... }`). An item that only a newer version makes private can go neither in the public JSON nor in the private part while the certificate pins an older version. Mint against the newest version to set it.

## Upload errors

| Status | `error` | Meaning |
| ------ | ------- | ------- |
| `400` | `invalid_upload_id`, `invalid_chunk_index` | Malformed id or chunk index in the path |
| `400` | `not_an_org_collection` | The collection does not belong to an organization |
| `403` | `not_owner`, `not_a_member`, `org_suspended` | You may not upload into this mint request, or the organization is suspended |
| `404` | `not_found` | Unknown mint request |
| `409` | `mint_not_active` | The mint request is already settled |
| `413` | `byte_limit_exceeded` | The mint request did not reserve enough bytes |
| `413` | `chunk_too_large` | A chunk is larger than `chunk_size` |
| `502` | `vetkd_unavailable`, `key_unavailable`, `upload_error`, `canister_unavailable` | Temporary upstream failure; retry |
