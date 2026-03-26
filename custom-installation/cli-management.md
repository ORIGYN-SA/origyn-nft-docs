---
icon: command-line
---

# CLI Management

Target Audience: Users who cloned the repo, built the `origyn_icrc7_cmdlinetools` binary, and have full terminal access. For DFX-based management (applicable to both Minting Studio and Custom Installation), see [DFX Management](../getting-started/publish-your-docs.md).

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

* Options: Add `--chunk_size 2000000` to adjust upload speeds.

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

***

### Troubleshooting

* **"UploadNotInitialized"**: If an upload fails midway, the canister might lose the init state. Run `init_upload` (or the CLI upload command) again.
* **"ConcurrentManagementCall"**: The canister prevents multiple management calls (like updates or mints) happening in the exact same block/batch to protect state. Retry the command.
* **Permission Denied**: Ensure the identity in `$IDENTITY_FILE` matches the controller or a principal with ManageAuthorities.
