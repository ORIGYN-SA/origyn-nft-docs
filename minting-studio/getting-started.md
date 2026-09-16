---
icon: bolt
---

# Getting Started

This guide walks you through launching an ORIGYN NFT collection using the Minting Studio. It is a managed service where ORIGYN handles the infrastructure. If you prefer full control over your smart contracts, see [Custom Installation](../custom-installation/setup.md) instead.

{% hint style="success" %}
**We recommend working over HTTP with an API key.** It needs no Internet Computer tooling, and it is currently the only way to use [private content](../private-content/overview.md). After applying for access (step 1), continue with [Obtaining an API Key](../rest-api/api-keys.md) and the [REST API Overview](../rest-api/overview.md).

The steps below show the HTTP endpoint where one exists, followed by the equivalent `dfx` command. Both reach the same canisters and produce identical collections.
{% endhint %}

### Prerequisites

To follow the `dfx` commands, you need the Internet Computer SDK (dfx) installed and a developer identity configured.

First, install DFX by following the [official setup guide](https://internetcomputer.org/docs/current/developer-docs/getting-started/install/). Once installed, verify your identity by running

```bash
dfx identity get-principal
```

in your terminal. This is the identity that will act in your organization.

---

### 1. Apply for access

The Minting Studio issues official ORIGYN certificates, so every issuer is validated by the ORIGYN Foundation first.

1. Go to [minting.origyn.com](https://minting.origyn.com) and sign in with Internet Identity, using the identity you want to work with.
2. Fill in the application form: your company name, contact details, country, website, registration number, industry, and how you will use ORIGYN.
3. Submit it. The ORIGYN team reviews it; the dashboard shows the status, and you can **Refresh status** while you wait.

When your application is approved, your **organization** is created on chain and you are its **Owner**. If the team asks for changes, the dashboard shows their feedback and lets you update and resubmit the form.

{% hint style="info" %}
Joining a company that already uses the Minting Studio? You do not need to apply. Ask an Owner or Admin of that organization to invite you. See [Organizations & Members](organizations.md#invitations).
{% endhint %}

Some calls take your organization id. Look it up over HTTP with `GET /orgs`, or over `dfx`:

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai get_orgs_for_principal '(record {
  "principal" = principal "<your_principal>"
})'
```

### 2. Prepare Your Environment

Collection and mint fees are charged to your organization's **billing principal**, which is you (the Owner) unless you name another wallet. Make sure it holds at least 15,000 OGY to cover the collection creation fee.

You will be interacting with the Minting Studio canister (**`uasjq-dyaaa-aaaas-qdwka-cai`**).

### 3. Create a Metadata Template

You must first define the structure of your NFTs by registering a JSON template. The easiest way is to use the [Visual Template Builder](https://ahegaoburger.github.io/claimlink-template-builder/) the drag-and-drop tool that generates the JSON for you. See the [Templates](templates.md) page for full details on template structure and field types.

Once you have your template JSON, register it:

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/create_template" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

**Using dfx instead**

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai create_template '(record {
  org_id = null;
  template_json = "<your_template_json_here>"
})'
```

`org_id = null` creates the template in the organization you own. If you work in someone else's organization, pass `org_id = opt <their_org_id>` instead (and `"org_id"` in the HTTP body).

Note the template_id returned by this command (e.g., 1), as you will need it shortly.

### 4. Approve Fee Payment

The **billing principal** authorizes the Minting Studio to spend the required fee. The OGY Ledger ID is `lkwrt-vyaaa-aaaaq-aadhq-cai`. If you create an API key in the dashboard, this approval is part of key creation and you can skip this step.

OGY has 8 decimals, so 15,000 OGY is `1_500_000_000_000` e8s. ICRC-2 also debits the ledger transfer fee (`200_000` e8s) from the allowance, so approve `1_500_000_200_000`.

```bash
# Run with the billing principal's identity
dfx canister --network ic call lkwrt-vyaaa-aaaaq-aadhq-cai icrc2_approve '(record {
  amount = 1_500_000_200_000;
  spender = record { owner = principal "uasjq-dyaaa-aaaas-qdwka-cai"; }
})'
```

Approve more if you plan to create several collections or mint soon after; the approval is a spending ceiling, not a payment.

### 5. Create the Collection

Submit the final request to spin up your NFT canister. Replace `template_id = 1` with the actual ID you received in Step 3.

{% hint style="danger" %}
**This spends 15,000 OGY.** Test it sends a real production request. Fill in `Idempotency-Key` deliberately.
{% endhint %}

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/create_collection" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

**Using dfx instead**

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai create_collection '(record {
  org_id = null;
  categories = vec {};
  name = "My Unique Collection";
  description = "A collection of rare digital artifacts.";
  symbol = "MUC";
  template_id = 1;
  certificate_type = null
})'
```

The template must belong to the same organization as the collection. Only an Owner or Admin can create collections.

Any category name you pass must already exist in the global taxonomy, otherwise the call fails with `UnknownCategory` before any OGY is charged. Read the current list with `list_categories`.

{% hint style="info" %}
Over `dfx`, `categories` is **required** and must be present even when empty (`vec {}`). Over REST it is optional and may be omitted entirely.
{% endhint %}

This command returns a `collection_id`. Monitor the installation, which typically completes in under a minute, by querying `get_collection_info` with this ID. Look for the status `TemplateUploaded` to confirm success.

{% hint style="warning" %}
A `Failed` status does **not** mean your fee has been refunded. Creation is retried automatically roughly once a minute, and a reimbursement is only requested once the retry limit is exhausted. See [Collection Lifecycle](collections-and-certificates.md#collection-lifecycle).
{% endhint %}

---

### What's Next?

Your collection is now live. Here's what to do next:

- **[Organizations & Members](organizations.md)** Invite your team, assign roles, and manage billing
- **[Templates](templates.md)** Learn more about template structure, field types, versions, and the visual builder
- **[Collections & Certificates](collections-and-certificates.md)** Understand the collection lifecycle and how to query certificates
- **[Minting](minting.md)** Mint certificates into your collection with the full API flow
- **[Managing Collections](managing-collections.md)** Edit metadata, set a logo, and settle mint requests
- **[Private Content](../private-content/overview.md)** Add fields and files only chosen readers can see
