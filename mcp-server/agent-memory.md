---
icon: code-branch
---

# Agent Memory

An agent's memory is usually a file on someone's laptop: unversioned, unauditable, and gone when the machine is. The MCP server offers an alternative — store it as a collection, where every version is an immutable certificate.

The model borrows git's object model and maps it onto ORIGYN primitives:

| Git | Here |
| --- | ---- |
| Repository | A collection, owned by the agent |
| Blob | A file's bytes, content-addressed by sha256 in the collection's storage |
| Commit | An NFT whose metadata is a **full snapshot manifest** of the file set |
| `HEAD`, tags | A small mutable map in collection-level metadata |

Commits are immutable NFTs. `HEAD` is the only mutable part, which is what makes rollback cheap and history impossible to quietly rewrite.

{% hint style="info" %}
Each commit stores a **complete** manifest, not a diff. Reading any version is one lookup, with no chain to replay. Unchanged files are deduped by hash, so a full snapshot does not mean a full re-upload.
{% endhint %}

## Creating a memory

**`init_memory` — paid, ~15,000 OGY. Cannot be undone.**

| Argument | Type | |
| -------- | ---- | --- |
| `name` | string | Usually the agent's identifier |
| `symbol` | string | Short ticker |
| `description` | string, optional | |

Returns the collection canister ID. Every later call takes that ID.

Creating a memory costs the same as creating any collection, because it *is* one. Commits after that are ordinary mints.

## Committing a version

**`commit_memory` — paid, cost scales with newly uploaded bytes.**

| Argument | Type | |
| -------- | ---- | --- |
| `collection_canister_id` | string | |
| `files` | array of `{ path, content }` | The **complete** file set for this version |
| `label` | string | Human-readable version name |

{% hint style="warning" %}
`files` is the whole memory at this version, not the changes since the last one. A file you leave out is a file you deleted. Read HEAD first if you are updating rather than replacing.
{% endhint %}

Each file's `content` is UTF-8 text — markdown notes, SQL, JSON, whatever the agent keeps.

The response distinguishes `uploadedFiles` from `reusedFiles`. Files whose sha256 matches the current HEAD are not re-uploaded, so committing a 50-file memory after editing one of them costs one file's worth of bytes, not fifty.

Each commit records its `parent` and a monotonic `seq`, so history is a chain you can walk back to the root.

## Reading

**`list_versions`** — free. Returns versions newest-first by walking HEAD's ancestry, plus the current `head` and any named tags. Each entry carries `seq`, `label`, `createdAt`, the file paths present, and whether it is HEAD.

**`read_version`** — free. Returns a version's full file set as UTF-8 text with each file's sha256. Defaults to HEAD.

| Argument | Type | Default |
| -------- | ---- | ------- |
| `collection_canister_id` | string | |
| `commit_token_id` | string, optional | HEAD |

The sha256 travels with the content, so a consumer can verify what it read matches what was committed.

## Tagging and rollback

**`tag_version`** — free, idempotent. Points a name at a commit.

```
tag_version(collection_canister_id, tag: "last-good", commit_token_id: "7")
```

**`rollback_memory`** — free, idempotent. Moves HEAD to an earlier commit, restoring that version's whole file set.

History is append-only. Rolling back moves a pointer; the commits you rolled past remain readable, and you can move HEAD forward again.

{% hint style="info" %}
Neither tagging nor rollback costs OGY. They write collection metadata, not certificates. Tag a known-good version before letting an agent commit experimental state — recovery is then free and instant.
{% endhint %}

## A typical loop

1. `read_version` at startup to load the agent's state.
2. Work.
3. `commit_memory` with the full file set and a descriptive label.
4. `tag_version "last-good"` once the result is verified.
5. `rollback_memory` to that tag if a later session goes wrong.

## Things to know before relying on it

* **Commits are public.** Collection data is on-chain and readable by anyone. Do not commit secrets, credentials, or personal data. Encrypted content is not exposed through MCP.
* **Every commit is a paid mint.** Commit at meaningful checkpoints, not on every token of agent output.
* **Text only.** `content` is UTF-8; there is no binary path through these tools.
* **`HEAD` is per collection.** One memory per agent. Concurrent processes committing to the same collection will race on HEAD.
