---
icon: sliders
---

# Managing Collections

What you can change after a collection exists: its display metadata, its categories and its logo.

Editing takes the Owner, Admin or Minter role in the collection's [organization](organizations.md), and is refused while the organization is suspended.

Two things are fixed at creation and cannot be changed: the collection's **template** and its **certificate type** (`standard` or `dpp`). To change either, create a new collection.

The HTTP examples use the same shell variables as [Minting](minting.md): `$API`, `$ORIGYN_API_KEY` and `$COLLECTION`.

## Editing collection metadata

Change the name, description, symbol, logo or categories at any time. Every field except the collection id is optional, and what you leave out stays as it is.

{% tabs %}
{% tab title="HTTP" %}
```bash
curl -X POST "$API/update_collection_metadata" \
  -H "Authorization: Bearer $ORIGYN_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{
        \"collection_canister_id\": \"$COLLECTION\",
        \"name\": \"My Renamed Collection\",
        \"description\": \"An updated description\"
      }"
```
{% endtab %}

{% tab title="dfx" %}
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
{% endtab %}
{% endtabs %}

Sending `categories` **replaces** the whole list, so send the full set you want, or an empty list to clear it. Names are checked against the global taxonomy before anything is written: an unknown one fails the call with `UnknownCategory` (`unknown_category` over HTTP) and changes nothing else.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/update_collection_metadata" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### Finding valid category names

Categories are a curated list, so you cannot invent them. `/categories/catalog` is the full list with descriptions, including names nothing uses yet; `/categories` is only the names collections currently use. Over `dfx`, read them with `list_categories '(null)'`.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/categories/catalog" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

## Setting a collection logo

The logo is an image stored on the collection canister and referenced from its metadata. Over HTTP it is a single multipart upload; over `dfx` it is the usual three calls.

Re-uploading replaces the previous logo: unlike certificate files, logo paths are keyed by collection rather than by mint request.

{% tabs %}
{% tab title="HTTP" %}
```bash
curl -X POST "$API/upload_logo/$COLLECTION" \
  -H "Authorization: Bearer $ORIGYN_API_KEY" \
  -F "file=@logo.png"
```
{% endtab %}

{% tab title="dfx" %}
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

The logo upload still takes `file_hash` as a plain string, unlike certificate uploads where it is optional.
{% endtab %}
{% endtabs %}

A logo can be at most **5 MiB** (5,242,880 bytes). The HTTP endpoint accepts a body up to 25 MiB, so an oversized file is rejected by the canister after the upload rather than before it.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/upload_logo/{collection_canister_id}" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

## Settling a mint request

Settlement belongs to the minting flow. In short: minting everything does not close the request, `close_mint_request` settles it and refunds the capacity and storage you did not use, and the hourly sweep does it for you after 24 hours of inactivity.

See [Minting: check status, then settle](minting.md#step-5-check-status-then-settle).
