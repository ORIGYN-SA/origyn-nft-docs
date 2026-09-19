---
icon: eye
---

# Reading Private Content

{% hint style="warning" %}
**Private content works only through the HTTP API for now.** Decrypted content is returned only by the authenticated gateway endpoints on this page, called with your [API key](../rest-api/api-keys.md). Public reads and direct canister calls return ciphertext.
{% endhint %}

The gateway decides, per certificate and per caller, which private items the caller may read (see [Who can read what](overview.md#who-can-read-what)). It then returns private **fields** already decrypted, and for private **files** it returns a key the caller uses to decrypt the file itself.

## Endpoints that decrypt

All under `https://gateway.origyn.com/gateway/v1/nft/production/`, all requiring `Authorization: Bearer <credential>`:

| Endpoint | Returns |
| -------- | ------- |
| `GET /collections/{id}/nfts/{token_id}` | One certificate, read live from its collection canister |
| `GET /nfts` | Certificates you minted or hold |
| `GET /owners/{principal}/nfts` | Certificates a principal minted or holds |
| `GET /orgs/{org_id}/nfts` | Certificates an organization issued |

Each returns certificates in the usual shape. The certificate JSON is under `metadata`, and its `private` block (`metadata.private`) is rewritten for you. Every certificate on these endpoints has a `private` field, which is `null` when it has no private content.

`GET /orgs/{org_id}/nfts` accepts the organization's id or its slug. None of these endpoints require membership: anyone with a credential can list them, and only the private content is decided per caller.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/collections/{id}/nfts/{token_id}" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/orgs/{org_id}/nfts" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

{% hint style="info" %}
A caller who may read nothing still gets `200`, with `"access": "none"`. Calling without a credential is `401`. A temporary key service failure is `502 key_unavailable`, never a silent lock, so retry it.
{% endhint %}

## The `private` block

The value of `access` tells you what you received.

**No private content on this certificate:**

```json
"private": null
```

**Full access** (a member of the organization):

```json
"private": {
  "access": "full",
  "blob": "<ciphertext, as stored>",
  "fields": {
    "purchase_price": { "content": { "en": "61 200 CHF" } }
  },
  "files": {
    "lab_report": [
      {
        "name": "Lab report.pdf",
        "url": "https://<collection_canister_id>.raw.icp0.io/77/p/0c4e9b7a5d1f4e2a9b3c8d7e6f5a4b3c",
        "size": 3355443,
        "sha256": "<hex>",
        "chunk_size": 1048560,
        "enc_size": 3355507,
        "key": "<base64 file key>"
      }
    ]
  }
}
```

**Partial access** (a reader group member or the certificate holder): the same shape, containing only the items granted to you. `fields` or `files` may be empty.

```json
"private": { "access": "partial", "blob": "...", "fields": {}, "files": { "lab_report": [ ... ] } }
```

**No access:** no `fields`, no `files`, and no file names.

```json
"private": { "access": "none", "blob": "..." }
```

**Unreadable:** the organization is suspended, or the block could not be opened for this collection.

```json
"private": { "access": "none", "blob": "...", "error": "unreadable" }
```

## Decrypting a private file

A file entry gives you everything needed: `url`, `key`, `chunk_size`, `size` and `sha256`. Decryption happens on your side; the file's bytes never pass through the gateway on the way to you.

The format:

| Part | Value |
| ---- | ----- |
| Cipher | AES-256-GCM, with `key` (base64, 32 bytes) |
| Chunks | The stored file is a sequence of chunks of `chunk_size + 16` bytes (1,048,576); the last may be shorter. Chunk `i` starts at byte `i * (chunk_size + 16)`. Each chunk is its ciphertext followed by the 16-byte tag. |
| Nonce | 12 bytes: ASCII `CLPC` followed by the chunk index as an unsigned 64-bit big-endian integer |
| Additional data | A UTF-8 string, see below |

The additional data for chunk `i` is:

```
clp1|<collection_canister_id>|<file_path>|<i>
```

where `file_path` is the path part of `url` without the leading slash, for example `77/p/0c4e9b7a5d1f4e2a9b3c8d7e6f5a4b3c`, and `i` is written in decimal.

After decrypting every chunk, check that the plaintext length equals `size` and its SHA-256 equals `sha256`.

In a browser or in Node 20+, with Web Crypto:

```js
async function decryptPrivateFile(file, collectionCanisterId) {
  const filePath = new URL(file.url).pathname.replace(/^\/+/, "");
  const key = await crypto.subtle.importKey(
    "raw",
    Uint8Array.from(atob(file.key), (c) => c.charCodeAt(0)),
    "AES-GCM",
    false,
    ["decrypt"],
  );

  const ciphertext = new Uint8Array(await (await fetch(file.url)).arrayBuffer());
  const cipherChunk = file.chunk_size + 16;
  const parts = [];

  for (let i = 0, offset = 0; offset < ciphertext.length; i++, offset += cipherChunk) {
    const iv = new Uint8Array(12);
    iv.set(new TextEncoder().encode("CLPC"));
    new DataView(iv.buffer).setBigUint64(4, BigInt(i));

    const plain = await crypto.subtle.decrypt(
      {
        name: "AES-GCM",
        iv,
        additionalData: new TextEncoder().encode(`clp1|${collectionCanisterId}|${filePath}|${i}`),
      },
      key,
      ciphertext.subarray(offset, offset + cipherChunk),
    );
    parts.push(new Uint8Array(plain));
  }

  const blob = new Blob(parts);
  if (blob.size !== file.size) throw new Error("size mismatch");

  const digest = await crypto.subtle.digest("SHA-256", await blob.arrayBuffer());
  const hex = Array.from(new Uint8Array(digest), (b) => b.toString(16).padStart(2, "0")).join("");
  if (hex !== file.sha256.toLowerCase()) throw new Error("sha256 mismatch");

  return blob;
}
```

A chunk that fails to decrypt means the file was corrupted, reordered, or the wrong key or path was used; GCM does not tell these apart.

Because chunks sit at fixed offsets, you can also fetch and decrypt a single chunk with an HTTP `Range` request instead of downloading the whole file.

{% hint style="warning" %}
**Treat file keys as secrets.** Keep them in memory only, never log or persist them. A key stays valid for its file forever: removing a reader from a group stops the gateway from handing out the key again, but it cannot invalidate a key already given out.
{% endhint %}

## Listing uploaded files

Two authenticated lists show the files uploaded into collections, with private files marked:

* `GET /orgs/{org_id}/uploads`: files uploaded into an organization's collections. `org_id` may be the organization's id or its slug.
* `GET /owners/{principal}/uploads`: files a principal uploaded.

Both accept `?private=include` (the default), `exclude` or `only`, plus `collection`, `status`, `sort`, `order`, `limit` and `offset`.

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/orgs/{org_id}/uploads" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

Each item gains a `private` field:

| Value | Meaning |
| ----- | ------- |
| `null` | A public file |
| `{ "access": "full", "size": ..., "chunk_size": ..., "key": "<base64>" }` | A private file, and you are an active member of the organization |
| `{ "access": "none", "size": ..., "chunk_size": ... }` | A private file you cannot decrypt from this list |

`size` is the plaintext size. Only organization members receive keys from these lists; reader group members and certificate holders get their file keys from the certificate reads above.
