---
icon: bolt
metaLinks:
  alternates:
    - https://app.gitbook.com/s/yE16Xb3IemPxJWydtPOj/getting-started/quickstart
---

# Installation

This guide outlines the process for launching an ORIGYN NFT collection. You have two primary options depending on your needs: the Minting Studio API, which is a managed solution where ORIGYN handles the infrastructure, or the Custom Installation, which uses the production-ready ICRC-7 standard for developers requiring full control.

### Prerequisites

Before beginning either method, you must have the Internet Computer SDK (dfx) installed and a developer identity configured.

First, install DFX by following the[ official setup guide](https://internetcomputer.org/docs/current/developer-docs/getting-started/install/). Once installed, verify your identity by running

```bash
 dfx identity get-principal
```

&#x20;in your terminal. Save this Principal ID, as it is required for authorization and ownership in both methods.

### Method 1: Minting Studio API (Managed Service)

This method is ideal for creators who prefer not to manage canister maintenance or cycle (gas) top-ups. ORIGYN handles the infrastructure in exchange for a fee of 15,000 OGY tokens.

#### 1. Request Authorization

The Minting Studio is a permissioned environment. To gain access, send an email to [admin@origyn.ch](mailto:admin@origyn.ch) including a brief description of your project and the Principal ID you retrieved in the prerequisites step. You must wait for confirmation that your address has been whitelisted before proceeding.

#### 2. Prepare Your Environment

Ensure your wallet holds at least 15,000 OGY to cover the creation fee. You will be interacting with the Minting Studio canister (**`uasjq-dyaaa-aaaas-qdwka-cai`**) and should reference the [ORIGYN-SA/minting-studio](https://github.com/ORIGYN-SA/claimlink) repository if you need deep technical details.

#### 3. Create a Metadata Template

You must first define the structure of your NFTs by registering a JSON template. The easiest way is to use the [Visual Template Builder](https://ahegaoburger.github.io/claimlink-template-builder/) — a drag-and-drop tool that generates the JSON for you. See the [Templates](templates.md) page for full details on template structure and field types.

Once you have your template JSON, register it:

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai create_template '(record {
 template_json = "<your_template_json_here>"
})'
```

Note the template\_id returned by this command (e.g., 1), as you will need it shortly.

#### 4. Approve Fee Payment

Authorize the Minting Studio to spend the required fee from your wallet. Replace OGY\_LEDGER\_ID in the command below with the official OGY Ledger ID lkwrt-vyaaa-aaaaq-aadhq-cai.

```bash
dfx canister --network ic call lkwrt-vyaaa-aaaaq-aadhq-cai icrc2_approve '(record {
  amount = 15000000000; 
  spender = record { owner = principal "uasjq-dyaaa-aaaas-qdwka-cai"; }
})'
```

#### 5. Create the Collection

Submit the final request to spin up your NFT canister using the create\_collection endpoint. Ensure you replace template\_id = 1 with the actual ID you received in Step 3.

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai create_collection '(record {
  name = "My Unique Collection";
  description = "A collection of rare digital artifacts.";
  symbol = "MUC";
  template_id = 1; 
})'
```

This command returns a request\_id. You can monitor the installation progress (which typically takes under a minute) by querying get\_collection\_info with this ID. Look for the status TemplateUploaded to confirm success. If the status is Failed, the protocol automatically reimburses the 15,000 OGY fee.

***

### Method 2: Custom Installation (ICRC-7 Standard)

This method provides a fully compliant implementation of the ICRC-7 (NFT) and ICRC-37 (Batch Approval) standards. **Note: While this is the best choice for developers who need full control over their smart contracts, it requires you to manage your own cycles usage (gas cost) and are in full responsibility of your collection.**

#### 1. Setup and Configuration

Start by cloning the repository and setting up your environment variables. This creates a cleaner workspace and simplifies the commands you will run later.

```bash
git clone https://github.com/ORIGYN-SA/nft.git
cd nft
# Set these variables to match your project details
export NFT_CANISTER_ID="YOUR_CANISTER_ID" 
export YOUR_PRINCIPAL_ID="YOUR_ACTUAL_PRINCIPAL_ID"
export COLLECTION_NAME="MyCollection"
export COLLECTION_SYMBOL="MC"
export COLLECTION_DESCRIPTION="My Description"
```

#### 2. Deploy the Collection

You can deploy the collection using the automated script provided in the repository, which handles identity creation and configuration for you.

```bash
./deploy_collection.sh
```

Alternatively, if you prefer manual control, you can use dfx deploy directly. This requires you to pass a complex argument record containing your principal permissions, collection name, symbol, and versioning details. Refer to the repository README for the full argument structure if choosing manual deployment.<br>
