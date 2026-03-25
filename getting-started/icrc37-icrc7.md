---
icon: sunglasses
---

# Icrc37 - Icrc7

This document outlines the technical standards implemented in the ORIGYN NFT canister.

### Standards Overview

| **Standard** | **Name**  | **Purpose**                                                                             |
| ------------ | --------- | --------------------------------------------------------------------------------------- |
| ICRC-7       | Core NFT  | Defines ownership, transfers, and metadata (name, symbol, logo).                        |
| ICRC-37      | Approvals | Enables "delegated authority," allowing marketplaces/dApps to sell NFTs on your behalf. |
| ICRC-3       | History   | Maintains a secure, verified, and immutable log of all transactions.                    |
| Indexer      | Query     | A companion service for filtering history by account or Token ID.                       |

***

### 1. ICRC-3: Transaction History

Purpose: The backbone of transparency. It records every action in a cryptographically certified ledger.

#### Key Features

* Validation: Prevents malformed data before writing to the ledger.
* Certification: Ensures the state cannot be tampered with.
* Archiving: Automatically offloads old history to separate archive canisters to manage storage.

#### Common Commands

Get Supported Block Types

```bash
dfx canister call $NFT_CANISTER_ID icrc3_supported_block_types '()' --network ic
```

Get Archive Canisters

```bash
dfx canister call $NFT_CANISTER_ID icrc3_get_archives '()' --network ic
```

Get Ledger Configuration

```bash
dfx canister call $NFT_CANISTER_ID icrc3_get_properties '()' --network ic
```

Fetch Transaction Blocks

_Retrieves a range of blocks (e.g., the first 10)._

```bash
dfx canister call $NFT_CANISTER_ID icrc3_get_blocks "(vec { 
  record { start = 0; length = 10 } 
})" --network ic
```

***

### 2. ICRC-7: Core NFT Standard

Purpose: Handles the creation, ownership, and transfer of the NFTs.

#### Key Features

* Batch Operations: Supports transferring multiple tokens in one call.
* Dynamic Metadata: Uses flexible storage (Maps, Arrays) for complex asset data.

#### Collection Information

```bash
# Basic Info
dfx canister call $NFT_CANISTER_ID icrc7_name '()' --network ic
dfx canister call $NFT_CANISTER_ID icrc7_symbol '()' --network ic
dfx canister call $NFT_CANISTER_ID icrc7_description '()' --network ic
dfx canister call $NFT_CANISTER_ID icrc7_logo '()' --network ic

# Supply Stats
dfx canister call $NFT_CANISTER_ID icrc7_total_supply '()' --network ic
dfx canister call $NFT_CANISTER_ID icrc7_supply_cap '()' --network ic

# Full Metadata Dump
dfx canister call $NFT_CANISTER_ID icrc7_collection_metadata '()' --network ic
```

#### Token Management

Check Balance (Multiple Accounts)

```bash
dfx canister call $NFT_CANISTER_ID icrc7_balance_of "(vec { 
  record { owner = principal \"$(dfx identity get-principal)\"; subaccount = null } 
})" --network ic
```

Find Owner of Token IDs

```bash
dfx canister call $NFT_CANISTER_ID icrc7_owner_of "(vec { 101; 102 })" --network ic
```

Transfer NFT

```bash
dfx canister call $NFT_CANISTER_ID icrc7_transfer "(vec {
  record {
    to = record { owner = principal \"RECIPIENT_PRINCIPAL\"; subaccount = null };
    token_id = 1;
    memo = null;
    from_subaccount = null;
    created_at_time = null;
  }
})" --network ic
```

List Tokens Owned by Account

```bash
dfx canister call $NFT_CANISTER_ID icrc7_tokens_of "(
  record { owner = principal \"$(dfx identity get-principal)\"; subaccount = null }, 
  opt 0, 
  opt 10
)" --network ic
```

***

### 3. ICRC-37: Token Approvals

Purpose: Allows third parties (marketplaces, games) to transfer tokens on your behalf without taking custody.

#### Common Commands

Approve Spender (Specific Token)

```bash
dfx canister call $NFT_CANISTER_ID icrc37_approve_tokens "(vec {
  record {
    token_id = 42;
    spender = record { owner = principal \"SPENDER_PRINCIPAL\"; subaccount = null };
    memo = null;
    expires_at = null;
    created_at_time = null;
    from_subaccount = null;
  }
})" --network ic
```

Approve Spender (Entire Collection)

```bash
dfx canister call $NFT_CANISTER_ID icrc37_approve_collection "(vec {
  record {
    spender = record { owner = principal \"MARKETPLACE_PRINCIPAL\"; subaccount = null };
    memo = null;
    expires_at = null;
    created_at_time = null;
    from_subaccount = null;
  }
})" --network ic
```

Revoke Approval

_To revoke, pass `opt` wrapped spender details. To revoke all, pass `null`._

```bash
dfx canister call $NFT_CANISTER_ID icrc37_revoke_token_approvals "(vec {
  record {
    token_id = 42;
    spender = opt record { owner = principal \"SPENDER_PRINCIPAL\"; subaccount = null };
    from_subaccount = null;
    memo = null;
    created_at_time = null;
  }
})" --network ic
```

Transfer From (Executed by Spender)

```bash
dfx canister call $NFT_CANISTER_ID icrc37_transfer_from "(vec {
  record {
    spender_subaccount = null;
    from = record { owner = principal \"OWNER_PRINCIPAL\"; subaccount = null };
    to = record { owner = principal \"BUYER_PRINCIPAL\"; subaccount = null };
    token_id = 42;
    memo = null;
    created_at_time = null;
  }
})" --network ic
```

***

### 4. ICRC Indexer

Purpose: A standalone service that indexes the ledger, allowing for complex queries like "Show me all transactions for User X."

#### Common Commands

Check Indexer Status

```bash
dfx canister call $INDEXER_ID status '()' --network ic
```

Get Transactions for an Account

_Filters the history to show only blocks involving a specific user._

```bash
dfx canister call $INDEXER_ID get_blocks "(record {
  sort_by = opt variant { Ascending };
  filters = vec { 
    variant { Account = record { owner = principal \"TARGET_PRINCIPAL\"; subaccount = null } } 
  };
  start = 0;
  length = 5;
})" --network ic
```
