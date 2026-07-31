---
icon: rectangle-code
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

The deployment assets live in the `example/` directory, not at the repository root. Change into it first:

```bash
cd example
```

#### Option A: Automated script

```bash
./deploy_collection.sh
```

The script checks prerequisites, prompts for your configuration, writes `canister_ids.json`, deploys, and builds the CLI tool.

> **This script is intended for testing.** It deploys with `--mode reinstall`, which **wipes all existing state** on the target canister, and it prompts for a canister you have already created. Do not run it against a live collection.

#### Option B: Manual deployment

```bash
dfx deploy --network ic nft --argument '(
  variant {
    Init = record {
      name = "'"$COLLECTION_NAME"'";
      symbol = "'"$COLLECTION_SYMBOL"'";
      description = opt "'"$COLLECTION_DESCRIPTION"'";
      logo = null;
      test_mode = false;
      version = record { major = 1 : nat32; minor = 1 : nat32; patch = 1 : nat32 };
      commit_hash = "";
      vetkd_key_name = "key_1";
      vetkd_context = "origyn_nft";
      permissions = record {
        user_permissions = vec {
          record {
            principal "'"$YOUR_PRINCIPAL_ID"'";
            vec {
              variant { UpdateMetadata };
              variant { Minting };
              variant { UpdateCollectionMetadata };
              variant { UpdateUploads };
              variant { ManageAuthorities };
              variant { ReadUploads };
            };
          };
        };
      };
      approval_init = record {};
      collection_metadata = vec {};
      base_url = null;
      supply_cap = null;
      tx_window = null;
      default_take_value = null;
      max_take_value = null;
      max_query_batch_size = null;
      max_update_batch_size = null;
      max_memo_size = null;
      max_canister_storage_threshold = null;
      permitted_drift = null;
      atomic_batch_transfers = null;
    }
  }
)'
```

**`vetkd_key_name` and `vetkd_context` are required.** They configure the vetKD key used for private (encrypted) content. Any install command written before these fields existed will fail to decode. Use a real mainnet key name in production; `dfx_test_key` is for local replicas only.

> **Do not set `test_mode = true` in production.** In test mode the canister grants all six permissions to the installing principal on top of whatever `permissions` you pass, which hides a misconfigured `permissions` record until the day you turn test mode off.

For the full walkthrough, including uploading files and minting, see [`example/README.md`](https://github.com/ORIGYN-SA/nft/blob/master/example/README.md) in the repository.

***

### What's Next?

* [**CLI Management**](cli-management.md): Use the CLI tools for uploading files, creating metadata, and minting tokens
* [**Management**](../managing-your-collection/management.md): Manage permissions, update metadata, and upload files via dfx
* [**ICRC-37 / ICRC-7**](../technical-reference/icrc37-icrc7.md): Technical reference for the NFT standards your collection implements
