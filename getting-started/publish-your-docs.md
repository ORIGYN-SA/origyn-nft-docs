---
icon: flask-round-potion
metaLinks:
  alternates:
    - >-
      https://app.gitbook.com/s/yE16Xb3IemPxJWydtPOj/getting-started/publish-your-docs
---

# DFX Management

This guide is for users managing their collection using dfx commands or the Candid UI. This applies to both Minting Studio and Custom Installation users. For CLI-based management, see [CLI Management](../custom-installation/cli-management.md).

#### Prerequisites

* DFX: Installed and configured.
* Controller Identity: You must be using the principal that is listed as a controller or has ManageAuthorities permissions.
* Canister ID: You must know your $NFT\_CANISTER\_ID.

#### 1. Managing Permissions (Granting/Revoking)

To add a new admin (e.g., a new developer or a multi-sig wallet), you use grant\_permission.

Permission Types: UpdateMetadata, Minting, UpdateCollectionMetadata, UpdateUploads, ManageAuthorities, ReadUploads.

Command:

```bash
dfx canister call $NFT_CANISTER_ID grant_permission "(record {
  permission = variant { Minting };
  principal = principal \"YOUR_TARGET_PRINCIPAL_HERE\"
})" --network ic


```

To Revoke:

```bash
dfx canister call $NFT_CANISTER_ID revoke_permission "(record {
  permission = variant { Minting };
  principal = principal \"YOUR_TARGET_PRINCIPAL_HERE\"
})" --network ic
```

```bash
Bash
```

#### 2. Updating Collection Metadata

If you need to change the collection name, description, or logo after launch.

Command:

```bash
dfx canister call $NFT_CANISTER_ID update_collection_metadata "(record {
  name = opt \"New Collection Name\";
  description = opt \"Updated Description\";
  logo = opt \"https://your-new-logo-url.com/logo.png\";
  symbol = opt \"NEW\";
  supply_cap = null;
  tx_window = null;
  default_take_value = null;
  max_canister_storage_threshold = null;
  permitted_drift = null;
  max_take_value = null;
  max_update_batch_size = null;
  max_query_batch_size = null;
  max_memo_size = null;
  atomic_batch_transfers = null;
  collection_metadata = null;
})" --network ic
```

#### 3. Uploading Files (Manual DFX Method)

Note: Uploading large files via dfx manually is difficult because you must split the file into chunks. For files < 2MB, you can often do it in one go.

Step A: Initialize Upload

```bash
dfx canister call $NFT_CANISTER_ID init_upload "(record {
  file_path = \"my_image.png\";
  file_size = 1000 : nat64; 
  file_hash = \"sha256_hash_of_file\"; 
  chunk_size = null;
})" --network ic
```

Step B: Store Chunk You must convert your image to a Blob (byte array) for this step.

```bash
dfx canister call $NFT_CANISTER_ID store_chunk "(record {
  chunk_id = 0 : nat;
  file_path = \"my_image.png\";
  chunk_data = blob \"...hex_representation_of_image...\"
})" --network ic
Step C: Finalize Upload Returns the public URL of the file.
Bash
dfx canister call $NFT_CANISTER_ID finalize_upload "(record {
  file_path = \"my_image.png\"
})" --network ic
```

#### 4. Updating NFT Metadata

To update the metadata of a specific token (e.g., revealing a mystery box).

```bash
dfx canister call $NFT_CANISTER_ID update_nft_metadata "(record {
  token_id = 1 : nat;
  metadata = vec {
    record { \"name\"; variant { Text = \"Revealed Name\" } };
    record { \"image\"; variant { Text = \"https://...\" } };
    record { \"rarity\"; variant { Text = \"Legendary\" } };
  }
})" --network ic
```

