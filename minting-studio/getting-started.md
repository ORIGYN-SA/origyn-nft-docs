---
icon: bolt
---

# Getting Started

This guide walks you through launching an ORIGYN NFT collection using the Minting Studio — a managed service where ORIGYN handles the infrastructure. If you prefer full control over your smart contracts, see [Custom Installation](../custom-installation/setup.md) instead.

### Prerequisites

Before beginning, you must have the Internet Computer SDK (dfx) installed and a developer identity configured.

First, install DFX by following the [official setup guide](https://internetcomputer.org/docs/current/developer-docs/getting-started/install/). Once installed, verify your identity by running

```bash
dfx identity get-principal
```

in your terminal. Save this Principal ID, as it is required for authorization and ownership.

***

### 1. Request Authorization

The Minting Studio is a permissioned environment. To gain access, send an email to [admin@origyn.ch](mailto:admin@origyn.ch) including a brief description of your project and the Principal ID you retrieved in the prerequisites step. You must wait for confirmation that your address has been whitelisted before proceeding.

### 2. Prepare Your Environment

Ensure your wallet holds at least 15,000 OGY to cover the collection creation fee. You will be interacting with the Minting Studio canister (**`uasjq-dyaaa-aaaas-qdwka-cai`**).

### 3. Create a Metadata Template

You must first define the structure of your NFTs by registering a JSON template. The easiest way is to use the [Visual Template Builder](https://ahegaoburger.github.io/claimlink-template-builder/) — a drag-and-drop tool that generates the JSON for you. See the [Templates](templates.md) page for full details on template structure and field types.

Once you have your template JSON, register it:

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai create_template '(record {
 template_json = "<your_template_json_here>"
})'
```

Note the template\_id returned by this command (e.g., 1), as you will need it shortly.

### 4. Approve Fee Payment

Authorize the Minting Studio to spend the required fee from your wallet. The OGY Ledger ID is `lkwrt-vyaaa-aaaaq-aadhq-cai`.

```bash
dfx canister --network ic call lkwrt-vyaaa-aaaaq-aadhq-cai icrc2_approve '(record {
  amount = 15000000000;
  spender = record { owner = principal "uasjq-dyaaa-aaaas-qdwka-cai"; }
})'
```

### 5. Create the Collection

Submit the final request to spin up your NFT canister. Replace `template_id = 1` with the actual ID you received in Step 3.

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai create_collection '(record {
  name = "My Unique Collection";
  description = "A collection of rare digital artifacts.";
  symbol = "MUC";
  template_id = 1;
})'
```

This command returns a `collection_id`. You can monitor the installation progress (which typically takes under a minute) by querying `get_collection_info` with this ID. Look for the status `TemplateUploaded` to confirm success. If the status is `Failed`, the protocol automatically reimburses the 15,000 OGY fee.

***

### What's Next?

Your collection is now live. Here's what to do next:

* **[Templates](templates.md)** — Learn more about template structure, field types, and the visual builder
* **[Collections & Certificates](collections-and-certificates.md)** — Understand the collection lifecycle and how to query certificates
* **[Minting](minting.md)** — Mint certificates into your collection with the full API flow
