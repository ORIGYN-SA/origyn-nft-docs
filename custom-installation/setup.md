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

{% hint style="warning" %}
**Use the manual command below, not `deploy_collection.sh`.** The script in `example/` (and the deploy command in `example/README.md`) omits the required `vetkd_key_name` and `vetkd_context` fields, so it currently fails to encode its install argument. It also deploys with `--mode reinstall`, which wipes all state on the target canister.
{% endhint %}

#### Manual deployment

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
      storage_cycles = null;
    }
  }
)'
```

**`vetkd_key_name` and `vetkd_context` are required.** They configure the vetKD key used for private (encrypted) content. Any install command written before these fields existed will fail to decode. Use a real mainnet key name in production; `dfx_test_key` is for local replicas only.

> **Do not set `test_mode = true` in production.** In test mode the canister grants all six permissions to the installing principal on top of whatever `permissions` you pass, which hides a misconfigured `permissions` record until the day you turn test mode off.

### Storage canister cycles

Your collection stores files in storage canisters it creates and funds itself, from its own cycles balance. `storage_cycles` tunes that. Leave it `null` for the defaults, or set any of its fields (each is optional):

```candid
storage_cycles = opt record {
  initial_cycles        = opt (2_000_000_000_000 : nat);  // cycles a new storage canister starts with
  reserved_cycles_limit = null;                            // reserved cycles cap of a storage canister
  funding_interval_secs = null;                            // how often storage canisters are checked
  funding_min_cycles    = null;                            // top up a storage canister below this balance
  funding_fund_cycles   = null;                            // how much each top-up sends
};
```

| Field | Default |
| ----- | ------- |
| `initial_cycles` | 2 TC |
| `reserved_cycles_limit` | 2 TC |
| `funding_interval_secs` | 3600 (hourly) |
| `funding_min_cycles` | 1 TC |
| `funding_fund_cycles` | 2 TC |

A new storage canister is created whenever the current one fills up (500 GiB each), and its starting cycles come out of the collection's balance, so keep the collection funded.

On an upgrade, pass `storage_cycles` inside the `Upgrade` record to change the settings; `null` keeps the ones already stored.

### Upgrading an existing collection

A collection upgrades its own storage canisters automatically, on a timer that starts right after the collection's upgrade. That step logs failures and does not retry them, so check it before sending uploads:

1. Read the collection's logs and confirm there is no `Storage canister upgrade failed` entry.
2. Confirm every storage canister's module hash equals the SHA-256 of `wasm/storage_canister.wasm.gz` from the build you installed:

   ```bash
   shasum -a 256 wasm/storage_canister.wasm.gz
   dfx canister --network ic info <storage_canister_id>   # Module hash: 0x...
   ```

If either check fails, run the collection upgrade again; it retries every storage canister still behind.

{% hint style="danger" %}
**Do not upload until both checks pass.** A storage canister left on an older version rejects the new upload arguments, and the collection reads that rejection as "this canister is full": it creates, and pays for, a new storage canister on every upload.
{% endhint %}

The ORIGYN NFT canister is open source under the Apache 2.0 license.

***

### What's Next?

* [**CLI Management**](cli-management.md): Use the CLI tools for uploading files, creating metadata, and minting tokens
* [**Management**](../managing-your-collection/management.md): Manage permissions, update metadata, and upload files via dfx
* [**ICRC-37 / ICRC-7**](../technical-reference/icrc37-icrc7.md): Technical reference for the NFT standards your collection implements
