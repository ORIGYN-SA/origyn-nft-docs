---
icon: terminal
---

# Setup & Deployment

This method provides a fully compliant implementation of the ICRC-7 (NFT) and ICRC-37 (Batch Approval) standards. If you prefer a managed service, see [Minting Studio](../minting-studio/getting-started.md) instead.

**Note:** While this is the best choice for developers who need full control over their smart contracts, it requires you to manage your own cycles usage (gas cost) and you are in full responsibility of your collection.

### Prerequisites

Before beginning, you must have the Internet Computer SDK (dfx) installed and a developer identity configured.

First, install DFX by following the [official setup guide](https://internetcomputer.org/docs/current/developer-docs/getting-started/install/). Once installed, verify your identity by running

```bash
dfx identity get-principal
```

in your terminal. Save this Principal ID, as it is required for authorization and ownership.

***

### 1. Setup and Configuration

Start by cloning the repository and setting up your environment variables.

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

### 2. Deploy the Collection

You can deploy the collection using the automated script provided in the repository, which handles identity creation and configuration for you.

```bash
./deploy_collection.sh
```

Alternatively, if you prefer manual control, you can use `dfx deploy` directly. This requires you to pass a complex argument record containing your principal permissions, collection name, symbol, and versioning details. Refer to the repository README for the full argument structure if choosing manual deployment.

***

### What's Next?

* **[CLI Management](cli-management.md)** — Use the CLI tools for uploading files, creating metadata, and minting tokens
* **[DFX Management](../getting-started/publish-your-docs.md)** — Manage permissions, update metadata, and upload files via dfx
* **[ICRC-37 / ICRC-7](../getting-started/icrc37-icrc7.md)** — Technical reference for the NFT standards your collection implements
