---
icon: gear
---

# Management

This guide covers managing your ORIGYN NFT collection after deployment using dfx commands. For CLI-based management (Custom Installation only), see [CLI Management](../custom-installation/cli-management.md).

## Minting Studio Permissions

If you created your collection via the **Minting Studio**, permissions are **automatically configured**, you do not need to manage them yourself. The Minting Studio canister acts as a trusted intermediary and handles minting, file uploads, and metadata updates on your behalf.

**Your permissions as a Minting Studio collection owner:**

| Permission                 | What you can do                                           |
| -------------------------- | --------------------------------------------------------- |
| `ReadUploads`              | Read uploaded files from your collection                  |
| `UpdateCollectionMetadata` | Update the collection name, description, logo, and symbol |

**Managed by the Minting Studio canister (not user-accessible):**

| Permission       | What the canister handles                                                           |
| ---------------- | ----------------------------------------------------------------------------------- |
| `Minting`        | Creating new certificates. done via the [Minting API](../minting-studio/minting.md) |
| `UpdateUploads`  | Uploading files. done via the proxy upload endpoints                                |
| `UpdateMetadata` | Updating individual token metadata                                                  |

This means you **do not** need to call `grant_permission` or `revoke_permission` on Minting Studio collections. All minting and upload operations go through the Minting Studio API, which forwards them to your collection canister with the correct permissions.

---

## Updating Collection Metadata

**Applies to:** Minting Studio and Custom Installation

If you need to change the collection name, description, or logo after launch, you can do so directly since you have the `UpdateCollectionMetadata` permission.

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

---

## Managing Permissions (Custom Installation Only)

> **Minting Studio users:** Skip this section. Permissions are automatically managed for you.

If you deployed via Custom Installation, you have full control over permissions. Use `grant_permission` to add admins and `revoke_permission` to remove them.

Permission Types: `UpdateMetadata`, `Minting`, `UpdateCollectionMetadata`, `UpdateUploads`, `ManageAuthorities`, `ReadUploads`.

Grant a permission:

```bash
dfx canister call $NFT_CANISTER_ID grant_permission "(record {
  permission = variant { Minting };
  principal = principal \"YOUR_TARGET_PRINCIPAL_HERE\"
})" --network ic
```

Revoke a permission:

```bash
dfx canister call $NFT_CANISTER_ID revoke_permission "(record {
  permission = variant { Minting };
  principal = principal \"YOUR_TARGET_PRINCIPAL_HERE\"
})" --network ic
```

---

## Uploading Files (Custom Installation Only)

> **Minting Studio users:** Use the [proxy upload endpoints](../minting-studio/minting.md) instead. Direct file uploads require the `UpdateUploads` permission, which is held by the Minting Studio canister.

Uploading large files via dfx manually requires splitting files into chunks. For files under 2 MB, you can often do it in one go.

Step A: Initialize Upload

```bash
dfx canister call $NFT_CANISTER_ID init_upload "(record {
  file_path = \"my_image.png\";
  file_size = 1000 : nat64;
  file_hash = \"sha256_hash_of_file\";
  chunk_size = null;
})" --network ic
```

Step B: Store Chunk

You must convert your image to a Blob (byte array) for this step.

```bash
dfx canister call $NFT_CANISTER_ID store_chunk "(record {
  chunk_id = 0 : nat;
  file_path = \"my_image.png\";
  chunk_data = blob \"...hex_representation_of_image...\"
})" --network ic
```

Step C: Finalize Upload

Returns the public URL of the file.

```bash
dfx canister call $NFT_CANISTER_ID finalize_upload "(record {
  file_path = \"my_image.png\"
})" --network ic
```

---

## Updating NFT Metadata (Custom Installation Only)

> **Minting Studio users:** This operation requires the `UpdateMetadata` permission, which is held by the Minting Studio canister and not directly accessible.

To update the metadata of a specific token (e.g., revealing a mystery box):

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
