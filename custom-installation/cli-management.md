---
icon: rectangle-terminal
---

# CLI Management

This page is for people who cloned the repository, built the `origyn_icrc7_cmdlinetools` binary and work from a terminal. For managing a collection with dfx, which applies to both Minting Studio and Custom Installation collections, see [Management](../managing-your-collection/management.md).

### Environment Setup

Setup your shell variables to make commands easier:

```bash
export NFT_CANISTER_ID="YOUR_CANISTER_ID"
export IDENTITY_FILE="identity.pem"
# Export your dfx identity to pem:
# dfx identity export $(dfx identity whoami) > identity.pem
```

***

### 1. Advanced File Uploads (Automatic Chunking)

The CLI tool handles the complex chunking process automatically.

Upload a file:

```bash
./origyn_icrc7_cmdlinetools \
  --network ic \
  --identity $IDENTITY_FILE \
  --canister $NFT_CANISTER_ID \
  upload-file ./local_path/image.png target_filename.png
```

Result: This returns a URL (e.g., `https://...raw.icp0.io/...`) which you should save for minting.

* Options: `--chunk_size <bytes>` (or `-s`) sets the chunk size. The default, `1048576` (1 MiB), is also the **maximum**. A larger value is rejected by the storage canister, and the failed attempt still costs the collection cycles because it spawns a new storage canister first.
* A single file can be at most **100 MiB**.

Upload a JSON metadata file (for example one produced by `create-metadata`):

```bash
./origyn_icrc7_cmdlinetools \
  --network ic \
  --identity $IDENTITY_FILE \
  --canister $NFT_CANISTER_ID \
  upload-metadata ./metadata.json
```

It accepts the same `--chunk_size` option.

### 2. Batch Metadata Creation & Validation

Before minting, you can generate and validate your metadata JSON files to ensure they are ICRC-97 compliant.

Interactive Creator:

```bash
./origyn_icrc7_cmdlinetools \
  --network ic \
  --identity $IDENTITY_FILE \
  --canister $NFT_CANISTER_ID \
  create-metadata --output metadata.json --interactive
```

Validate External JSON:

```bash
./origyn_icrc7_cmdlinetools \
  --network ic \
  --identity $IDENTITY_FILE \
  --canister $NFT_CANISTER_ID \
  validate-metadata ./my_metadata.json
```

### 3. Minting Tokens

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
  --metadata "name:My CLI NFT" \
  --metadata "description:Created via CLI" \
  --metadata "rarity:Legendary"
```

Two things about this command catch people out.

**`--name` does not become metadata.** It is required by the CLI but its value is discarded, so a token minted without a `name` metadata entry ends up unnamed. Pass the name twice, as shown above.

**`--metadata` splits on the first `:`.** A value containing a colon is truncated at it, so `--metadata "image:https://example.com/i.png"` stores just `https`. Set URL-valued fields with the ICRC-97 flag below, or upload the file and reference it from the metadata JSON.

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

### 4. Permission Management (CLI)

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

Revoke a permission:

```bash
./origyn_icrc7_cmdlinetools \
  --network ic \
  --identity $IDENTITY_FILE \
  --canister $NFT_CANISTER_ID \
  permissions revoke --principal "TARGET_PRINCIPAL" --permission "minting"
```

Check a single permission:

```bash
./origyn_icrc7_cmdlinetools \
  --network ic \
  --identity $IDENTITY_FILE \
  --canister $NFT_CANISTER_ID \
  permissions has --principal "TARGET_PRINCIPAL" --permission "minting"
```

Permission names: `minting`, `manage_authorities`, `update_metadata`, `update_collection_metadata`, `read_uploads`, `update_uploads`.

`--network` defaults to `local`. Always pass `--network ic` for a mainnet collection.

***

### Troubleshooting

* **"UploadNotInitialized"**: If an upload fails midway, the canister might lose the init state. Run `init_upload` (or the CLI upload command) again.
* **"ConcurrentManagementCall"**: The canister allows only **one management call in flight at a time, across all callers**. If anyone else's mint, metadata update, permission change, or upload chunk is executing, your call is rejected. Retry the command. The lock is held per call and released when it returns, so a chunked upload does not block others for its whole duration, only for each individual chunk.
* **Permission Denied**: Ensure the identity in `$IDENTITY_FILE` matches the controller or a principal with ManageAuthorities.
