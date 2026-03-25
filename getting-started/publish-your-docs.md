---
icon: flask-round-potion
metaLinks:
  alternates:
    - >-
      https://app.gitbook.com/s/yE16Xb3IemPxJWydtPOj/getting-started/publish-your-docs
---

# Management

### Minting Studio / DFX Management

Target Audience: Users who created a canister via the minting studio and need to manage it using dfx or the Candid UI.

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

***

### Part 2: Direct Installation / CLI Management

Target Audience: Users who cloned the repo, built the origyn\_icrc7\_cmdlinetools binary, and have full terminal access.

#### Environment Setup

Setup your shell variables to make commands easier:

```bash
export NFT_CANISTER_ID="YOUR_CANISTER_ID"
export IDENTITY_FILE="identity.pem" 
# Export your dfx identity to pem:
# dfx identity export $(dfx identity whoami) > identity.pem
```

#### 1. Advanced File Uploads (Automatic Chunking)

The CLI tool handles the complex chunking process (Step 3 in the DFX guide) automatically.

Upload a file:

```bash
./origyn_icrc7_cmdlinetools \
  --network ic \
  --identity $IDENTITY_FILE \
  --canister $NFT_CANISTER_ID \
  upload-file ./local_path/image.png target_filename.png  
```

\
Result: This returns a URL (e.g., https://...raw.icp0.io/...) which you should save for minting.

* Options: Add `--chunk_size 2000000` to adjust upload speeds.

#### 2. Batch Metadata Creation & Validation

Before minting, you can generate and validate your metadata JSON files to ensure they are ICRC-97 compliant.

Interactive Creator:

```bash
./origyn_icrc7_cmdlinetools \
  --network ic \
  --identity $IDENTITY_FILE \
  --canister $NFT_CANISTER_ID \
```

&#x20; create-metadata `--output metadata.json --interactive`

Validate External JSON:

```bash
./origyn_icrc7_cmdlinetools \
  --network ic \
  --identity $IDENTITY_FILE \
  --canister $NFT_CANISTER_ID \
  validate-metadata ./my_metadata.json
```

#### 3. Minting Tokens

The CLI simplifies the mint argument construction.

Mint with specific metadata:

```bash
./origyn_icrc7_cmdlinetools \
  --network ic \
  --identity $IDENTITY_FILE \
  --canister $NFT_CANISTER_ID \
  mint \
  --owner "RECEIVER_PRINCIPAL" \
  --name "My CLI NFT" \
  --metadata "description:Created via CLI" \
  --metadata "image:https://$NFT_CANISTER_ID.raw.icp0.io/image.png" \
  --metadata "rarity:Legendary"
```

Mint using a hosted JSON file (ICRC-97 URL):

```bash
./origyn_icrc7_cmdlinetools \
  --network ic \
  --identity $IDENTITY_FILE \
  --canister $NFT_CANISTER_ID \
  mint \
  --owner "RECEIVER_PRINCIPAL" \
  --name "My NFT" \
  --icrc97_url "https://$NFT_CANISTER_ID.raw.icp0.io/metadata.json"
```

#### 4. Permission Management (CLI)

Managing multi-sig signers or admin roles is faster via CLI.

Check Permissions:

```bash
./origyn_icrc7_cmdlinetools \
  --network ic \
  --identity $IDENTITY_FILE \
  --canister $NFT_CANISTER_ID \
   permissions list --principal "TARGET_PRINCIPAL"
```

Grant Minting Rights:

```bash
./origyn_icrc7_cmdlinetools \
  --network ic \
  --identity $IDENTITY_FILE \
  --canister $NFT_CANISTER_ID \
  permissions grant --principal "TARGET_PRINCIPAL" --permission "minting"
```

#### Troubleshooting

* "UploadNotInitialized": If an upload fails midway, the canister might lose the init state. Run init\_upload (or the CLI upload command) again.
* "ConcurrentManagementCall": The canister prevents multiple management calls (like updates or mints) happening in the exact same block/batch to protect state. Retry the command.
* Permission Denied: Ensure the identity in $IDENTITY\_FILE matches the controller or a principal with ManageAuthorities.
