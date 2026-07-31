---
icon: key
---

# Obtaining an API Key

An API key lets you drive the Minting Studio over plain HTTP, with no Internet Computer tooling.

You create and manage it in the Minting Studio dashboard. That is the only way to get one.

## Before you start

**An authorized principal.** The Minting Studio is a permissioned environment. If your identity has not been approved you will see an access warning when you sign in. You can still create a key, but every call that creates a collection or mints will be rejected, so the key is useless until access is granted.

Request access by emailing [admin@origyn.ch](mailto:admin@origyn.ch?subject=Minting%20Studio%20Access%20Request) with a short description of your project and your principal.

**OGY in your wallet.** Collections and mints are paid for in OGY, charged from your own account. You approve a spending allowance as part of creating the key.

{% hint style="info" %}
Approval is manual and there is no self-service path, so request access early.
{% endhint %}

## Create your key

{% stepper %}
{% step %}
### Sign in

Go to [minting.origyn.com](https://minting.origyn.com) and connect with Internet Identity.
{% endstep %}

{% step %}
### Open the API Keys page

Select **API Keys** in the sidebar.

<!-- SCREENSHOT 1: API Keys page, empty state -->
{% endstep %}

{% step %}
### Start key creation

Click **Create API Key**. The dialog tells you how many wallet signatures to expect: **twice** normally, or **once** if you already have a sufficient OGY allowance in place.

<!-- SCREENSHOT 2: create-key dialog showing its steps -->
{% endstep %}

{% step %}
### Approve OGY spending

Sign the approval in your wallet. This lets the Minting Studio charge your account for the collections and mints you later pay for.

Skipped automatically if an earlier approval is still sufficient.
{% endstep %}

{% step %}
### Prove your identity

Sign the second prompt. This records on-chain that you consent to the gateway acting on your behalf, and it is what ties the key to your principal.
{% endstep %}

{% step %}
### Copy your key

Your key is shown once and never again. Copy it now and store it somewhere safe.

Keys look like `sk_live_` followed by 32 hexadecimal characters.

<!-- SCREENSHOT 3: created-key dialog, KEY VALUE REDACTED -->

{% hint style="danger" %}
The key cannot be recovered. If you lose it, create a new key first, then use the new key to revoke the lost one.
{% endhint %}
{% endstep %}
{% endstepper %}

## Managing your keys

Everything after creation happens on the same dashboard page.

**Your keys** are listed with their id, creation date, and status. The key value itself is never shown again.

<!-- SCREENSHOT 4: keys table with an active key -->

**Revoking** takes effect immediately. You can hold several keys at once, which is the clean way to rotate: create the new one, deploy it, then revoke the old one.

<!-- SCREENSHOT 5: revoke-key dialog -->

**Your allowance** is a finite budget, not open-ended permission. When it runs low, top it up from the same page. The key keeps working throughout; only the spending budget needs renewing.

<!-- SCREENSHOT 6: allowance card -->

{% hint style="warning" %}
Check your allowance before a large minting run. If it runs out, `initialize_mint` fails and reserves nothing, so top up and call it again.
{% endhint %}

## Using your key

Send it as a bearer token on every request:

```
Authorization: Bearer sk_live_...
```

Keep it in an environment variable rather than in source control.

Three things worth knowing:

* **The key authenticates you, and only you.** Your principal is resolved from the key on every request and is never read from a URL, body, or header.
* **You stay the owner and the payer.** Collections you create belong to your principal and refunds return to your account. The gateway only pays the Internet Computer cycles.
* **Revoking a key does not withdraw your on-chain consent.** That consent is permanent. Revoking the key is the control you have.

## Next steps

* [REST API Overview](overview.md) for base URLs, pagination, sorting, and errors.
* [Paid Requests & Idempotency](paid-requests.md) for the two endpoints that spend OGY.
* [Endpoint Reference](reference.md) to call any endpoint directly.
