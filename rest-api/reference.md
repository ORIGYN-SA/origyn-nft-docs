---
icon: list-tree
---

# Endpoint Reference

Every read endpoint, live. These need no API key, so you can call any of them right here with **Test it**.

{% hint style="info" %}
The `env` field is prefilled with `production`. Leave it as it is.
{% endhint %}

## Collections

### `GET /collections`

GET /v1/nft/{env}/collections?category=&limit=&offset=

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/collections" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### `GET /collections/count`

GET /v1/nft/{env}/collections/count

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/collections/count" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### `GET /collections/{canister_id}`

GET /v1/nft/{env}/collections/{canister_id}

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/collections/{canister_id}" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### `GET /collections/{canister_id}/stats`

GET /v1/nft/{env}/collections/{canister_id}/stats

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/collections/{canister_id}/stats" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### `GET /collections/{canister_id}/holders`

GET /v1/nft/{env}/collections/{canister_id}/holders?limit=&offset=

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/collections/{canister_id}/holders" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### `GET /collections/{canister_id}/template`

GET /v1/nft/{env}/collections/{canister_id}/template

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/collections/{canister_id}/template" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### `GET /collections/{canister_id}/token-ids`

GET /v1/nft/collections/{canister_id}/token-ids?prev=&take= -> live icrc7_tokens passthrough.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/collections/{canister_id}/token-ids" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

## Certificates

### `GET /nfts`

GET /v1/nft/{env}/nfts?category=&collection=&sort=minted_at|random&order=asc|desc&limit=&offset=

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/nfts" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### `GET /collections/{canister_id}/nfts`

GET /v1/nft/collections/{canister_id}/nfts?ids=1,2,3 -> live batch (max 100).

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/collections/{canister_id}/nfts" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### `GET /collections/{canister_id}/nfts/{token_id}`

GET /v1/nft/collections/{canister_id}/nfts/{token_id} -> live metadata + current owner.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/collections/{canister_id}/nfts/{token_id}" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### `GET /collections/{canister_id}/tokens/{token_id}`

GET /v1/nft/{env}/collections/{canister_id}/tokens/{token_id}

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/collections/{canister_id}/tokens/{token_id}" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

## Accounts and owners

### `GET /accounts/{principal}/nfts`

GET /v1/nft/{env}/accounts/{principal}/nfts?collection=&limit=&offset=

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/accounts/{principal}/nfts" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### `GET /accounts/{principal}/past-nfts`

GET /v1/nft/{env}/accounts/{principal}/past-nfts?limit=&offset=

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/accounts/{principal}/past-nfts" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### `GET /accounts/{principal}/collections`

GET /v1/nft/{env}/accounts/{principal}/collections?limit=&offset=

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/accounts/{principal}/collections" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### `GET /accounts/{principal}/stats`

GET /v1/nft/{env}/accounts/{principal}/stats

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/accounts/{principal}/stats" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### `GET /owners/{principal}/nfts`

GET /v1/nft/{env}/owners/{principal}/nfts?limit=&offset=

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/owners/{principal}/nfts" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### `GET /owners/{principal}/collections`

GET /v1/nft/{env}/owners/{principal}/collections?limit=&offset=

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/owners/{principal}/collections" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### `GET /owners/{principal}/templates`

GET /v1/nft/{env}/owners/{principal}/templates -> ALL of an owner's templates, live from the
studio (the API pages through the studio's 2-per-call limit internally).

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/owners/{principal}/templates" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

## Search, categories and templates

### `GET /search`

GET /v1/nft/{env}/search?q=&category=&type=&limit=

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/search" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### `GET /categories`

GET /v1/nft/{env}/categories

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/categories" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### `GET /categories/catalog`

GET /v1/nft/{env}/categories/catalog

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/categories/catalog" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### `GET /templates/{template_id}`

GET /v1/nft/{env}/templates/{template_id} -> one template, live from the studio.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/templates/{template_id}" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

## Transactions and freshness

### `GET /transactions`

GET /v1/nft/{env}/transactions

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/transactions" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### `GET /sync-status`

GET /v1/nft/{env}/sync-status -> minting-studio sync health for the environment.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/sync-status" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

## Write endpoints

These require an API key and are documented with worked examples on their own pages:

* [Obtaining an API Key](api-keys.md) covers the key lifecycle endpoints.
* [Paid Requests & Idempotency](paid-requests.md) covers `create_collection` and `initialize_mint`.
* [Managing Collections](../minting-studio/managing-collections.md) covers metadata, logo, and settlement.
* [Minting](../minting-studio/minting.md) covers the upload and mint flow.

### Templates

#### `POST /create_template`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/create_template" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `POST /update_template`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/update_template" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `POST /delete_template`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/delete_template" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `GET /templates`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/templates" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### Uploads

#### `POST /init_upload`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/init_upload" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `POST /store_chunk`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/store_chunk" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `POST /finalize_upload`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/finalize_upload" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### Mint sessions

#### `POST /mint_json_nfts`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/mint_json_nfts" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `GET /mint_requests`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/mint_requests" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `GET /mint_requests/{id}`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/mint_requests/{id}" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `POST /close_mint_request`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/close_mint_request" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `POST /request_mint_refund`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/request_mint_refund" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### Collections (keyed)

#### `POST /update_collection_metadata`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/update_collection_metadata" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `POST /upload_logo/{collection_canister_id}`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/upload_logo/{collection_canister_id}" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `GET /collections`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/collections" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `GET /nfts`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/nfts" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `GET /account/stats`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/account/stats" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

