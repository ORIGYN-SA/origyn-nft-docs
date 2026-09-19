---
icon: shield-check
---

# Paid Requests & Idempotency

Two endpoints in the REST API spend your OGY:

| Endpoint | What it costs |
| -------- | ------------- |
| `POST /create_collection` | The collection creation fee (15,000 OGY) |
| `POST /initialize_mint`   | The minting fee for the batch you reserve |

See [Pricing](../core-concepts/pricing.md) for the numbers.

Both require an **`Idempotency-Key`** header. A request without one is rejected before anything is charged.

Both charge your organization's **billing principal**, not the member who makes the call. The billing principal must have approved the Minting Studio to spend OGY; see [Paying: the billing principal](../minting-studio/organizations.md#paying-the-billing-principal).

There is no sandbox: these endpoints go to production and spend real OGY.

The header does two jobs. It stops an accidental charge, because nothing fires until you fill it in yourself, and it makes retries safe: without it, retrying after a timeout you never saw the answer to would charge you twice.

## Producing an idempotency key

The key is any string you choose. It only has to be **unique per distinct request you intend to make**. A UUID is the usual choice:

{% tabs %}
{% tab title="macOS / Linux" %}
```bash
uuidgen
# 4f0a1d2e-9c3b-4a71-8e55-2b1d6f0c9a84
```
{% endtab %}

{% tab title="Python" %}
```python
import uuid
print(uuid.uuid4())
```
{% endtab %}

{% tab title="Node" %}
```js
crypto.randomUUID()
```
{% endtab %}

{% tab title="Browser" %}
```js
crypto.randomUUID()
```
{% endtab %}
{% endtabs %}

A readable prefix is often more useful than a bare UUID when you are reconciling later, for example `collection-goldbars-2026-08-01-a3f9`.

### Generate the key once, before you send

**The key names the purchase, not the request.** One purchase, one key, however many attempts it takes.

A retry carrying a new key looks like a new purchase and is charged again, so generate the key before the first attempt and reuse it for every retry.

{% hint style="danger" %}
**Never generate the key inside the retry loop.**

```bash
# WRONG: a new key each attempt, so a timeout followed by a retry charges twice
for attempt in 1 2 3; do
  curl ... -H "Idempotency-Key: $(uuidgen)"
done

# RIGHT: one key for the operation, reused by every attempt
KEY=$(uuidgen)
for attempt in 1 2 3; do
  curl ... -H "Idempotency-Key: $KEY"
done
```
{% endhint %}

The same applies outside shell loops. If your HTTP client retries for you, fix the key before handing it the request. If the job can restart, store the key with the job.

One time bound: a charged but unconfirmed first attempt is replayed against the OGY ledger, which only accepts transactions timestamped within about 24 hours. Resume within a day, or reconcile by hand rather than retrying blind.

## What each outcome means

The key is scored against the **payer plus a fingerprint of the full request body**.

| Situation | Result |
| --------- | ------ |
| No `Idempotency-Key` header | `400` `missing_idempotency_key`. Nothing is charged |
| First use of a key | The request runs and you are charged once |
| Same key, **same** body | The original result is replayed. You are **not** charged again |
| Same key, **different** body | `409` `idempotency_key_conflict`. Nothing is charged |
| Same key, first call still running | `409` `concurrent_request`. Wait, then retry |

So: **reuse the key to retry, change the key to make a new purchase.**

Errors always come back in the same shape:

```json
{ "error": "idempotency_key_conflict", "message": "idempotency key reused with a different body" }
```

## Creating a collection

```bash
IDEMPOTENCY_KEY=$(uuidgen)

curl -X POST https://gateway.origyn.com/gateway/v1/nft/production/create_collection \
  -H "Authorization: Bearer $ORIGYN_API_KEY" \
  -H "Idempotency-Key: $IDEMPOTENCY_KEY" \
  -H "Content-Type: application/json" \
  -d '{
        "name": "My Gold Bar Collection",
        "symbol": "GBC",
        "description": "Certified gold bar certificates",
        "template_id": 1,
        "categories": [],
        "org_id": 12
      }'
```

```json
{ "request_id": "1042", "status": "Queued" }
```

| Field | Type | Notes |
| ----- | ---- | ----- |
| `name`, `symbol`, `description` | string | Required |
| `template_id` | integer | Required. A template you own |
| `categories` | string[] | Optional over HTTP. Each name must already exist in the taxonomy |
| `org_id` | integer | Optional. The organization that will own the collection. Omitted: your key's organization if it is bound to one, otherwise the organization you own. Required in practice if you are a member of an organization you do not own |
| `certificate_type` | string | Optional. `"standard"` (default) or `"dpp"`. Cannot be changed later |

Only an Owner or Admin can create collections, and the template must belong to the same organization. A refusal is `403 not_permitted`, or `403 org_suspended` for a suspended organization.

**`request_id` is not the collection canister id.** It is the identifier of the creation request, returned as a string. Provisioning happens asynchronously, so poll for the canister id:

```bash
curl https://gateway.origyn.com/gateway/v1/nft/production/collections/1042/status
```

Poll until `status` is `TemplateUploaded` and `canister_id` is non-null. That `canister_id` is what every later call wants. This endpoint needs no API key.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/collections/{id}/status" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### Try it

**This spends 15,000 OGY** and is a real production call.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/create_collection" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

## Initializing a mint

Reserves capacity and charges the fee up front. Use the `canister_id` from the previous step, not the `request_id`.

```bash
IDEMPOTENCY_KEY=$(uuidgen)

curl -X POST https://gateway.origyn.com/gateway/v1/nft/production/initialize_mint \
  -H "Authorization: Bearer $ORIGYN_API_KEY" \
  -H "Idempotency-Key: $IDEMPOTENCY_KEY" \
  -H "Content-Type: application/json" \
  -d '{
        "collection_canister_id": "aaaaa-bbbbb-ccccc-ddddd-cai",
        "num_mints": 10,
        "total_file_size_bytes": "5000000"
      }'
```

```json
{ "mint_request_id": "77", "status": "Initialized" }
```

| Field | Type | Notes |
| ----- | ---- | ----- |
| `collection_canister_id` | string | The collection's NFT canister principal |
| `num_mints` | integer | Certificates to reserve. `0` opens an upload-only session |
| `total_file_size_bytes` | **string** | Total bytes you will upload, as a decimal string |
| `private_file_sizes` | integer[] | Optional. Plaintext sizes of the private files you will upload, so the reservation covers encryption. See [Minting Private Content](../private-content/minting.md#1-reserve-room-for-encryption) |

Any Owner, Admin or Minter of the collection's organization can open a mint request.

`total_file_size_bytes` is a string, not a number, and it is a hard cap you have already paid for: uploads stop once you reach it. Size it generously, because the unused part is refunded when you close the request, minus the ledger transfer fee.

Estimate the cost first, which is free. See [Pricing](../core-concepts/pricing.md):

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/estimate" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### Try it

**This charges OGY** and is a real production call.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/initialize_mint" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

## Retrying safely

Note that **both** 409s share a status code but mean opposite things: `concurrent_request` is worth retrying, `idempotency_key_conflict` never is. Branch on the `error` field, not on the status:

```bash
# Generate ONCE, outside the retry loop.
IDEMPOTENCY_KEY=$(uuidgen)

REQUEST_BODY='{
  "collection_canister_id": "aaaaa-bbbbb-ccccc-ddddd-cai",
  "num_mints": 10,
  "total_file_size_bytes": "5000000"
}'

for attempt in 1 2 3; do
  response=$(curl -s -X POST \
    https://gateway.origyn.com/gateway/v1/nft/production/initialize_mint \
    -H "Authorization: Bearer $ORIGYN_API_KEY" \
    -H "Idempotency-Key: $IDEMPOTENCY_KEY" \
    -H "Content-Type: application/json" \
    -d "$REQUEST_BODY")

  # Not every failure is the JSON envelope: a malformed request is rejected by the
  # HTTP layer as plain text, so treat unparseable output as a hard failure.
  error=$(jq -r '.error // empty' <<<"$response" 2>/dev/null) || {
    echo "failed (non-JSON response): $response"; break
  }

  case "$error" in
    "")                 echo "$response"; break ;;          # success
    concurrent_request) sleep 2; continue ;;                # in flight: retry the SAME key
    *)                  echo "failed: $response"; break ;;  # fix it, then use a NEW key
  esac
done
```

Two rules cover every case:

* **Retrying the same operation?** Reuse the key. Never regenerate inside the loop.
* **Making a genuinely new purchase?** Use a new key. Reusing one with a different body returns `409` and does nothing.
