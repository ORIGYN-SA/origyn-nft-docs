---
icon: layer-group
metaLinks:
  alternates:
    - >-
      https://app.gitbook.com/s/yE16Xb3IemPxJWydtPOj/getting-started/collections-and-certificates
---

# Collections & Certificates

## Collections

A collection is an ORIGYN NFT canister that holds your certificates. Each collection belongs to an [organization](organizations.md), is created from one of that organization's [templates](templates.md), and is managed by the Minting Studio. The Minting Studio handles canister creation, cycle management, and infrastructure and you only focus on defining your template and minting certificates.

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

1. The **Owner** or **Admin** role in an [organization](organizations.md)
2. A template registered in that organization (see [Templates](templates.md))
3. An OGY spend allowance approved by the organization's billing principal (see [Getting Started](getting-started.md))

{% hint style="danger" %}
**This spends 15,000 OGY.** Test it sends a real production request.
{% endhint %}

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/create_collection" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

**Using dfx instead**

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai create_collection '(record {
  org_id = null;
  categories = vec {};
  name = "My Gold Bar Collection";
  description = "Certified gold bar certificates with full provenance tracking";
  symbol = "GBC";
  template_id = 1;
  certificate_type = opt "standard"
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
| `template_id` | nat      | ID of a template registered in the same organization                                                               |
| `org_id`      | opt nat64 | The organization that will own the collection. `null` means the organization you own. Pass it explicitly if you work in an organization you do not own. See [Choosing the organization you act for](organizations.md#choosing-the-organization-you-act-for). |
| `certificate_type` | opt text | Which kind of certificate this collection issues: `"standard"` (the default) or `"dpp"`. Case-insensitive. Pass `null` for standard. **Cannot be changed after creation.** |

The 15,000 OGY fee is charged to the organization's billing principal. The collection creation process typically completes in under a minute. Monitor progress with `get_collection_info`.

**Errors:** `Unauthorized` (you have no organization, are not a member of `org_id`, or your role cannot create collections), `OrgSuspended`, `InvalidNftTemplateId` (the template does not exist or belongs to another organization), `UnknownCategory`, `UnknownCertificateType`.

***

### Certificate Types

Every collection issues one kind of certificate, chosen when the collection is created:

| Value        | Meaning                                                                       |
| ------------ | ----------------------------------------------------------------------------- |
| `"standard"` | The certificates the Minting Studio has always issued. This is the default.   |
| `"dpp"`      | A **Digital Product Passport**.                                               |

The type determines the structure of the template the collection uses and how its certificates are
rendered. Choose it at creation with the `certificate_type` field; omit it and you get `"standard"`.

**It cannot be changed afterwards.** `update_collection_metadata` does not accept it. To change
type, create a new collection.

Every collection created before certificate types existed reports `"standard"`, so an existing
integration sees no change.

{% hint style="info" %}
`certificate_type` is a different axis from whether a collection is AI-created. A collection has
both: an AI collection issues `"standard"` certificates unless it says otherwise. See
[REST API Overview](../rest-api/overview.md) for how the two filters differ.
{% endhint %}

Every collection-shaped and NFT-shaped read endpoint accepts a `certificate_type` filter. On the
canister, `list_all_collections`, `get_collections_by_owner` and `get_collections_for_user` take
`certificate_type : opt vec text`:

```bash
# Only DPP collections
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai list_all_collections '(record {
  pagination = record { offset = null; limit = null };
  categories = null;
  certificate_type = opt vec { "dpp" }
})'
```

Passing `null` (or an empty vector) returns every type. A name this canister does not recognise
returns an empty page rather than an error, so you can filter on a certificate type before it ships.

**Adding a new certificate type never breaks your integration.** The value travels as `text` in both
directions rather than as a Candid variant, precisely so that a third type can be introduced without
invalidating bindings you generated from an older `can.did`.

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
| `owner`         | Principal of the owning organization's current Owner (changes when ownership is transferred) |
| `ogy_charged`   | OGY tokens charged for creation                              |
| `certificate_type` | `"standard"` or `"dpp"`                                  |
| `is_ai`         | `true` for [AI collections](../ai-collections/overview.md)   |
| `temaplte_url`  | URL of the newest version of the collection's template, `null` until the template upload finishes. The field name is misspelled in the interface; use it as written. |
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

**Returns:** `CollectionsResult` with a vector of `CollectionInfo` and `total_count`. For an Owner this includes the collections of the organization they own.

#### List an Organization's Collections

Every collection of an organization, whatever your role in it:

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/orgs/{id_or_slug}/collections" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

**Using dfx instead**

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai get_collections_by_org '(record {
  org_id = 12 : nat64;
  categories = null;
  certificate_type = null;
  pagination = record { offset = null; limit = opt 10 }
})'
```

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

**Returns:** A vector of `NftDetails`. Burned tokens are dropped, which means the vector can be **shorter than `token_ids`**: match entries by `token_id`, never by position. Each entry contains:

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

Ownership, holder and statistics queries are served over HTTP by the ORIGYN gateway under `https://gateway.origyn.com/v1/nft/production/`. These endpoints are public and need no API key, so you can call any of them right here.

| Question | Endpoint |
| -------- | -------- |
| Certificates a collection creator minted or holds | `GET /owners/{principal}/nfts` |
| Certificates an account currently holds | `GET /accounts/{principal}/nfts` |
| Certificates an account held and transferred away | `GET /accounts/{principal}/past-nfts` |
| Current holders of a collection | `GET /collections/{canister_id}/holders` |
| Totals for one account | `GET /accounts/{principal}/stats` |
| Totals for one collection | `GET /collections/{canister_id}/stats` |

### Certificates by collection creator

Certificates minted in collections the principal created, plus certificates they currently hold from other collections.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/owners/{principal}/nfts" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### Certificates by holder

Certificates an account currently holds, regardless of who minted them.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/accounts/{principal}/nfts" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### Past certificates of a holder

Certificates an account used to hold and gave away by transfer.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/accounts/{principal}/past-nfts" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### Holders of a collection

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/collections/{canister_id}/holders" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### Account stats

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/accounts/{principal}/stats" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### Collection stats

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/collections/{canister_id}/stats" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

Things to know about these endpoints:

* **`owners` and `accounts` are not the same thing.** Under `/owners/{principal}/…` the principal is a collection **creator**; under `/accounts/{principal}/…` it is a current **holder**.
* **Timestamps are milliseconds**, and the field names say so (`minted_at_ms`, `released_at_ms`).
* **Pagination is `?limit=` and `?offset=`**, with `limit` defaulting to 50 and capped at 100. Out-of-range values are rejected with a 400 rather than clamped.
* **Principals carry no subaccount**; they are plain strings.

The four **list** endpoints also support `?sort=` and `?order=`. The two `/stats` endpoints take no query parameters.

{% hint style="info" %}
The full, always-current schema for every HTTP endpoint is published at [gateway.origyn.com/docs](https://gateway.origyn.com/docs/).
{% endhint %}

### `get_nft` (single certificate)

Fetch one certificate plus its collection-level info.

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai get_nft '(record {
  collection = principal "<collection_canister_id>";
  token_id   = 1 : nat
})'
```

`get_nft` queries the collection canister directly, so a certificate is readable the moment it is minted, with no indexing delay.

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
