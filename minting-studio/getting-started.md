---
icon: bolt
---

# Getting Started

This guide walks you through launching an ORIGYN NFT collection using the Minting Studio. It is a managed service where ORIGYN handles the infrastructure. If you prefer full control over your smart contracts, see [Custom Installation](../custom-installation/setup.md) instead.

{% hint style="success" %}
**We recommend working over HTTP with an API key.** It needs no Internet Computer tooling, and it is currently the only way to use [private content](../private-content/overview.md). Step 2 explains the difference between the two ways in.
{% endhint %}

---

### 1. Apply for access

The Minting Studio issues official ORIGYN certificates, so every issuer is validated by the ORIGYN Foundation first.

1. Go to [minting.origyn.com](https://minting.origyn.com) and sign in with Internet Identity, NFID Wallet or OISY. The wallet you sign in with is the principal your organization will belong to.
2. Fill in the application form: your company name, contact details, country, website, registration number, industry, and how you will use ORIGYN.
3. Submit it. The ORIGYN team reviews it; the dashboard shows the status, and you can **Refresh status** while you wait.

When your application is approved, your **organization** is created on chain and your wallet is its **Owner**. If the team asks for changes, the dashboard shows their feedback and lets you update and resubmit the form.

{% hint style="info" %}
Joining a company that already uses the Minting Studio? You do not need to apply. Ask an Owner or Admin of that organization to invite you. See [Organizations & Members](organizations.md#invitations).
{% endhint %}

### 2. Choose how you will integrate

Everything the Minting Studio does can be driven two ways. Both act as your principal, reach the same canisters and produce identical collections. The difference is **how you prove who you are** and **what you have to run**.

| | API key over HTTP (recommended) | `dfx` (direct canister calls) |
| --- | ------------------------------- | ----------------------------- |
| What you call | `https://gateway.origyn.com`, with any HTTP client | The Minting Studio canister `uasjq-dyaaa-aaaas-qdwka-cai`, with `dfx` or an agent library |
| How you are identified | An API key in the `Authorization` header | Every call is signed with the private key of an Internet Computer identity |
| Who acts | The ORIGYN gateway calls the canister for you, after you authorize it once on chain | You call the canister yourself |
| Setup | Create a key once in the dashboard | Install `dfx` and make its identity a member of your organization |
| Paying OGY | The approval is signed in your wallet while you create the key | You sign `icrc2_approve` yourself |
| Private content | Yes | No |
| Secret to protect | The key | The identity's private key |

With an API key, your wallet's private key never leaves your wallet. The one-time authorization you sign lets the ORIGYN gateway act for you on the Minting Studio canister, and you can [revoke the key](../rest-api/api-keys.md#managing-your-keys) or [withdraw that authorization](../rest-api/api-keys.md#withdrawing-the-gateways-authorization) at any time.

You are not locked in. A collection created over HTTP is an ordinary ORIGYN NFT canister you can also call with `dfx`, and the other way round.

### 3. Prepare your environment

Whichever way you choose, collection and mint fees are charged to your organization's **billing principal**, which is the Owner's wallet unless the Owner names another one. Make sure it holds at least **15,000 OGY** for the collection creation fee, plus whatever you plan to mint.

{% tabs %}
{% tab title="API key (recommended)" %}
**1. Create your API key.** In the dashboard, open **API keys**, click **Create new key**, and follow the dialog:

* approve the Minting Studio to spend OGY from your wallet (skipped if you already approved enough),
* sign a one-time code, which authorizes the ORIGYN gateway to act for you,
* copy the key. It is shown only once.

The full walkthrough, including creating keys without the dashboard, is in [Obtaining an API Key](../rest-api/api-keys.md).

**2. Keep the key out of your code.**

```bash
export ORIGYN_API_KEY="sk_live_..."
```

**3. Check it works**, and note your organization id:

```bash
curl https://gateway.origyn.com/gateway/v1/nft/production/me \
  -H "Authorization: Bearer $ORIGYN_API_KEY"
```

The response lists your principal and, under `orgs`, each organization you belong to with its `org_id` and your `role`.

{% hint style="info" %}
The key-creation dialog approves OGY from **your** wallet. If someone else is your organization's billing principal, that person must approve the Minting Studio from their own wallet. Check the approval that actually pays with `GET /orgs/{org_id}/allowance`.
{% endhint %}
{% endtab %}

{% tab title="dfx" %}
**1. Install dfx** by following the [official setup guide](https://internetcomputer.org/docs/current/developer-docs/getting-started/install/), then print the principal of your identity:

```bash
dfx identity get-principal
```

**2. Make that identity a member of your organization.** A `dfx` identity has a **different principal** from the wallet you signed in to the dashboard with, so it is not in your organization yet. Invite it:

1. In the dashboard, as Owner, invite the `dfx` principal with the role it needs (Admin to create templates and collections, Minter to mint). See [Inviting a member](organizations.md#inviting-a-member).
2. Accept the invitation with the `dfx` identity, using your organization id:

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai get_pending_invites '(record {
  "principal" = principal "<dfx_principal>"
})'

dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai accept_invite '(record { org_id = <org_id> : nat64 })'
```

Because this identity does not **own** the organization, pass `org_id = opt <org_id>` whenever a call takes `org_id`.

**3. Approve the fee payment** from the billing principal. OGY has 8 decimals, so 15,000 OGY is `1_500_000_000_000` e8s, and ICRC-2 also debits the ledger transfer fee (`200_000` e8s) from the allowance:

```bash
# Run with the billing principal's identity
dfx canister --network ic call lkwrt-vyaaa-aaaaq-aadhq-cai icrc2_approve '(record {
  amount = 1_500_000_200_000;
  spender = record { owner = principal "uasjq-dyaaa-aaaas-qdwka-cai"; }
})'
```

If the billing principal is the Owner's dashboard wallet, approve there instead: the dashboard asks that Owner to **Approve OGY spending** when no approval is in place. Approve more if you plan to create several collections or mint soon after; the approval is a spending ceiling, not a payment.
{% endtab %}
{% endtabs %}

### 4. Create a template

A template defines the structure of your certificates. The easiest way to build one is the [Visual Template Builder](https://arturshirokov.github.io/claimlink-template-builder/), a drag-and-drop tool that generates the JSON for you. See [Templates](templates.md) for the full structure and field types.

Register the template JSON. `template_json` is the template as a **string**:

```bash
curl -X POST https://gateway.origyn.com/gateway/v1/nft/production/create_template \
  -H "Authorization: Bearer $ORIGYN_API_KEY" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --rawfile t template.json --argjson org 12 '{ template_json: $t, org_id: $org }')"
```

```json
{ "template_id": "1", "version": 1, "current_version": 1, "template_url": "https://...", "template": { ... } }
```

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/create_template" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

**Using dfx instead**

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai create_template '(record {
  org_id = opt <org_id>;
  template_json = "<your_template_json_here>"
})'
```

`org_id` can be left out over HTTP (or `null` over `dfx`) only when you **own** the organization. Creating templates takes the Owner or Admin role.

Note the `template_id`; you need it in the next step.

### 5. Create the collection

{% hint style="danger" %}
**This spends 15,000 OGY** from the billing principal. The **Test it** button below sends a real production request, so fill in `Idempotency-Key` deliberately. See [Paid Requests & Idempotency](../rest-api/paid-requests.md).
{% endhint %}

```bash
curl -X POST https://gateway.origyn.com/gateway/v1/nft/production/create_collection \
  -H "Authorization: Bearer $ORIGYN_API_KEY" \
  -H "Idempotency-Key: $(uuidgen)" \
  -H "Content-Type: application/json" \
  -d '{
        "org_id": <org_id>,
        "template_id": 1,
        "name": "My Unique Collection",
        "symbol": "MUC",
        "description": "A collection of rare digital artifacts.",
        "categories": []
      }'
```

```json
{ "request_id": "1042", "status": "Queued" }
```

Creation runs in the background, typically in under a minute. Poll until `status` is `TemplateUploaded` and `canister_id` is set; that canister id is what minting needs.

```bash
curl https://gateway.origyn.com/gateway/v1/nft/production/collections/1042/status
```

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/create_collection" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

**Using dfx instead**

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai create_collection '(record {
  org_id = opt <org_id>;
  categories = vec {};
  name = "My Unique Collection";
  description = "A collection of rare digital artifacts.";
  symbol = "MUC";
  template_id = 1;
  certificate_type = null
})'
```

This returns a `collection_id`. Poll `get_collection_info` with it until the status is `TemplateUploaded`:

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai get_collection_info '(variant { CollectionId = <collection_id> })'
```

A few rules for both ways:

* The template must belong to the same organization as the collection. Creating collections takes the Owner or Admin role.
* Every category name must already exist in the global taxonomy, otherwise the call fails with `UnknownCategory` before any OGY is charged. Read the list with `GET /v1/nft/production/categories/catalog` or `list_categories`.
* Over `dfx`, `categories` is **required** even when empty (`vec {}`). Over HTTP it may be omitted.

{% hint style="warning" %}
A `Failed` status does **not** mean your fee has been refunded. Creation is retried automatically roughly once a minute, and a reimbursement is only requested once the retry limit is exhausted. See [Collection Lifecycle](collections-and-certificates.md#collection-lifecycle).
{% endhint %}

---

### What's Next?

Your collection is now live. Here's what to do next:

- **[Minting](minting.md)** Mint certificates into your collection with the full API flow
- **[Organizations & Members](organizations.md)** Invite your team, assign roles, and manage billing
- **[Templates](templates.md)** Learn more about template structure, field types, versions, and the visual builder
- **[Private Content](../private-content/overview.md)** Add fields and files only chosen readers can see
- **[Collections & Certificates](collections-and-certificates.md)** Understand the collection lifecycle and how to query certificates
- **[Managing Collections](managing-collections.md)** Edit metadata, set a logo, and settle mint requests
