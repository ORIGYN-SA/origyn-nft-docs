---
icon: list-tree
---

# Endpoint Reference

Every endpoint, live. Public reads need no credential, so you can call them right here with **Test it**. Gateway endpoints need an API key or session token: paste it into **Authorize** first. See [Obtaining an API Key](api-keys.md).

{% hint style="info" %}
The `env` field is prefilled with `production`. Leave it as it is.
{% endhint %}

{% hint style="info" %}
**Filtering by certificate type.** These endpoints accept `?certificate_type=standard`,
`?certificate_type=dpp`, or a comma-separated list; omit it for all types:

`GET /collections`, `GET /nfts`, `GET /search`, `GET /accounts/{principal}/nfts`,
`GET /accounts/{principal}/past-nfts`, `GET /accounts/{principal}/collections`,
`GET /owners/{principal}/nfts`, `GET /owners/{principal}/collections`, `GET /orgs/{id_or_slug}/collections`,
`GET /orgs/{id_or_slug}/nfts`, `GET /transactions`.

An unrecognised value returns an empty page rather than an error. This is a different axis from
`collection_type=ai|normal`; see [Overview](overview.md#filtering-by-collection-type).

On the write side, `POST /create_collection` takes an optional `certificate_type` in the body
(`"standard"` by default, or `"dpp"`); an invalid value returns `400 unknown_certificate_type`.
The choice is immutable once the collection exists.
{% endhint %}

## Public reads

### Collections

#### `GET /collections`

Filter by `category`, `org_id`, `collection_type` and `certificate_type`.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/collections" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `GET /collections/count`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/collections/count" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `GET /collections/{canister_id}`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/collections/{canister_id}" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `GET /collections/{canister_id}/stats`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/collections/{canister_id}/stats" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `GET /collections/{canister_id}/holders`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/collections/{canister_id}/holders" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `GET /collections/{canister_id}/template`

The collection's template. Pass `?version=n` for a specific [template version](../minting-studio/templates.md#template-versions).

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/collections/{canister_id}/template" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `GET /collections/{canister_id}/token-ids`

Live `icrc7_tokens` passthrough, paged with `?prev=&take=`.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/collections/{canister_id}/token-ids" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### Certificates

#### `GET /nfts`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/nfts" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `GET /collections/{canister_id}/nfts`

Live batch read from the collection canister, `?ids=1,2,3` (at most 100).

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/collections/{canister_id}/nfts" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `GET /collections/{canister_id}/nfts/{token_id}`

Live metadata and current owner, straight from the collection canister.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/collections/{canister_id}/nfts/{token_id}" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `GET /collections/{canister_id}/tokens/{token_id}`

Served from the index; lags slightly behind a mint.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/collections/{canister_id}/tokens/{token_id}" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### Accounts and owners

#### `GET /accounts/{principal}/nfts`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/accounts/{principal}/nfts" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `GET /accounts/{principal}/past-nfts`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/accounts/{principal}/past-nfts" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `GET /accounts/{principal}/collections`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/accounts/{principal}/collections" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `GET /accounts/{principal}/stats`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/accounts/{principal}/stats" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `GET /owners/{principal}/nfts`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/owners/{principal}/nfts" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `GET /owners/{principal}/collections`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/owners/{principal}/collections" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `GET /owners/{principal}/templates`

All of an owner's templates, read live from the Minting Studio.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/owners/{principal}/templates" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `GET /owners/{principal}/uploads`

Files a principal uploaded. Private files are listed but not decrypted; use the gateway version below for keys.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/owners/{principal}/uploads" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### Organizations

`{id_or_slug}` is an organization id (all digits) or its slug. A slug the organization has given up answers `301` to its current address.

#### `GET /orgs/{id_or_slug}/collections`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/orgs/{id_or_slug}/collections" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `GET /orgs/{id_or_slug}/nfts`

Certificates issued by the organization's collections.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/orgs/{id_or_slug}/nfts" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `GET /orgs/{id_or_slug}/templates`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/orgs/{id_or_slug}/templates" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `GET /orgs/{id_or_slug}/uploads`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/orgs/{id_or_slug}/uploads" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### Search, categories and templates

#### `GET /search`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/search" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `GET /categories`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/categories" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `GET /categories/catalog`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/categories/catalog" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `GET /templates/{template_id}`

One template. Pass `?version=n` for a specific version; the response carries `version` and `current_version`.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/templates/{template_id}" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `GET /templates/{template_id}/versions`

Every version of a template, newest first.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/templates/{template_id}/versions" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### Transactions and freshness

#### `GET /transactions`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/transactions" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `GET /sync-status`

How current the index is.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/sync-status" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

## Gateway endpoints

These need an API key or session token, except where noted. Worked examples live on the guide pages:

* [Obtaining an API Key](api-keys.md): keys, sessions and the gateway's authorization.
* [Paid Requests & Idempotency](paid-requests.md): `create_collection` and `initialize_mint`.
* [Organizations & Members](../minting-studio/organizations.md): members, invitations, billing.
* [Private Content](../private-content/overview.md): reader groups, private uploads, decrypted reads.
* [Minting](../minting-studio/minting.md) and [Managing Collections](../minting-studio/managing-collections.md).

### Authentication and keys

#### `POST /auth/challenge`

No credential. Returns a one-time code to sign on chain with `prove_principal`.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/auth/challenge" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `POST /auth/session`

No credential. Exchanges a signed code for a session token of at most 24 hours.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/auth/session" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `POST /keys`

No credential. Exchanges a signed code for an API key, optionally bound to an organization.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/keys" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `GET /keys`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/keys" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `DELETE /keys/{id}`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/keys/{id}" method="delete" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### Your account

#### `GET /me`

Who you are: your profile, your organizations and roles, pending invitations, and your application while you have no organization.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/me" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `PUT /me`

Replaces your personal profile. Omitted fields are cleared.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/me" method="put" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `POST /me/email`

Records an email address as unverified and mails a verification link. At most 5 per hour.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/me/email" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `POST /me/email/verify`

No credential. Confirms the address with the token from the link, valid for 24 hours.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/me/email/verify" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `GET /allowance`

Your own OGY approval to the Minting Studio, or the billing principal's for an organization-bound key.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/allowance" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### Applications

The dashboard's application form uses these. See [Getting Started](../minting-studio/getting-started.md#1-apply-for-access).

#### `POST /applications`

Submit the form, or resubmit after changes were requested. Only `kind` (`company` or `person`) is required.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/applications" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `GET /applications/me`

Your application and its status: `pending`, `approved` or `rejected` (with the reviewer's reason).

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/applications/me" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `PATCH /applications/me`

Correct an application that is still pending. Omitted fields are kept; `""` clears one.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/applications/me" method="patch" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### Organizations

#### `GET /orgs`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/orgs" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `GET /orgs/{org_id}`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/orgs/{org_id}" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `PATCH /orgs/{org_id}`

Owner or Admin.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/orgs/{org_id}" method="patch" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `PATCH /orgs/{org_id}/slug`

Owner only.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/orgs/{org_id}/slug" method="patch" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `GET /orgs/{org_id}/allowance`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/orgs/{org_id}/allowance" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `PATCH /orgs/{org_id}/billing`

Owner only. Allowed while suspended.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/orgs/{org_id}/billing" method="patch" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `POST /orgs/{org_id}/transfer`

Owner only. Allowed while suspended.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/orgs/{org_id}/transfer" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `GET /orgs/{org_id}/members`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/orgs/{org_id}/members" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `PATCH /orgs/{org_id}/members/{principal}`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/orgs/{org_id}/members/{principal}" method="patch" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `DELETE /orgs/{org_id}/members/{principal}`

Allowed while suspended.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/orgs/{org_id}/members/{principal}" method="delete" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `GET /orgs/{org_id}/invites`

Owner or Admin. Email invitations not yet claimed.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/orgs/{org_id}/invites" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `POST /orgs/{org_id}/invites`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/orgs/{org_id}/invites" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### Invitations

#### `GET /invites`

Direct invitations addressed to you.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/invites" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `POST /invites/{org_id}/accept`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/invites/{org_id}/accept" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `POST /invites/{org_id}/decline`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/invites/{org_id}/decline" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `POST /invites/claim`

Claim an email invitation with the `slot_id` and `secret` from its link.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/invites/claim" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### Reader groups

#### `GET /orgs/{org_id}/groups`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/orgs/{org_id}/groups" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `POST /orgs/{org_id}/groups`

Owner or Admin.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/orgs/{org_id}/groups" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `DELETE /orgs/{org_id}/groups/{group_id}`

Owner or Admin.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/orgs/{org_id}/groups/{group_id}" method="delete" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `POST /orgs/{org_id}/groups/{group_id}/members`

Owner or Admin.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/orgs/{org_id}/groups/{group_id}/members" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `DELETE /orgs/{org_id}/groups/{group_id}/members/{principal}`

Owner or Admin.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/orgs/{org_id}/groups/{group_id}/members/{principal}" method="delete" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

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

The templates of the organization you own.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/templates" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### Collections

#### `POST /create_collection`

**Spends 15,000 OGY.** Requires `Idempotency-Key`.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/create_collection" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `GET /collections/{id}/status`

No credential. Poll a creation request until it has a `canister_id`.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/collections/{id}/status" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `POST /update_collection_metadata`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/update_collection_metadata" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `POST /upload_logo/{collection_canister_id}`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/upload_logo/{collection_canister_id}" method="post" %}
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

#### `POST /private_uploads`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/private_uploads" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `POST /private_uploads/p/{upload_hex}/chunks/{n}`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/private_uploads/p/{upload_hex}/chunks/{n}" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `POST /private_uploads/p/{upload_hex}/finalize`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/private_uploads/p/{upload_hex}/finalize" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

### Mint sessions

#### `GET /estimate`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/estimate" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `POST /initialize_mint`

**Charges OGY.** Requires `Idempotency-Key`.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/initialize_mint" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `POST /mint_json_nfts`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/mint_json_nfts" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `GET /mint_requests`

Mint requests you opened yourself.

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

### Keyed reads

These return certificates and uploads with their private content decided for you. See [Reading Private Content](../private-content/reading.md).

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

#### `GET /collections/{id}/nfts/{token_id}`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/collections/{id}/nfts/{token_id}" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `GET /owners/{principal}/nfts`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/owners/{principal}/nfts" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `GET /orgs/{org_id}/nfts`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/orgs/{org_id}/nfts" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `GET /owners/{principal}/uploads`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/owners/{principal}/uploads" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

#### `GET /orgs/{org_id}/uploads`

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/orgs/{org_id}/uploads" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}
