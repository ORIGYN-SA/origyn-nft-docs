---
icon: arrow-right-arrow-left
---

# Adding it to a live collection

You already mint certificates and now want some fields to be private. Here is exactly what can and cannot be changed, and what it means for the integration you have.

## The one rule that decides everything

**Private content is written at mint time and never afterwards.** Certificates you have already minted cannot gain it. Anything already public stays public: the chain keeps its history, so a value cannot be taken back.

So the question is only ever about the certificates you mint from now on.

## What has to change

| | Change needed |
| --- | ------------- |
| Your collection | None, if it belongs to an organization. It almost certainly does. |
| Your template | Add `"private": true` and `readers` to the fields you want hidden. This creates a new version. |
| Your minting code | Move the mint call to the HTTP gateway and send the private values in a `private` block. |
| Your reading code | Read through the authenticated gateway endpoints to get decrypted values. |
| Certificates already minted | Nothing can be done. Re-mint them if the data must be private. |

## Step 1: Confirm the collection belongs to an organization

Private content only works in organization collections; anything else answers `400 not_an_org_collection`. Every collection created through the Minting Studio since organizations shipped belongs to one, and collections that predate organizations were moved into the organization of the principal that owned them.

Check by listing your organization's collections and looking for yours:

```bash
curl "https://gateway.origyn.com/v1/nft/production/orgs/$ORG_ID/collections"
```

If it is not there, contact ORIGYN before going further.

## Step 2: Decide who may read

Organization members read everything private, whatever their role. For anyone else, create a [reader group](reader-groups.md) and name it in the template, or grant the certificate's current holder with the built-in `owner`.

```bash
curl -X POST "https://gateway.origyn.com/gateway/v1/nft/production/orgs/$ORG_ID/groups" \
  -H "Authorization: Bearer $ORIGYN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "group_id": "insurer", "name": "Insurer" }'
```

## Step 3: Add private fields to the template

Edit the template and mark the fields. Use `from_version` if a grant should apply only from this version on.

```json
{
  "id": "purchase_price",
  "type": "input",
  "label": "Purchase price",
  "private": true,
  "readers": [{ "group": "insurer" }, { "group": "owner" }]
}
```

Saving creates a **new template version**. Old certificates keep pointing at the version they were minted with, so nothing about them changes. New certificates use the newest version automatically.

Two limits to know before you start:

* **A template holds at most 10 versions**, and versions are never deleted. If yours is already at 10, this edit is refused with `409 too_many_template_versions`, and your only route is a new template.
* **A collection's template cannot be swapped.** `update_collection_metadata` does not accept a `template_id`. So if you need a new template, you need a new collection too.

## Step 4: Move minting to the gateway

This is the real work. The Minting Studio canister refuses any certificate carrying a `private` block unless the gateway wrote it, so a `dfx`-based mint cannot produce private content at all.

What changes in the call:

* The values of private fields move **out** of `json_metadata` and into a `private` block beside it. A private field left in `data` is rejected with `400 private_in_data`.
* Private files use their own upload endpoints, and the batch reserves extra bytes for encryption with `private_file_sizes`.

The full shape is in [Minting Private Content](minting.md). Everything else about your mint code stays: the same envelope, the same `public_content`, the same batching.

{% hint style="info" %}
Adding a private field does not break your existing mint code. Extra keys are ignored, and a field you never send is simply empty, unless you also mark it `required`. Mark it required only after your new code is live.
{% endhint %}

## Step 5: Move the reads that need decrypting

Public reads keep working and return the encrypted block as stored. To get values back in the clear, read through the authenticated endpoints with your API key, for example `GET /gateway/v1/nft/production/collections/{id}/nfts/{token_id}`. Private files come with a per-file key you use to decrypt them yourself. See [Reading Private Content](reading.md).

## A safe order to do it in

1. Create the reader groups.
2. Add the private fields to the template, without `required`.
3. Deploy minting code that sends the `private` block, and mint one certificate.
4. Read it back through the gateway and confirm the right people see the right things.
5. Only then mark fields `required`, if you want them enforced.
