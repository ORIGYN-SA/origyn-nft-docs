---
icon: lock
---

# Private Content

{% hint style="warning" %}
**Private content works only through the HTTP API for now.** Uploading private files, minting private fields and reading them decrypted all go through `https://gateway.origyn.com` with an [API key](../rest-api/api-keys.md). Direct canister calls (`dfx` or an agent library) cannot write or read private content: the Minting Studio canister refuses a certificate that carries a private part unless the gateway wrote it.
{% endhint %}

Private content lets a certificate carry information that only chosen readers can see: a purchase price, the owner's name, a lab report, an insurance document. The rest of the certificate stays public, exactly as before.

## How it works

1. A [template](../minting-studio/templates.md) marks some of its fields **private** and names who may read them.
2. When you mint, you send the private values and files in plain form, over HTTPS, to the ORIGYN gateway. The gateway encrypts them, and only ciphertext is stored on chain.
3. When someone reads the certificate through the gateway with their own credential, the gateway works out what that caller may see and decrypts exactly that.

Each collection has its own encryption key, derived with the Internet Computer's vetKD. The Minting Studio canister releases it only to the ORIGYN gateway. Neither you nor your readers ever handle a collection key.

{% hint style="info" %}
**This is not end-to-end encryption.** The ORIGYN gateway encrypts on your behalf and decrypts for authorized readers, so it handles the plaintext. What private content guarantees is that the data is never public: everything stored on chain is encrypted, and only callers the access rules allow receive it decrypted.
{% endhint %}

## Requirements

* **A collection owned by an organization.** Private content works only in [organization](../minting-studio/organizations.md) collections. Any other collection answers `400 not_an_org_collection`.
* **A template with private fields.** See [Marking fields private](#marking-fields-private).
* **An API key** for every private write and every decrypted read.

## Who can read what

| Reader | What they see |
| ------ | ------------- |
| An active member of the collection's organization, **any role** (Viewer included) | Every private item of every certificate in the organization's collections |
| The current holder of the certificate | The items whose readers include `owner` |
| A member of a [reader group](reader-groups.md) | The items whose readers name that group |
| Anyone else | Nothing private, only the fact that a private part exists |

Holding the certificate is checked at read time, so a transfer moves `owner` access to the new holder.

{% hint style="danger" %}
**A suspended organization locks its private content for everyone, members included,** until it is reinstated.
{% endhint %}

## Marking fields private

A template item becomes private with `"private": true`. Its `readers` list says who, besides organization members, may read it.

```json
{
  "id": "lab_report",
  "type": "document",
  "label": "Lab report",
  "required": true,
  "private": true,
  "readers": [
    { "group": "insurer", "from_version": 4 },
    { "group": "owner" }
  ]
}
```

| Key | Meaning |
| --- | ------- |
| `private` | `true` makes the item private. Omitted or `false` means public. |
| `required` | As for any item: the certificate cannot be minted without a value. For a private item the value goes in the private part. |
| `readers[].group` | The id of a [reader group](reader-groups.md) in the organization, or the built-in `owner`, meaning whoever currently holds the certificate. `owner` is the only built-in name. |
| `readers[].from_version` | Optional. The grant applies only to certificates pinned to this [template version](../minting-studio/templates.md#template-versions) or later. Omit it to grant access on every certificate. |

`from_version` is how you widen access for new certificates without widening it for the ones already issued.

{% hint style="warning" %}
* **Mark private items inside `structure.sections[].items`.** Items elsewhere in the template are not recognized as private.
* **Templates are public.** Anyone can read a template, so labels and which items are private are visible. Only the values are hidden.
* **Group names in `readers` are not checked when you save a template.** A misspelled group is accepted and grants nobody.
{% endhint %}

### Which template version decides what

A certificate records the template version it was minted with. Two rules follow from that:

* **Whether an item is private** is taken from the certificate's pinned version.
* **Who may read it** is always taken from the template's **newest** version, filtered by `from_version`. Editing `readers` in a new version therefore changes access for certificates you have already minted.

## What is stored on chain

The certificate's public JSON gains a `private` block holding only ciphertext:

```json
{
  "name": "Diamond #A17",
  "template": { "id": 7, "version": 3 },
  "data": { "serial": "A17" },
  "private": { "v": 1, "salt": "<base64>", "blob": "<base64>" }
}
```

Public reads (the `/v1/nft/` HTTP endpoints, `get_nft`, ICRC-7 metadata) return this block as it is stored. Private files live in the collection at `https://<collection_canister_id>.raw.icp0.io/<mint_request_id>/p/<hex>`. That address is public, but the bytes behind it are encrypted, and the real file name is part of the private data.

## Revoking access

Removing someone from a reader group, deleting a group, or removing an organization member stops their access once the gateway's read service picks up the change, normally within a minute.

{% hint style="danger" %}
**Revocation stops future reads only.** Anything a reader already received stays with them. That includes downloaded files and the per-file keys the gateway handed them: because an encrypted file stays publicly served, a former reader who kept a file's key can still decrypt that file.
{% endhint %}

* **Deleting a group does not erase its name** from templates or certificates. The name matches nobody until a group with the same id is created again, which restores that access.
* **Reader overrides set at mint are permanent.** See [Minting Private Content](minting.md#per-certificate-readers).

## Next steps

* [Reader Groups](reader-groups.md): create groups and manage who is in them.
* [Minting Private Content](minting.md): upload private files and mint the private part.
* [Reading Private Content](reading.md): read decrypted fields and decrypt private files.
