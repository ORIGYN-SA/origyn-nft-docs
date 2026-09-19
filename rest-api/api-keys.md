---
icon: key
---

# Obtaining an API Key

An API key lets you drive the Minting Studio over plain HTTP, with no Internet Computer tooling. It is the recommended way to integrate, and currently the only way to use [private content](../private-content/overview.md).

## Before you start

**Membership in an organization.** Collections, templates and mints belong to [organizations](../minting-studio/organizations.md). Apply for your own in the [Minting Studio dashboard](https://minting.origyn.com) (see [Getting Started](../minting-studio/getting-started.md#1-apply-for-access)), or accept an invitation from an organization you work with. Until you are an active member, every call that creates or mints is refused.

**OGY on the billing principal.** Collections and mints are charged to your organization's **billing principal** (the Owner, unless the Owner named another wallet), which must approve the Minting Studio to spend its OGY. See [Paying: the billing principal](../minting-studio/organizations.md#paying-the-billing-principal).

## Create your key in the dashboard

{% stepper %}
{% step %}
### Sign in

Go to [minting.origyn.com](https://minting.origyn.com) and connect with Internet Identity. Signing in authorizes the dashboard's own session for 24 hours.
{% endstep %}

{% step %}
### Open the API keys page

Select **API keys** in the sidebar.

<!-- SCREENSHOT 1: API keys page, empty state -->
{% endstep %}

{% step %}
### Start key creation

Click **Create new key**. The dialog tells you whether your wallet will ask you to confirm **twice**, or only **once** if you already have an OGY approval of at least 100,000 OGY in place.

<!-- SCREENSHOT 2: "Create your API key" dialog showing its four steps -->
{% endstep %}

{% step %}
### Let Minting Studio pay from your wallet

Sign the OGY approval in your wallet. It lets the Minting Studio charge your wallet, up to 1,000,000,000 OGY, for the collections and mints you pay for. Skipped automatically when your existing approval is sufficient.

{% hint style="info" %}
This approves **your** wallet. If someone else is your organization's billing principal, the organization's charges come from **their** approval, not this one.
{% endhint %}
{% endstep %}

{% step %}
### Sign the one-time code

Sign the second prompt within 5 minutes. It records on chain that you authorize the ORIGYN gateway to act on your behalf, and it is what ties the key to your principal. The authorization stays in place until you withdraw it; see [Withdrawing the gateway's authorization](#withdrawing-the-gateways-authorization).
{% endstep %}

{% step %}
### Copy your key

Your key is shown once and never again. Copy it, store it somewhere safe such as a password manager, then tick **I have copied and stored this key**.

Keys look like `sk_live_` followed by 32 hexadecimal characters.

<!-- SCREENSHOT 3: "Copy your API key now" dialog, KEY VALUE REDACTED -->
{% endstep %}
{% endstepper %}

Keys created in the dashboard belong to your principal and are not bound to one organization. To bind a key to an organization, create it [programmatically](#create-a-key-programmatically).

## Managing your keys

Everything after creation happens on the same dashboard page, which uses your signed-in wallet session rather than a key.

**Your keys** are listed with their status, **Active** or **Revoked**. The key value itself is never shown again.

<!-- SCREENSHOT 4: keys table with an active key -->

**Revoking** takes effect immediately; any software still sending that key stops working. You can hold several keys at once, which is the clean way to rotate: create the new one, deploy it, then revoke the old one.

<!-- SCREENSHOT 5: "Revoke key" dialog -->

**Lost a key?** Sign in to the dashboard and revoke it there. You do not need a working key to manage your keys.

**Your allowance** is a finite budget, not open-ended permission. When your approval drops below 100,000 OGY the page shows a card to renew it.

<!-- SCREENSHOT 6: "Renew what Minting Studio can spend" allowance card -->

{% hint style="warning" %}
Check the paying allowance before a large minting run. If it runs out, `initialize_mint` fails and reserves nothing, so top up and call it again. `GET /orgs/{org_id}/allowance` always shows the approval that actually pays for your organization.
{% endhint %}

Over HTTP, the same management endpoints accept an API key or a session token:

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/keys" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/keys/{id}" method="delete" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

`GET /keys` lists every key of your principal, revoked ones included, newest first.

## Create a key programmatically

Key creation needs no existing credential. You prove who you are by signing a one-time code on chain.

**1. Get a code.**

```bash
curl -X POST https://gateway.origyn.com/gateway/v1/nft/production/auth/challenge
```

```json
{ "nonce": "3f1c6a52-8e0b-4d7a-9b61-2f4c9d0e7a18" }
```

**2. Sign it on chain** with the identity the key should belong to, within 5 minutes:

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai prove_principal \
  '("3f1c6a52-8e0b-4d7a-9b61-2f4c9d0e7a18", null)'
```

The second argument sets how long the gateway's authorization lasts. `null` grants it until you withdraw it; `opt <nanoseconds>` makes it expire after that duration, at most 30 days. When authorizations overlap, the longer one wins.

**3. Exchange the code for a key.**

```bash
curl -X POST https://gateway.origyn.com/gateway/v1/nft/production/keys \
  -H "Content-Type: application/json" \
  -d '{ "nonce": "3f1c6a52-8e0b-4d7a-9b61-2f4c9d0e7a18", "org_id": 12 }'
```

```json
{ "api_key": "sk_live_…" }
```

Each code works once. `org_id` is optional:

| | Without `org_id` | With `org_id` |
| --- | ---------------- | ------------- |
| Creating a template or collection without `org_id` in the body | Goes to the organization you own | Goes to the key's organization |
| Naming a different `org_id` in the body | Allowed if you are a member | `403 org_mismatch` |
| `GET /allowance` reports | Your own approval | The organization's billing principal's approval |
| Requirement | none | You must be an active member (`403 not_a_member`) |

The binding is permanent for the life of the key.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/keys" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

## Session tokens

For short-lived tools, exchange a signed code for a session token instead of a key. Repeat steps 1 and 2 above with a fresh code, then:

```bash
curl -X POST https://gateway.origyn.com/gateway/v1/nft/production/auth/session \
  -H "Content-Type: application/json" \
  -d '{ "nonce": "<fresh_nonce>" }'
```

```json
{ "token": "<jwt>", "expires_at": 1789640000, "principal": "<your_principal>" }
```

| | API key | Session token |
| --- | ------- | ------------- |
| Lifetime | Until revoked | At most 24 hours, and never past your authorization's expiry. `expires_at` is in **seconds**. |
| Organization binding | Optional | Never |
| Accepted by | Every authenticated endpoint | Every authenticated endpoint |
| Requires | A signed code | A signed code and a live authorization (`403 no_delegation` otherwise) |

Send it the same way, `Authorization: Bearer <token>`. There is no refresh endpoint; request a new token when it expires.

## Withdrawing the gateway's authorization

The authorization you signed belongs to your **principal**, not to any key. Revoking a key does not withdraw it. To withdraw it, call the Minting Studio canister with the same identity:

```bash
# Check it: null means none; expires_at = null means it does not expire
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai get_gateway_delegation \
  '(principal "<your_principal>")'

# Withdraw it
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai revoke_gateway_delegation '()'
```

Once withdrawn, your keys still authenticate and reads keep working, but **every write fails** until you authorize again:

| Call | Result |
| ---- | ------ |
| `create_collection`, `initialize_mint` | `403 not_delegable` |
| Every other write | `403 not_authorized` |
| `POST /auth/session` | `403 no_delegation` |

Signing a new code with `prove_principal` restores writes for all your keys. Creating a key in the dashboard does this for you.

## Using your key

Send it as a bearer token on every request:

```
Authorization: Bearer sk_live_...
```

Keep it in an environment variable rather than in source control.

Three things worth knowing:

* **The key authenticates you, and only you.** Your principal is resolved from the key on every request and is never read from a URL, body, or header.
* **Your organization owns and pays.** What you create belongs to the organization, collection and mint fees are charged to its billing principal, and refunds return there. The gateway only pays the Internet Computer cycles.
* **A key only works in its own environment.** A production key (`sk_live_`) is rejected anywhere else.

## Next steps

* [REST API Overview](overview.md) for base URLs, pagination, sorting, and errors.
* [Paid Requests & Idempotency](paid-requests.md) for the two endpoints that spend OGY.
* [Organizations & Members](../minting-studio/organizations.md) for roles, invitations and billing.
* [Endpoint Reference](reference.md) to call any endpoint directly.
