---
icon: life-ring
---

# Troubleshooting

Gateway refusals all carry `{ "error": "<code>", "message": "..." }`. Branch on `error`, not on the status alone. Public reads answer in plain text with no envelope.

## It failed, what now?

Start from what you observed, not from the code.

### Credentials and permissions

| What you see | Cause, and what to do |
| ------------ | --------------------- |
| `401` on every call | One code covers a missing header, a revoked or mistyped key, an expired session token, and a key from another environment; it never says which. Production keys start `sk_live_` and work only there. |
| `403 not_delegable`, `not_authorized` or `no_delegation`, while reads keep working | Your principal's on-chain authorization of the gateway is missing or expired, so only writes stop. Sign `prove_principal` again, or create a dashboard key, which restores every key. [How](api-keys.md#withdrawing-the-gateways-authorization). |
| `403 not_permitted` | Collapsed on purpose: not a member, no such organization, or too small a [role](../minting-studio/organizations.md#roles). Check which organization you are acting for before asking for a promotion. |
| `403 insufficient_role` | Genuinely your role, from the membership and reader-group endpoints. An Owner or Admin can promote you. |
| `403 not_a_member` | No active membership. A nonexistent organization gets the same answer, so a wrong `org_id` looks identical. |
| `403 org_mismatch` | The key is bound to one organization and the body named another. |
| `403 not_owner` on minting, uploads or `initialize_mint` | Often a [suspended organization](../minting-studio/organizations.md#suspension) rather than ownership: those endpoints do not answer `org_suspended`. |

### Paying, and retrying safely

| What you see | Cause, and what to do |
| ------------ | --------------------- |
| `402 insufficient_allowance` or `402 insufficient_funds` | The organization's billing principal pays, never the member calling: the first is its approval ceiling, the second the balance behind it. `GET /orgs/{org_id}/allowance` reports that approval, a ceiling and not a balance. [Billing](../minting-studio/organizations.md#paying-the-billing-principal). |
| `400 missing_idempotency_key` | `create_collection` and `initialize_mint` require the header; nothing was charged. Generate one key per purchase, outside your retry loop. |

{% hint style="warning" %}
**The two 409s mean opposite things.** `concurrent_request` means that key's first call is still in flight: wait, then retry the **same** key. `idempotency_key_conflict` means you reused a key with a different body, and is never retryable. See [Paid Requests](paid-requests.md).
{% endhint %}

### Size limits

| What you see | Cause, and what to do |
| ------------ | --------------------- |
| `413` in plain text, with no `error` field | The body passed the 2 MB request limit. Mint in smaller batches; about 40 items works. |
| `413 json_too_large`, `byte_limit_exceeded` or `chunk_too_large` | Three ceilings: 50 KiB per certificate including its private block; per mint request, the bytes reserved in `total_file_size_bytes`, so declare `private_file_sizes` too; per chunk, 1 MiB public (the gateway only refuses above 2 MiB, but the canister rejects anything larger than 1 MiB) or 1,048,560 plaintext bytes private, lower because the gateway encrypts. |

### Minting and reading

| What you see | Cause, and what to do |
| ------------ | --------------------- |
| `400 invalid_metadata`, or `InvalidMetadata` from the canister | Validation is lenient; only three things fail. A `required` field with no non-empty value, a `template` pin naming another template or a missing version, or your own top-level `private` key, which the private mint path calls `private_not_allowed`. |
| `403 file_not_uploaded` | The batch named a file that is not a finalized upload of this collection. `GET /mint_requests/{id}` lists the exact stored paths. |
| A certificate mints, then renders incompletely or without its image | Lenient validation lets extra keys and unknown types through. The viewer looks for reserved field ids (`name`, `certificate_title`, `certificate_image`, `description`, `company_logo`, `certified_by`), and `path` must be the `file_url` finalize returned, not the `file_path` you sent. |
| A just-minted certificate is missing from a list | Index lag of a few seconds, not a failed mint. Read it live with `GET /collections/{canister_id}/nfts/{token_id}`; `GET /v1/nft/production/sync-status` says how current the index is. |
| A private read returns `200` with `"access": "none"` | You were granted nothing: not an active member of the collection's organization, not in a reader group its items name, not the holder. A template's group names are never checked, so a typo grants nobody; membership changes reach the read service in about a minute. [Who can read what](../private-content/overview.md#who-can-read-what). |
| A private read adds `"error": "unreadable"` | The whole block is locked and deliberately does not say which: the organization is suspended, or the gateway could not open it for that collection. A suspension locks everyone, members included. `502 key_unavailable` is different, and worth retrying. |
| A collection sits in `Failed` | An attempt failed; creation retries automatically, about once a minute. Only when retries are exhausted does it move to `ReimbursingQueued`, then `Reimbursed`. `QuarantinedReimbursement` means the payout failed and needs manual resolution. [Lifecycle](../minting-studio/collections-and-certificates.md#collection-lifecycle). |
| `?sort=` appears to be ignored | Sorting is not validated: an unrecognised key falls back to the default and still returns `200`. The response envelope's `sort` field echoes what was applied. [Sorting](overview.md#sorting). |

### Direct canister calls

| What you see | Cause, and what to do |
| ------------ | --------------------- |
| `ConcurrentManagementCall` | The canister allows one management call in flight at a time, across all callers. Retry: the lock is per call, so a chunked upload blocks others only per chunk. [CLI Management](../custom-installation/cli-management.md#troubleshooting). |
| `dfx` rejects your arguments before the call goes out | Two Candid traps. An optional field needs the wrapper: `version = opt 3`, not `version = 3`. And `principal` is a Candid keyword, so a record field of that name must be quoted: `record { "principal" = principal "aaaaa-aa" }`. |

## Error code index

The codes the create, upload, mint and read path returns. Invitation and platform-admin codes are in [Organizations & Members](../minting-studio/organizations.md#errors).

| `error` | Status | What it means | Where to look |
| ------- | ------ | ------------- | ------------- |
| `unauthorized` | 401 | Any credential failure; never says which | [Keys](api-keys.md) |
| `missing_idempotency_key` | 400 | A paid call arrived without the header | [Paid](paid-requests.md) |
| `invalid_metadata` | 400 | The certificate failed template validation | [Minting](../minting-studio/minting.md) |
| `invalid_id`, `invalid_org_id`, `invalid_template_id`, `invalid_upload_id`, `invalid_chunk_index`, `invalid_num_mints`, `invalid_total_bytes`, `invalid_private_file_sizes`, `invalid_owner`, `invalid_nonce`, `invalid_json`, `invalid_template_json` | 400 | A field did not parse, or is out of range | [Reference](reference.md) |
| `too_many_items`, `no_items`, `nothing_to_update`, `too_many_reused_mint_requests`, `unknown_category`, `duplicate_category`, `unknown_certificate_type` | 400 | Empty, oversized, or naming something that does not exist | [Minting](../minting-studio/minting.md) |
| `bad_multipart`, `missing_file` | 400 | A multipart upload had no readable `file` part | [Reference](reference.md) |
| `missing_private`, `not_private`, `private_in_data`, `private_not_allowed`, `invalid_reader`, `not_a_private_upload`, `upload_not_finalized`, `private_file_from_another_collection`, `reserved_path`, `private_flag_removed`, `not_an_org_collection` | 400 | The private part or upload does not match its template, collection or endpoint | [Validation](../private-content/minting.md#validation-rules) |
| `insufficient_allowance`, `insufficient_funds` | 402 | The billing principal's approval, or its balance, is too low | [Billing](../minting-studio/organizations.md#paying-the-billing-principal) |
| `not_delegable`, `not_authorized`, `no_delegation` | 403 | The gateway's on-chain authorization is missing or expired | [Keys](api-keys.md#withdrawing-the-gateways-authorization) |
| `not_permitted`, `not_a_member`, `insufficient_role`, `not_org_owner`, `org_mismatch` | 403 | Membership, role, or the key's organization. Only `insufficient_role` is certainly a role | [Roles](../minting-studio/organizations.md#roles) |
| `org_suspended`, `not_owner` | 403 | Ownership, role, or suspension; on the mint path `not_owner` covers all three | [Suspension](../minting-studio/organizations.md#suspension) |
| `file_not_uploaded` | 403 | A file that is not an upload of this collection | [Minting](../minting-studio/minting.md) |
| `not_found`, `collection_not_found`, `org_not_found`, `group_not_found`, `member_not_found`, `indexer_error` | 404, or 400 for a query the index refused | No such record | [Reference](reference.md) |
| `idempotency_key_conflict` | 409 | Same key, different body. Never retry it | [Paid](paid-requests.md) |
| `concurrent_request` | 409 | That key's first call is still running. Retry it | [Paid](paid-requests.md) |
| `collection_not_ready`, `mint_not_active`, `mint_limit_exceeded`, `not_refundable`, `already_refunded`, `credits_used` | 409 | The collection or mint request is in the wrong state | [Lifecycle](../minting-studio/collections-and-certificates.md#collection-lifecycle) |
| `template_in_use`, `template_limit_exceeded`, `too_many_template_versions` | 409 | A template limit; the message names the cap | [Versions](../minting-studio/templates.md#template-versions) |
| `collection_not_indexed`, `collection_org_mismatch` | 409 | The gateway has not caught up with that collection. Retry shortly | [Private content](../private-content/overview.md) |
| `json_too_large`, `chunk_too_large`, `byte_limit_exceeded`, `file_too_large` | 413 | A certificate, chunk, reservation or logo over its limit | [Uploads](../private-content/minting.md) |
| `rate_limited` | 429 | Invitations (200 per organization per day) or verification email (5 per hour) | [Limits](overview.md#limits) |
| `internal` | 500 | The gateway's fault, not your request | [Errors](overview.md#errors) |
| `canister_error`, `canister_unavailable`, `upstream_unavailable`, `key_unavailable`, `vetkd_unavailable`, `mint_error`, `upload_error`, `update_failed`, `transfer_from_error`, `broken_template` | 502 | A downstream canister, ledger, key service or index failed. Retry | [Errors](overview.md#errors) |
| `price_unavailable`, `pricing_not_configured` | 503 | The OGY price or mint pricing is briefly unavailable. Retry | [Paid](paid-requests.md) |
