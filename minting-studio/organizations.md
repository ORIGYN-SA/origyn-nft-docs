---
icon: building
---

# Organizations & Members

Everything you create in the Minting Studio belongs to an **organization**: its templates, its collections, the files uploaded into them, and the mint requests that fill them. People work inside an organization as **members**, each with a **role**, and the organization pays for what they do through its **billing principal**.

## Getting an organization

There are two ways in.

**Apply for your own.** Sign in to the [Minting Studio dashboard](https://minting.origyn.com) and fill in the application form. The ORIGYN team reviews it; once approved, your organization is created on chain and you are its **Owner**. See [Getting Started](getting-started.md#1-apply-for-access).

**Join an existing one.** An Owner or Admin invites you, either by principal or by email. See [Invitations](#invitations).

{% hint style="info" %}
If you had Minting Studio access before organizations existed, you already own an organization that holds your existing collections and templates. Your existing API keys keep working.
{% endhint %}

## Roles

Every member has exactly one role.

| Permission | Owner | Admin | Minter | Viewer |
| ---------- | :---: | :---: | :----: | :----: |
| Create collections (spends the organization's OGY) | ✓ | ✓ | | |
| Create, edit and delete templates | ✓ | ✓ | | |
| Mint: open mint requests, upload files, mint, settle and refund (spends OGY) | ✓ | ✓ | ✓ | |
| Edit collection metadata, categories and logo | ✓ | ✓ | ✓ | |
| Invite, re-role and remove Minters and Viewers | ✓ | ✓ | | |
| Manage [reader groups](../private-content/reader-groups.md) | ✓ | ✓ | | |
| Invite, re-role and remove Admins | ✓ | | | |
| Change the billing principal | ✓ | | | |
| Transfer ownership | ✓ | | | |
| Read the organization and its [private content](../private-content/overview.md) | ✓ | ✓ | ✓ | ✓ |

A few rules that follow from this:

* **There is exactly one Owner.** The Owner role cannot be invited, assigned, removed or demoted; it only moves with [a transfer](#transferring-ownership).
* **A principal can own one organization**, and be a member of any number of others.
* **Members work on shared resources.** Any Minter can continue a mint request a colleague opened, and any member's uploads can be attached to a certificate in the same collection.
* **There is no "leave" action.** To leave an organization, ask an Owner or Admin to remove you.

## Choosing the organization you act for

Two calls create something new and so need to know which organization it belongs to: creating a **template** and creating a **collection**. Both take an optional `org_id`.

| You send | The organization used |
| -------- | --------------------- |
| `org_id` | That organization. You must be an active member with the right role. |
| No `org_id`, with an organization-bound API key | The key's organization |
| No `org_id` otherwise | The organization **you own**. If you own none, the call is refused. |

{% hint style="warning" %}
**Admins, Minters and Viewers of someone else's organization must pass `org_id`.** Without it the Minting Studio looks for an organization you own, which is not the one you work in.
{% endhint %}

Every other write takes its organization from what it names: a template, a collection, or a mint request.

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai create_collection '(record {
  org_id = opt 12;
  categories = vec {};
  name = "My Gold Bar Collection";
  description = "Certified gold bar certificates";
  symbol = "GBC";
  template_id = 1;
  certificate_type = null
})'
```

The template must belong to the same organization as the collection, otherwise the call fails with `InvalidNftTemplateId`.

## Paying: the billing principal

Collection fees and mint fees are charged to the organization's **billing principal**, never to the member making the call. Refunds and reimbursements go back to it too.

* The billing principal defaults to the Owner.
* Only the Owner can change it, and it does not need to be a member.
* It must hold OGY and approve the Minting Studio to spend it (`icrc2_approve` on the OGY ledger, see [Getting Started](getting-started.md#3-prepare-your-environment)). A member's own approval does not pay for the organization.

When the approval runs short, paid calls fail with `402 insufficient_allowance`. Check the organization's standing approval:

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/orgs/{org_id}/allowance" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

The response reports `allowance`, `billing_principal`, `expires_at` and a `state` of `ok`, `warning` (below 20,000 OGY), `empty` or `expired`. When an organization-bound key runs low, Owners, Admins, Minters and the billing principal receive an email at their verified address.

Change the billing principal (Owner only):

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/orgs/{org_id}/billing" method="patch" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

{% hint style="warning" %}
Refunds go to whoever is the billing principal **when the refund is paid out**. Changing it while a mint request is still open redirects that request's refund to the new principal.
{% endhint %}

## Invitations

### Inviting a member

`POST /orgs/{org_id}/invites` takes a `role` (`Viewer`, `Minter` or `Admin`, exact case) and exactly one target:

{% tabs %}
{% tab title="By principal" %}
```bash
curl -X POST https://gateway.origyn.com/gateway/v1/nft/production/orgs/12/invites \
  -H "Authorization: Bearer $ORIGYN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "principal": "<their_principal>", "role": "Minter" }'
```

```json
{ "kind": "direct", "principal": "<their_principal>", "role": "Minter" }
```

The invitee sees the invitation when they sign in, and accepts or declines it. If they have a verified email address they are also notified by email.
{% endtab %}

{% tab title="By email" %}
```bash
curl -X POST https://gateway.origyn.com/gateway/v1/nft/production/orgs/12/invites \
  -H "Authorization: Bearer $ORIGYN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "email": "colleague@example.com", "role": "Viewer" }'
```

```json
{ "kind": "slot", "slot_id": "…", "expires_at": 1790000000000000000, "delivery": "queued" }
```

The address receives a one-time link. Use this for someone who has no Minting Studio account yet.
{% endtab %}
{% endtabs %}

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/orgs/{org_id}/invites" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

Owners and Admins can invite Minters and Viewers; only the Owner can invite an Admin. An organization can send up to 200 invitations a day.

{% hint style="warning" %}
**An email invitation link is a bearer secret.** Whoever opens it first and signs in joins the organization with that role, whatever principal they use. The link is valid for **14 days**, stops working after 5 wrong attempts, and **cannot be revoked**: if it went to the wrong place, remove the member who claims it.
{% endhint %}

List the email invitations still waiting to be claimed (Owner or Admin):

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/orgs/{org_id}/invites" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

Direct invitations appear in the member list with status `Invited` instead.

### Responding to an invitation

| Invitation | How to respond |
| ---------- | -------------- |
| By principal | `GET /invites` lists yours; `POST /invites/{org_id}/accept` or `POST /invites/{org_id}/decline` (no body, `204`) |
| By email | Open the link, or `POST /invites/claim` with `{ "slot_id": "…", "secret": "<64 hex characters>" }` from the link |

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/invites/{org_id}/accept" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/invites/claim" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

Claiming an email invitation also records that address as your verified email, unless another account already uses it.

## Managing members

List members:

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/orgs/{org_id}/members" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

Members with status `Invited` have not accepted yet. Email addresses are shown to Owners and Admins only.

Change a member's role with `{ "role": "Viewer" }`. An Admin can move members between Minter and Viewer; making someone an Admin, or changing an Admin, takes the Owner.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/orgs/{org_id}/members/{principal}" method="patch" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

Remove a member (this also cancels a pending direct invitation):

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/orgs/{org_id}/members/{principal}" method="delete" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

Removal stops on-chain permissions immediately. Access to private content stops once the gateway's read service picks up the change, normally within a minute.

### Transferring ownership

The Owner can hand the organization to another **active** member with `{ "new_owner": "<principal>" }`. The Owner role and the billing principal move to the new Owner, and the previous Owner becomes an Admin. The new Owner must not already own an organization.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/orgs/{org_id}/transfer" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

{% hint style="info" %}
After a transfer, the billing principal is the new Owner. If a different account should keep paying, set it again with `PATCH /orgs/{org_id}/billing`.
{% endhint %}

## Profile and public address

An organization's name and details live in its profile, which Owners and Admins edit:

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/orgs/{org_id}" method="patch" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

The Owner can also claim a **slug**, a readable identity for the organization's public pages:

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/orgs/{org_id}/slug" method="patch" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

| Slug rule | |
| --------- | - |
| Length | 3 to 40 characters |
| Characters | Lowercase letters, digits and `-`; no leading or trailing `-`, no `--`; not all digits |
| Reserved | `admin`, `api`, `me`, `new`, `orgs`, `health`, `healthz`, `v1`, `static`, `assets`, `settings`, `login`, `logout` |
| Changes | 3 changes after the first claim. A slug you give up is never given to another organization, and its old address redirects. |

## Reading an organization's resources

Anyone can read what an organization has issued, by id or slug, with no API key:

| Endpoint | Returns |
| -------- | ------- |
| `GET /v1/nft/production/orgs/{id_or_slug}/collections` | Its collections |
| `GET /v1/nft/production/orgs/{id_or_slug}/nfts` | Its certificates |
| `GET /v1/nft/production/orgs/{id_or_slug}/templates` | Its templates |
| `GET /v1/nft/production/orgs/{id_or_slug}/uploads` | Files uploaded into its collections |

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/orgs/{id_or_slug}/collections" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

With an API key, `GET /gateway/v1/nft/production/orgs/{org_id}/nfts` and `.../orgs/{org_id}/uploads` return the same lists with private content decrypted for you. See [Reading Private Content](../private-content/reading.md).

{% hint style="info" %}
The keyed lists `GET /templates` and `GET /mint_requests` are scoped to **you**, not to your organization: `/templates` shows the templates of the organization you own, and `/mint_requests` shows the requests you opened yourself. Use the organization endpoints above to see everything an organization holds.
{% endhint %}

Your own memberships, with your role in each:

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/orgs" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

## Suspension

ORIGYN can suspend an organization. Its members stay members, but it can no longer spend or change anything except to wind down.

| Still works | Refused |
| ----------- | ------- |
| Settling and refunding open mint requests (`close_mint_request`, `request_mint_refund`) | Creating or editing templates and collections |
| Removing members | Opening mint requests, uploading, minting |
| Changing the billing principal | Inviting members and changing roles |
| Transferring ownership | Managing reader groups |
| All reads | Reading private content, for everyone |

Suspended requests answer `403 org_suspended`. Minting, uploads and `initialize_mint` answer `403 not_owner` instead.

## Errors

| Status | `error` | Meaning |
| ------ | ------- | ------- |
| `400` | `invalid_role`, `owner_role_not_assignable` | Role is not `Viewer`, `Minter` or `Admin` |
| `400` | `invalid_principal`, `anonymous_principal`, `invalid_email`, `invite_target` | Bad invitation target (send exactly one of `principal` or `email`) |
| `400` | `invalid_slug`, `numeric_slug`, `reserved_slug` | Slug breaks a rule above |
| `403` | `not_a_member` | You are not an active member of this organization |
| `403` | `insufficient_role`, `not_permitted` | Your role does not allow this |
| `403` | `org_mismatch` | Your API key is bound to a different organization than `org_id` |
| `403` | `org_suspended` | The organization is suspended |
| `403` | `not_org_owner` | Only the Owner can do this |
| `403` | `not_authorized` | Your principal has not authorized the gateway (see [Obtaining an API Key](../rest-api/api-keys.md)) |
| `404` | `member_not_found` | No such member |
| `409` | `already_invited`, `already_a_member` | Nothing to invite |
| `409` | `cannot_remove_owner`, `cannot_change_owner_role` | Use a transfer instead |
| `409` | `new_owner_not_a_member`, `already_owns_an_org` | The transfer target is not eligible |
| `409` | `slug_taken`, `slug_changes_exhausted` | Pick another slug |
| `410` | `invite_expired` | The email invitation is older than 14 days |
| `429` | `rate_limited` | Too many invitations today |

## Direct canister calls

The organization model lives on the Minting Studio canister, so every read and member action is also available with `dfx`. Timestamps on chain are in nanoseconds.

```bash
# Your memberships
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai get_orgs_for_principal '(record {
  "principal" = principal "<your_principal>"
})'

# An organization and its members
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai get_org '(record { org_id = 12 : nat64 })'
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai get_org_members '(record {
  org_id = 12 : nat64;
  pagination = record { offset = null; limit = null }
})'

# Invite by principal, then the invitee accepts
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai invite_member '(record {
  org_id = 12 : nat64;
  "principal" = principal "<their_principal>";
  role = variant { Minter }
})'
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai accept_invite '(record { org_id = 12 : nat64 })'
```

| Purpose | Methods |
| ------- | ------- |
| Members | `invite_member`, `accept_invite`, `decline_invite`, `get_pending_invites`, `change_member_role`, `remove_member`, `transfer_ownership` |
| Billing | `set_billing_principal` |
| Resources | `get_collections_by_org`, `get_templates_by_org` (at most 2 per call), `get_template_ids_by_org`, `get_mint_requests_by_org` |

Email invitations are sent by the gateway, so `POST /orgs/{org_id}/invites` is the practical way to send one. The slots behind them (`create_invite_slot`, `claim_invite_slot`) are on the canister, but you would have to generate and hash the secret and deliver the link yourself.
