---
icon: cloud
---

# REST API Overview

Everything the Minting Studio does is available over plain HTTP. You can create collections, upload files, mint certificates, manage your organization, and query the index without writing any Internet Computer code.

This is the recommended way to integrate, and the only way to use [private content](../private-content/overview.md): private values can only be written and read here.

## Base URL

```
https://gateway.origyn.com
```

There are two families of endpoint underneath it, and they behave differently.

| Family | Path prefix | Auth | What it does |
| ------ | ----------- | ---- | ------------ |
| **Public reads** | `/v1/nft/production/` | none | Query collections, certificates, holders, organizations, templates, transactions, search |
| **Gateway** | `/gateway/v1/nft/production/` | API key or session token | Create collections and templates, upload files, mint, settle, manage your organization and reader groups, read private content |

The public read API needs no credential. The gateway needs a bearer credential; see [Obtaining an API Key](api-keys.md).

## Filtering by collection type

Two filters look similar and are easy to confuse. They select on different things, and you can use
either, both, or neither:

| Filter             | Values                                              | Selects by                                    |
| ------------------ | --------------------------------------------------- | --------------------------------------------- |
| `collection_type`  | `ai`, `normal`                                      | **how the collection was created**            |
| `certificate_type` | `standard`, `dpp` (comma-separate for several)      | **what kind of certificate it issues**        |

Omit either one to get everything. `certificate_type` accepts a comma-separated list, so
`?certificate_type=standard,dpp` is the same as omitting it today.

```bash
# Every DPP collection
curl "https://gateway.origyn.com/v1/nft/production/collections?certificate_type=dpp"

# Every collection of one organization
curl "https://gateway.origyn.com/v1/nft/production/collections?org_id=12"

# DPP certificates held by one account
curl "https://gateway.origyn.com/v1/nft/production/accounts/$PRINCIPAL/nfts?certificate_type=dpp"
```

Two things worth knowing:

- Collections created before certificate types existed report `standard`, so nothing you already
  query changes shape or disappears.
- An unrecognised value returns an **empty page, not an error**. That is deliberate: it means you
  can start filtering on a certificate type before it has shipped, and a new type never turns your
  working request into a `400`.

## Authentication

Gateway endpoints take a bearer credential, either an API key or a session token:

```bash
curl https://gateway.origyn.com/gateway/v1/nft/production/collections \
  -H "Authorization: Bearer $ORIGYN_API_KEY"
```

Your credential identifies you on every request. There is no way to act on behalf of another principal: the identity comes from the credential alone and is never read from a URL, body, or header.

| Credential | Looks like | Lifetime | Get it from |
| ---------- | ---------- | -------- | ----------- |
| API key | `sk_live_` + 32 hex characters | Until revoked | The dashboard, or `POST /keys` |
| Session token | a JWT | At most 24 hours | `POST /auth/session` |

Both are accepted by every gateway endpoint. See [Obtaining an API Key](api-keys.md).

Writes also need your principal's on-chain authorization of the gateway, which you sign when you create a key. Without it, writes fail with `403 not_delegable` or `403 not_authorized` while reads keep working. See [Withdrawing the gateway's authorization](api-keys.md#withdrawing-the-gateways-authorization).

Reads need no authentication at all. Try one now:

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/collections" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

## Interactive reference

**Every endpoint is callable from these pages.** Each operation renders with its full request and response schema and a **Test it** button that sends a real request. See [Endpoint Reference](reference.md) for all of them in one place, or use the blocks on each guide page.

Public reads need nothing. For write endpoints, paste your key into **Authorize** first.

The same spec is also browsable at [gateway.origyn.com/docs](https://gateway.origyn.com/docs/).

{% hint style="warning" %}
**Test it** sends a real production request. Reads are harmless, but `create_collection` and `initialize_mint` spend real OGY, which is why both make you fill in an `Idempotency-Key` first. See [Paid Requests](paid-requests.md).
{% endhint %}

## Response conventions

### Pagination

List endpoints take `?limit=` and `?offset=`.

| Parameter | Default | Range |
| --------- | ------- | ----- |
| `limit`   | 50      | 1 to 100 |
| `offset`  | 0       | 0 or greater |

Values outside those ranges are **rejected with a 400**, not silently clamped, so a bad page size fails loudly rather than returning a page you did not ask for.

Responses share one envelope:

```json
{
  "items":  [ ... ],
  "total":  1284,
  "limit":  50,
  "offset": 0,
  "sort":   "minted_at"
}
```

`total` is the full number of matching records, independent of the page you requested.

`/search` is the exception: its `limit` defaults to 20 and is clamped rather than rejected, and it takes no `offset`, `sort` or `order`.

### Sorting

Most list endpoints accept `?sort=` and `?order=`. Prefix a sort key with `-` for descending:

Newest first can be spelled two equivalent ways: `sort=-minted_at`, or `sort=minted_at&order=desc`.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/nfts" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

A `-` prefix wins over a conflicting `?order=`. Passing `?order=` alone flips the endpoint's default sort key.

{% hint style="warning" %}
Sorting is **not validated**. An unrecognised sort key falls back to the endpoint default and still returns `200`, so a typo gives you correct-looking data in the wrong order. Check the `sort` field in the response to see what was applied.
{% endhint %}

### Timestamps

Every timestamp is an epoch value in **milliseconds**, and the field names say so: `created_at_ms`, `minted_at_ms`, `released_at_ms`, `first_activity_ms`.

The canister API uses **nanoseconds**, so divide by 1,000,000 when porting from `dfx`.

### Errors

Gateway endpoints return a consistent error body:

```json
{ "error": "insufficient_allowance", "message": "approve more OGY to claimlink" }
```

Branch on the `error` code. The body is JSON, but it is sent with `Content-Type: text/plain`, so parse the body rather than relying on the header.

| Status | Meaning | Common codes |
| ------ | ------- | ------------ |
| `400` | Malformed request | `missing_idempotency_key`, `invalid_id`, `invalid_metadata`, `unknown_category`, `invalid_org_id`, `reserved_path` |
| `401` | Missing, bad, revoked or expired credential | `unauthorized` |
| `402` | Not enough OGY | `insufficient_allowance`, `insufficient_funds` |
| `403` | Not permitted | `not_owner`, `not_permitted`, `insufficient_role`, `not_a_member`, `org_suspended`, `org_mismatch`, `not_delegable`, `not_authorized`, `no_delegation`, `file_not_uploaded` |
| `404` | No such thing | `not_found` |
| `409` | Conflict | `idempotency_key_conflict`, `concurrent_request`, `collection_not_ready`, `mint_limit_exceeded`, `template_in_use`, `template_limit_exceeded`, `too_many_template_versions` |
| `413` | Body too large | plain-text, no JSON envelope (see the batch-size note in [Minting](../minting-studio/minting.md#step-4-mint-the-certificates)) |
| `429` | Rate limited | `rate_limited` |
| `500` | Internal error | `internal` |
| `502` | Upstream unavailable | `upstream_unavailable`, `canister_unavailable`, `key_unavailable` |

Public read endpoints return a plain-text message with the status code rather than a JSON envelope.

`GET /allowance` reports an ICRC-2 **approval**, not a **balance**, so a paid call can still fail with `402 insufficient_funds` while it shows a large number. Whose approval it reports depends on the credential: an organization-bound key reports the organization's billing principal, anything else reports your own. Charges always come from the billing principal, so check `GET /orgs/{org_id}/allowance` before paying.

## Limits

There is no rate limiting on the read API today. Please be considerate with automated polling; index data is refreshed continuously, so aggressive polling gains you very little.

A few gateway actions are rate limited: sending organization invitations (200 per organization per day) and email verification (5 verification emails per hour). They answer `429 rate_limited`.

Fresh certificates take a few seconds to reach the index after minting. If you need a certificate the instant it is minted, read it live with `GET /collections/{canister_id}/nfts/{token_id}`, which goes straight to the collection canister. `GET /v1/nft/production/sync-status` reports how current the index is.

## Next steps

* [Obtaining an API Key](api-keys.md): credentials and the gateway's authorization.
* [Paid Requests](paid-requests.md): the two endpoints that spend OGY.
* [Troubleshooting](troubleshooting.md): what a refused call means.
* [Endpoint Reference](reference.md): every endpoint, callable.
