---
icon: shield-check
---

# Paid Requests & Idempotency

Two endpoints in the REST API spend your OGY:

| Endpoint | What it costs |
| -------- | ------------- |
| `POST /create_collection` | The collection creation fee (15,000 OGY) |
| `POST /initialize_mint`   | The minting fee for the batch you reserve |

Both require an **`Idempotency-Key`** header. A request without one is rejected before anything is charged.

## Why this is gated

The header is a deliberate safety gate, and it does two jobs.

**It makes a charge impossible to trigger by accident.** These endpoints are live on this page, so pressing **Test it** sends a real request against production. Because the header is required and carries no default value, nothing can fire until you deliberately fill it in.

**It makes retries safe.** Networks time out, browsers get closed, and clients retry. Without an idempotency key, a retry after a timeout you never saw the response to would charge you a second time. With one, the retry returns the original result and charges you once.

{% hint style="warning" %}
There is no sandbox. These endpoints go to production and spend real OGY.
{% endhint %}

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

This is the one rule that decides whether the mechanism protects you or does nothing at all.

**Generate the key once per operation you intend to perform, store it, and reuse that same value for every retry of that operation.**

The key is what tells the server "this is the same purchase you already saw." If a retry carries a *new* key, the server has no way to know it is a retry, so it treats it as a second, separate purchase and charges you again. Generating the key inside your retry loop therefore defeats the entire mechanism, and it fails in the worst possible way: it looks completely fine until the one time a request times out, which is exactly when you needed the protection.

{% hint style="danger" %}
**Never generate the key inside the retry loop.**

```bash
# WRONG - a new key each attempt, so a timeout followed by a retry charges twice
for attempt in 1 2 3; do
  curl ... -H "Idempotency-Key: $(uuidgen)"
done

# RIGHT - one key for the operation, reused by every attempt
KEY=$(uuidgen)
for attempt in 1 2 3; do
  curl ... -H "Idempotency-Key: $KEY"
done
```
{% endhint %}

The same reasoning applies beyond shell loops. If your HTTP client retries automatically, the key must be fixed *before* it is handed to that client, not computed per attempt. And if the operation spans a process restart, for example a job that resumes after a crash, persist the key alongside the job so the resumed run reuses it rather than minting a fresh one.

{% hint style="info" %}
There is one time bound on that. If a first attempt was charged but never confirmed, the retry replays the original ledger transfer, and the OGY ledger only accepts a transaction whose timestamp falls inside its window (about 24 hours). A retry that lands later cannot be deduplicated against the original, so resume within a day, or reconcile manually rather than retrying blind.
{% endhint %}

A useful way to think about it: **the key names the purchase, not the request.** One purchase, one key, however many attempts it takes.

## What each outcome means

The key is scored against the **payer plus a fingerprint of the full request body**.

| Situation | Result |
| --------- | ------ |
| No `Idempotency-Key` header | `400` `missing_idempotency_key` — nothing is charged |
| First use of a key | The request runs and you are charged once |
| Same key, **same** body | The original result is replayed. You are **not** charged again |
| Same key, **different** body | `409` `idempotency_key_conflict` — nothing is charged |
| Same key, first call still running | `409` `concurrent_request` — wait, then retry |

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
        "categories": []
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

{% hint style="info" %}
`categories` is optional here, unlike the direct canister call, where it is a required field and must be sent as `vec {}` even when empty.
{% endhint %}

**`request_id` is not the collection canister id.** It is the identifier of the creation request, returned as a string. Provisioning happens asynchronously, so poll for the canister id:

```bash
curl https://gateway.origyn.com/gateway/v1/nft/production/collections/1042/status
```

Poll until `status` is `TemplateUploaded` and `canister_id` is non-null. That `canister_id` is what every later call wants. This endpoint needs no API key.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/collections/{id}/status" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### Try it

{% hint style="danger" %}
**This spends 15,000 OGY.** It is a real production call. Fill in `Idempotency-Key` deliberately.
{% endhint %}

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

{% hint style="warning" %}
`total_file_size_bytes` is a **string**, not a number, and it is a hard cap you have paid for. Uploads that exceed it are rejected. Size it generously; the unused portion is refunded when you close the request, minus the ledger transfer fee. A residual smaller than that fee cannot be sent and is burned instead.
{% endhint %}

Estimate the cost first, which is free:

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/estimate" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### Try it

{% hint style="danger" %}
**This charges OGY.** It is a real production call.
{% endhint %}

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
