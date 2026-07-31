---
icon: sliders
---

# Managing Collections

Operations you perform on a collection after it exists: editing its display metadata, setting its logo, and settling mint requests.

Each is shown both ways. Pick whichever track you are integrating with; they do the same thing.

{% hint style="info" %}
**`certificate_type` is not editable.** A collection's certificate type (`"standard"` or `"dpp"`) is
fixed when the collection is created; `update_collection_metadata` does not accept it. To change
type, create a new collection. See
[Collections & Certificates](collections-and-certificates.md#certificate-types).
{% endhint %}

## Editing collection metadata

Change a collection's name, description, symbol, logo, or categories at any time. Every field is optional; omitted fields are left alone.

**Using dfx instead**

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai update_collection_metadata '(record {
  collection_canister_id = principal "<collection_canister_id>";
  name = opt "My Renamed Collection";
  description = opt "An updated description";
  symbol = null;
  logo = null;
  categories = null
})'
```

Pass `null` for anything you are not changing.


{% hint style="warning" %}
Category names are validated against the global taxonomy **before** anything is written. An unknown name fails the whole call with `UnknownCategory` (`unknown_category` over REST) and changes nothing else.

Passing `categories` **replaces** the entire list rather than adding to it. Send the full set you want, and an empty list to clear them.
{% endhint %}

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/update_collection_metadata" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### Finding valid category names

Categories are a curated list maintained by ORIGYN, so you cannot invent them. Read the current set before tagging a collection:

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai list_categories '(null)'
```

Over HTTP, the categories actually in use are available publicly:

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/categories" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

{% hint style="info" %}
There are two related endpoints and they answer different questions. `/categories` returns the names **currently in use** by collections. `/categories/catalog` returns the full curated catalogue with descriptions, including names nothing uses yet.
{% endhint %}

## Setting a collection logo

The logo is an image stored on the collection canister and referenced from its metadata.

**Using dfx instead**

Three calls, mirroring the file upload flow:

```bash
# 1. Declare the file
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai proxy_logo_init_upload '(record {
  collection_canister_id = principal "<collection_canister_id>";
  file_path = "logo.png";
  file_size = 48000 : nat64;
  file_hash = "<sha256_hex>";
  chunk_size = null
})'

# 2. Send the bytes (repeat with chunk_id 0, 1, 2 ... for larger files)
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai proxy_logo_store_chunk '(record {
  collection_canister_id = principal "<collection_canister_id>";
  file_path = "logo.png";
  chunk_id = 0 : nat;
  chunk_data = blob "..."
})'

# 3. Finalize
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai proxy_logo_finalize_upload '(record {
  collection_canister_id = principal "<collection_canister_id>";
  file_path = "logo.png"
})'
```


{% hint style="warning" %}
**Maximum logo size is 5 MB.** The HTTP endpoint accepts a larger request body, but the canister rejects anything above 5 MB, so a bigger file fails after the upload rather than before it.
{% endhint %}

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/upload_logo/{collection_canister_id}" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

Unlike certificate uploads, logo paths are **not** namespaced per mint request; they are keyed by collection, so re-uploading replaces the previous logo.

## Settling a mint request

A mint request is a paid reservation: capacity to mint, and storage to upload into. Settling it burns the portion you used and refunds the rest.

**Minting all your certificates does not settle the request.** It stays `Initialized` until you close it, or until the hourly sweep closes it for you after 24 hours of inactivity.

**Using dfx instead**

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai close_mint_request '(record {
  mint_request_id = 77 : nat64
})'
```


{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/close_mint_request" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

What settlement does, precisely:

* The OGY corresponding to what you actually minted and uploaded is **burned**.
* The unused portion of **both** reservations is refunded: certificates you did not mint, and storage you did not fill.
* The refund is reduced by the ledger transfer fee. Amounts at or below that fee are burned instead of paid out.

{% hint style="info" %}
If you have not touched a request at all, `request_mint_refund` returns the whole amount in one step. It stops working the moment you mint or upload anything. See [Minting](minting.md#closing-a-request-and-getting-money-back).
{% endhint %}

### Checking a request before you settle

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/mint_requests/{id}" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

**Using dfx instead**

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai get_mint_request '(77 : nat64)'
```


Look at `minted_count` against `num_mints`, and `bytes_uploaded` against `allocated_bytes`, to see what you are about to reclaim.
