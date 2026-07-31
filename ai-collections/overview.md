---
icon: robot
---

# AI Collections

AI collections are a lighter-weight path for minting certificates when you do not need your own template or your own collection canister.

They differ from standard collections in three ways that matter:

| | Standard collection | AI collection |
| --- | ------------------- | ------------- |
| Who can create one | Authorized principals only | Any signed-in principal |
| Template | Yours, registered in advance | A shared built-in template |
| Metadata validation | Validated against your template | **Not validated** |

{% hint style="warning" %}
Certificates minted this way are **not checked against a template**. Your JSON must still parse and stay within the per-item size cap, but no field is required and no shape is enforced, so a missing or misshapen field is stored exactly as sent. Validate your metadata yourself; the usual safety net is not there.
{% endhint %}

You still pay the normal OGY fees. That cost is what keeps the open-to-anyone path from being abused.

## When to use this

Use an AI collection when you want to mint certificates without designing a template or running your own collection, and when you can guarantee your metadata is well formed.

Use a [standard collection](../minting-studio/getting-started.md) when you want your own branding, your own template, template validation, and control over who can mint.

{% hint style="info" %}
The two paths do not mix. `initialize_mint` refuses AI collections, and `agent_initialize_mint` cannot target a standard one. Choose per collection.
{% endhint %}

## Two ways to use it

You can either mint into the **shared** AI collection that ORIGYN operates, or create your **own** AI collection.

### Minting into the shared collection

Pass `collection_canister_id = null` and your certificates go into the shared "Origyn AI" collection. Nothing to create first.

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai agent_initialize_mint '(record {
  collection_canister_id = null;
  num_mints = 5 : nat64;
  total_file_size_bytes = 2000000 : nat
})'
```

Find its canister id at any time:

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai get_ai_collection '()'
```

### Creating your own

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai agent_create_collection '(record {
  name = "My AI Collection";
  description = "Generated certificates";
  symbol = "MAI"
})'
```

Note there is no `template_id` and no `categories`: the built-in template is used, and the standard OGY creation fee applies. Poll `get_collection_info` for the resulting canister id, exactly as with a standard collection.

Then target it explicitly:

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai agent_initialize_mint '(record {
  collection_canister_id = opt principal "<your_ai_collection_canister_id>";
  num_mints = 5 : nat64;
  total_file_size_bytes = 2000000 : nat
})'
```

## Minting

Upload files with `agent_proxy_init_upload`, `agent_proxy_store_chunk`, and `agent_proxy_finalize_upload`. These behave exactly like their [standard counterparts](../minting-studio/minting.md#step-3-upload-files).

Then mint:

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai agent_mint_nfts '(record {
  mint_request_id = <your_mint_request_id> : nat64;
  mint_items = vec {
    record {
      token_owner = record { owner = principal "<recipient>"; subaccount = null };
      json_metadata = "<json string>";
      memo = null
    }
  }
})'
```

The JSON envelope is the same as for standard collections: display keys at the top level, field values under `data`. See [Producing your mint JSON](../minting-studio/minting.md#producing-your-mint-json-from-a-template).

The difference is that it is not validated against a template. The JSON must still parse and stay within the per-item size cap, but no field is required and no shape is enforced.

Refunds work the same way, via `agent_request_mint_refund`.

## Attaching files after minting

This is the one capability AI collections have that standard collections do not: `append_file` attaches already-uploaded files to a certificate that has **already been minted**.

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai append_file '(record {
  collection_canister_id = principal "<ai_collection_canister_id>";
  requests = vec {
    record {
      token_id = 1 : nat;
      entries = vec { record { "attachment_1"; "<finalized_file_path>" } }
    }
  }
})'
```

Each entry maps a name to the path of a file you have already uploaded and finalized. Standard collections can only attach files at mint time.

{% hint style="warning" %}
`append_file` has **no REST equivalent** today. If you are integrating over HTTP, this is the one operation you cannot perform without calling the canister directly.
{% endhint %}

## Telling AI collections apart

AI collections appear in the normal collection listings alongside standard ones, flagged with `is_ai = true`. Filter on that field if you want one kind or the other.

Over HTTP, use the `collection_type` query parameter:

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/collections" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

Pass `collection_type=ai` for AI collections only, or `normal` to exclude them.
