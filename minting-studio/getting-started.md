---
icon: bolt
---

# Getting Started

This guide walks you through launching an ORIGYN NFT collection using the Minting Studio. It is a managed service where ORIGYN handles the infrastructure. If you prefer full control over your smart contracts, see [Custom Installation](../custom-installation/setup.md) instead.

{% hint style="info" %}
**There are two ways to do everything on this page.** This guide uses `dfx`, which talks to the canister directly with your own Internet Computer identity.

If you would rather work over plain HTTP with an API key and no IC tooling, start with the [REST API Overview](../rest-api/overview.md). Both paths reach the same canisters and produce identical collections.
{% endhint %}

### Prerequisites

Before beginning, you must have the Internet Computer SDK (dfx) installed and a developer identity configured.

First, install DFX by following the [official setup guide](https://internetcomputer.org/docs/current/developer-docs/getting-started/install/). Once installed, verify your identity by running

```bash
dfx identity get-principal
```

in your terminal. Save this Principal ID, as it is required for authorization and ownership.

---

### 1. Request Authorization

The Minting Studio is a permissioned environment. To gain access, send an email to [admin@origyn.ch](mailto:admin@origyn.ch) including a brief description of your project and the Principal ID you retrieved in the prerequisites step. You must wait for confirmation that your address has been whitelisted before proceeding.

### 2. Prepare Your Environment

Ensure your wallet holds at least 15,000 OGY to cover the collection creation fee. You will be interacting with the Minting Studio canister (**`uasjq-dyaaa-aaaas-qdwka-cai`**).

### 3. Create a Metadata Template

You must first define the structure of your NFTs by registering a JSON template. The easiest way is to use the [Visual Template Builder](https://ahegaoburger.github.io/claimlink-template-builder/) the drag-and-drop tool that generates the JSON for you. See the [Templates](templates.md) page for full details on template structure and field types.

Once you have your template JSON, register it:

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/create_template" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

**Using dfx instead**

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai create_template '(record {
 template_json = "<your_template_json_here>"
})'
```

Note the template_id returned by this command (e.g., 1), as you will need it shortly.

### 4. Approve Fee Payment

Authorize the Minting Studio to spend the required fee from your wallet. The OGY Ledger ID is `lkwrt-vyaaa-aaaaq-aadhq-cai`.

OGY has 8 decimals, so 15,000 OGY is `1_500_000_000_000` e8s. ICRC-2 also debits the ledger transfer fee (`200_000` e8s) from the allowance, so approve `1_500_000_200_000`.

```bash
dfx canister --network ic call lkwrt-vyaaa-aaaaq-aadhq-cai icrc2_approve '(record {
  amount = 1_500_000_200_000;
  spender = record { owner = principal "uasjq-dyaaa-aaaas-qdwka-cai"; }
})'
```

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
  categories = vec {};
  name = "My Unique Collection";
  description = "A collection of rare digital artifacts.";
  symbol = "MUC";
  template_id = 1
})'
```

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

- **[Templates](templates.md)** Learn more about template structure, field types, and the visual builder
- **[Collections & Certificates](collections-and-certificates.md)** Understand the collection lifecycle and how to query certificates
- **[Minting](minting.md)** Mint certificates into your collection with the full API flow
- **[Managing Collections](managing-collections.md)** Edit metadata, set a logo, and settle mint requests
