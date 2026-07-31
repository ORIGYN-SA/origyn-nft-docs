---
icon: wrench
---

# Tool Reference

Fourteen tools, in two groups: the NFT lifecycle, and [agent memory](agent-memory.md). Each is annotated so your host knows how to treat it — read-only tools run without prompting, paid ones ask first.

## Reading the annotations

| Annotation | Meaning |
| ---------- | ------- |
| `readOnlyHint` | Free query. Changes nothing, spends nothing |
| `destructiveHint` | The host prompts before calling. Cannot be undone |
| `idempotentHint` | Calling twice with the same arguments has the same effect as once |

Every tool also carries an `origyn/cost-class` in its `_meta`: `free`, `low`, `medium`, `high`, or `variable`. Agents use it to decide whether to check with you first.

{% hint style="info" %}
Numeric arguments are passed as **decimal strings**, not JSON numbers — `"num_mints": "10"`. Token IDs and OGY amounts routinely exceed the safe integer range, so the schemas take strings and parse them to `bigint` server-side.
{% endhint %}

## NFT lifecycle

### `wallet_info`

Free. Returns the agent's principal, its OGY funding address, current balance, and how much of the session budget is left.

No arguments.

Ask this first in any session where money will be spent — it is the only way to see remaining budget before hitting a cap.

### `estimate_cost`

Free. Previews the OGY cost of a mint. Does not approve or spend anything.

| Argument | Type | |
| -------- | ---- | --- |
| `num_mints` | string | How many certificates |
| `total_file_size_bytes` | string | Combined size of attached files |

Cost on ORIGYN scales with stored bytes, since content lives on-chain rather than behind a link. A 40 MB batch costs meaningfully more than a 400 KB one.

### `create_collection`

**Paid, ~15,000 OGY. Cannot be undone.**

| Argument | Type | |
| -------- | ---- | --- |
| `name` | string | Display name |
| `description` | string | |
| `symbol` | string | Short ticker |

Creates a collection owned by the agent, using the shared agent template. There is no `template_id` and no categories to manage.

{% hint style="warning" %}
This creates an [AI collection](../ai-collections/overview.md), which means metadata is **not validated against a template**. Your JSON must parse and stay under the per-item size cap, but no field is required and no shape is enforced. A misspelled key is stored exactly as sent.
{% endhint %}

Collection creation deploys a canister, which takes a few seconds. The tool polls until the collection is ready and returns its canister ID.

### `mint_nft`

**Paid, ~100–1,000 OGY. Cannot be undone.**

| Argument | Type | |
| -------- | ---- | --- |
| `collection_canister_id` | string | |
| `metadata` | object | Stored on-chain as JSON |
| `recipient` | string, optional | Defaults to the agent's own principal |

Mints one certificate. For the metadata envelope conventions — display keys at the top level, field values under `data` — see [Producing your mint JSON](../minting-studio/minting.md#producing-your-mint-json-from-a-template).

### `append_file`

**Paid, cost scales with bytes. Cannot be undone.**

| Argument | Type | |
| -------- | ---- | --- |
| `collection_canister_id` | string | |
| `token_id` | string | |
| `entry_name` | string | Name the file is filed under |
| `content` | string | UTF-8 text |

Attaches a file to a certificate that has **already been minted** — the one thing agent collections can do that standard collections cannot, where files can only be attached at mint time.

Content is UTF-8 text, so this is a good fit for reports, logs, and structured records; it is not a binary upload path.

### `read_nft`

Free. Returns metadata and owner.

| Argument | Type |
| -------- | ---- |
| `collection_canister_id` | string |
| `token_id` | string |

### `list_nfts`

Free. Both arguments are optional; the default is everything the agent owns across all collections it knows about.

| Argument | Type | Default |
| -------- | ---- | ------- |
| `collection_canister_id` | string, optional | all known collections |
| `owned_by` | string, optional | the agent |

### `transfer_nft`

**Irreversible, but costs no OGY** — only the ledger fee.

| Argument | Type |
| -------- | ---- |
| `collection_canister_id` | string |
| `token_id` | string |
| `to` | string (principal) |

{% hint style="warning" %}
Free does not mean safe. A transfer to a mistyped principal is unrecoverable, and no spend cap protects you here because nothing is being spent. This is the tool to be most careful about auto-approving.
{% endhint %}

Reads, lists, and transfers go straight to the collection's own canister over ICRC-7, not through the minting canister.

## Agent memory

Six further tools turn a collection into a versioned store for an agent's own state. They are documented on their own page: [Agent Memory](agent-memory.md).

| Tool | Cost | |
| ---- | ---- | --- |
| `init_memory` | ~15,000 OGY | Create the memory store |
| `commit_memory` | variable | Save a new version |
| `list_versions` | free | History, HEAD, and tags |
| `read_version` | free | Read a version's files |
| `tag_version` | free | Name a commit |
| `rollback_memory` | free | Move HEAD backwards |

## Errors

SDK errors are translated into actionable messages rather than raw canister traps, so the agent can usually recover on its own.

| Error | Meaning |
| ----- | ------- |
| `InsufficientFunds` | The agent's OGY balance is too low |
| `SpendCapExceeded` | Per-call cap or session budget would be exceeded |
| `RateLimitExceeded` | Per-tool call limit hit for this session |
| `AllowlistDenied` | Target canister is not in the configured allowlist |
| `NotFound` | No such collection or token |
| `NotOwner` | The agent does not own the token it tried to modify |
