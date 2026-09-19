---
icon: puzzle-piece
metaLinks:
  alternates:
    - https://app.gitbook.com/s/yE16Xb3IemPxJWydtPOj/getting-started/templates
---

# Templates

Templates define the structure, layout, and field types of your certificates (ORIGYN NFTs). Every collection is created from a template, and every certificate minted in that collection follows the template's structure.

## Visual Template Builder (Recommended)

> **The fastest and easiest way to create templates** is the [Minting Studio Template Builder](https://arturshirokov.github.io/claimlink-template-builder/). It provides a visual drag-and-drop interface for designing templates, previewing them in real-time, and downloading the ready-to-use JSON file. This approach eliminates manual JSON editing, reduces errors significantly, and is the recommended starting point for all users.

With the Template Builder you can:

- Design certificate layouts visually with drag-and-drop
- Add fields, images, sections, and backgrounds
- Preview your certificate in real-time
- Configure multi-language support
- Export the final JSON file for registration via the API

For most users, the Template Builder is all you need. The sections below cover the underlying JSON structure for advanced use cases or programmatic template creation.

---

## Templates and Certificate Types

A collection declares a `certificate_type` when it is created (`"standard"` or `"dpp"`, see
[Collections & Certificates](collections-and-certificates.md)), and that choice determines the
structure the template should have and how the certificate is rendered.

**Nothing cross-checks the pairing.** Templates are not themselves typed: `create_template` takes
raw JSON, and creating a collection with `certificate_type = "dpp"` does not verify that
`template_id` points at a DPP-shaped template. Pairing the right template with the right certificate
type is yours to get right.

Certificates are still validated against whatever template the collection references at mint time,
so a mismatched pairing surfaces as mint-time validation errors, not at collection creation.

## Template JSON Structure

A template is stored as a JSON string. At the top level:

```json
{
  "name": "My Certificate Template",
  "description": "Template for gold bar certificates",
  "category": "manual",
  "structure": {
    "sections": [...],
    "languages": [...],
    "translations": {...},
    "background": {...},
    "searchIndexField": "serial_number"
  }
}
```

| Field         | Type              | Description                                                    |
| ------------- | ----------------- | -------------------------------------------------------------- |
| `name`        | string            | Template display name                                          |
| `description` | string            | Brief description of the template's purpose                    |
| `category`    | string            | One of `manual`, `ai`, `existing`, or `preset`                 |
| `structure`   | object            | The full template definition (sections, languages, background) |
| `thumbnail`   | string (optional) | Base64 data URI for a preview thumbnail                        |

### Sections

Templates are organized into sections. There is no limit on the number of sections, and each can be renamed. The two standard sections are:

- **Certificate** The visual certificate tab. Contains the fields displayed prominently on the certificate.
- **Information** The detailed data tab. Contains additional metadata and supporting information.

Beyond these two, you can declare additional custom sections (for example, a "Provenance" or "Service History" tab). Each section has its own `id`, `name`, `order`, and `items` array, exactly like the standard sections.

```json
{
  "sections": [
    {
      "id": "certificate",
      "name": "Certificate",
      "order": 0,
      "items": [...]
    },
    {
      "id": "information",
      "name": "Information",
      "order": 1,
      "items": [...]
    }
  ]
}
```

### Field Types

Each section contains items (fields). The following field types are supported:

| Type       | Description       | Use Case                                                 |
| ---------- | ----------------- | -------------------------------------------------------- |
| `title`    | Heading text      | Section headings (h1–h4), supports alignment             |
| `input`    | Data entry field  | Text, number, date, email, URL, textarea                 |
| `badge`    | Tag/badge display | Status indicators, categories, predefined values         |
| `image`    | Image upload      | Single image with configurable aspect ratio and max size |
| `video`    | Video content     | Video with duration, autoplay, and loop controls         |
| `document` | File upload       | PDFs, Word documents, and other attachments              |
| `signature`| Signature image   | Image upload semantically marked as a signature (same `FileReference` shape as `image`) |
| `readonly` | Static text       | Immutable content that cannot be edited during minting   |

> Each field `id` becomes a top-level key in the per-NFT mint JSON. See [Minting → Producing your mint JSON from a template](minting.md#writing-the-certificate-json) for the mapping rules and a worked example.

#### Field Properties

Every field has these common properties:

```json
{
  "id": "serial_number",
  "type": "input",
  "label": "Serial Number",
  "order": 0,
  "required": true,
  "description": "Unique identifier for the asset",
  "validation": {
    "minLength": 1,
    "maxLength": 50,
    "pattern": "^[A-Z0-9-]+$",
    "errorMessage": "Must be uppercase alphanumeric with dashes"
  },
  "size": "md"
}
```

| Property      | Type    | Description                                                    |
| ------------- | ------- | -------------------------------------------------------------- |
| `id`          | string  | Unique identifier for this field                               |
| `type`        | string  | Field type (see table above)                                   |
| `label`       | string  | Display label                                                  |
| `order`       | number  | Position within the section (for drag-and-drop ordering)       |
| `required`    | boolean | Whether the field must be filled during minting                |
| `immutable`   | boolean | If true, the value cannot be changed after minting             |
| `description` | string  | Helper text shown below the field                              |
| `validation`  | object  | Validation rules (minLength, maxLength, pattern, errorMessage) |
| `size`        | string  | Display size: `sm`, `md`, or `lg`                              |
| `private`     | boolean | If true, the value is encrypted and only authorized readers can see it. See [Private Content](../private-content/overview.md) |
| `readers`     | array   | For a private field: the reader groups (or `owner`) allowed to read it. See [Marking fields private](../private-content/overview.md#marking-fields-private) |

### Tree Format (Advanced)

Templates also support a tree-based node format for more complex layouts. This format uses nested nodes with 18 node types:

**Layout nodes:** `columns`, `elements`, `section`
**Text nodes:** `title`, `subTitle`, `text`, `valueField`, `field`
**Media nodes:** `image`, `mainImage`, `multiImage`, `collectionImage`, `gallery`, `video`, `attachments`
**Special nodes:** `separator`, `history`, `certificate`

The tree format is used internally and can be created through the Visual Template Builder.

---

## Multi-Language Support

Templates can define multiple languages for field labels and values:

```json
{
  "languages": [
    { "id": "lang_en", "code": "en", "name": "English", "isDefault": true },
    { "id": "lang_fr", "code": "fr", "name": "French" },
    { "id": "lang_it", "code": "it", "name": "Italian" }
  ],
  "translations": {
    "serial_number": {
      "en": { "label": "Serial Number", "placeholder": "Enter serial number" },
      "fr": {
        "label": "Numero de serie",
        "placeholder": "Entrez le numero de serie"
      },
      "it": {
        "label": "Numero di serie",
        "placeholder": "Inserisci il numero di serie"
      }
    }
  }
}
```

When certificates are minted with multi-language templates, the certificate viewer displays a language toggle to switch between translations.

---

## Background Customization

Templates support custom backgrounds for the certificate view:

```json
{
  "background": {
    "type": "custom",
    "dataUri": "data:image/png;base64,...",
    "mediaType": "image"
  }
}
```

| Option             | Description                                    |
| ------------------ | ---------------------------------------------- |
| `type: "standard"` | Uses the default ORIGYN certificate background |
| `type: "custom"`   | Uses a custom image or video as background     |

**Size guidance:** nothing rejects a large template, but the whole call has to fit in the Internet Computer's 2 MB ingress message. Keep background images under **800 KB** and the template JSON under **1.5 MB**; above that the call fails at the network layer rather than with a clean error.

---

## Template Versions

Every saved change to a template creates a new **version** instead of overwriting the old one. A certificate remembers the version it was minted with, so editing a template never changes how an existing certificate is validated or rendered.

* The first version is `1`. Templates created before versioning existed start at version `1` with their current content.
* Saving JSON identical to the current version creates nothing and returns that version. Differences in whitespace or key order alone do not count as a change.
* A template holds at most **10 versions**. Versions are never deleted, because certificates keep pointing at them. Once a template has 10, a changed save is refused with `TooManyVersions` (`409 too_many_template_versions` over REST) and the template is left unchanged. To keep editing, create a new template from the latest JSON.
* A collection always uses the **newest** version for new certificates. Its template URL points at the newest version.

### How a certificate pins its version

When you mint, the Minting Studio writes the version into the certificate JSON as a top-level `template` key:

```json
{
  "name": "Gold Bar #001",
  "template": { "id": 7, "version": 3 },
  "data": { ... }
}
```

You normally leave `template` out and get the current version. To mint against an older version, set `template` yourself; the certificate is then validated against that version.

| What you send | Result |
| ------------- | ------ |
| No `template` key | Validated against the current version, and `template` is added with that version |
| `{ "id": <this collection's template>, "version": n }` | Validated against version `n` |
| A version that does not exist | `InvalidMetadata` (`400 invalid_metadata`) |
| The id of a different template | `InvalidMetadata` (`400 invalid_metadata`) |
| A partial or malformed `template` (for example only `id`) | Ignored and replaced with the current version |

One batch may mix versions. The 50 KiB per-item limit applies to the JSON after `template` has been added. Certificates minted before versioning existed carry no `template` key and are treated as version `1`.

### Reading a specific version

To render a certificate with the template it was minted with, read `template.version` from the certificate and request that version:

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/templates/{template_id}" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

Pass `?version=n`; omit it for the newest version. The response carries `version` (the one returned), `current_version` (the newest) and `template_url`. The same `?version=` parameter works on `GET /collections/{canister_id}/template`.

Every version of a template, newest first:

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/templates/{template_id}/versions" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

Each version is also served as a static file by the Minting Studio canister, cacheable forever because a version never changes:

```
https://uasjq-dyaaa-aaaas-qdwka-cai.raw.icp0.io/templates/<template_id>/v<version>.json
```

---

## Template API Reference

All commands below use the Minting Studio canister ID `uasjq-dyaaa-aaaas-qdwka-cai`.

### Register a Template

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/create_template" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

**Using dfx instead**

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai create_template '(record {
  org_id = opt <your_org_id>;
  template_json = "<your_template_json_string>"
})'
```

`org_id` is the [organization](organizations.md) that will own the template. Pass `null` to use the organization you own. Over REST, the response is `201` with the new `template_id`, `version` (`1`), `current_version` and `template_url`.

**Returns:** `template_id` (nat) on success.

**Errors:**

- `LimitExceeded { max_templates }` The organization has reached its maximum number of templates.
- `JsonError` The JSON string is malformed.
- `UnauthorizedCall` You have no organization to create the template in, or your role cannot manage templates.
- `OrgSuspended` The organization is suspended.

### Get a Template by ID

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/templates/{template_id}" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

**Using dfx instead**

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai get_template_by_id '(record {
  template_id = 1;
  version = null
})'
```

**Returns:** `Template` record with `template_id`, `template_json`, `version` and `current_version`. Pass `version = opt 2` to read a specific version; `null` returns the newest. An unknown template or version returns `TemplateNotFound`.

### List Your Template IDs

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai get_template_ids_by_owner '(record {
  owner = principal "<your_principal>"
})'
```

**Returns:** `template_ids` (vec nat) and `total_count`.

### List Your Templates (with JSON)

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/owners/{principal}/templates" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

**Using dfx instead**

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai get_templates_by_owner '(record {
  owner = principal "<your_principal>";
  pagination = record { offset = opt 0; limit = opt 2 }
})'
```

**Note:** Due to the 3 MB response limit on the Internet Computer, `limit` should be kept at 2 or less when templates contain large backgrounds. For large collections, fetch IDs first with `get_template_ids_by_owner`, then fetch individually with `get_template_by_id`.

The `*_by_owner` calls return the templates of the organization the principal **owns**. Members of an organization they do not own should list by organization instead.

### List an Organization's Templates

{% openapi src="https://gateway.origyn.com/openapi.json" path="/v1/nft/{env}/orgs/{id_or_slug}/templates" method="get" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

**Using dfx instead**

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai get_templates_by_org '(record {
  org_id = 12 : nat64;
  pagination = record { offset = opt 0; limit = opt 2 }
})'
```

`get_templates_by_org` returns at most 2 templates per call. `get_template_ids_by_org` takes the same arguments and returns only the ids.

### Update a Template

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/update_template" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

**Using dfx instead**

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai update_template '(record {
  template_id = 1;
  new_tempalte_json = "<updated_json_string>"
})'
```

The field name is misspelled in the interface. Write it as `new_tempalte_json`, exactly as shown.

Each successful change creates a new [version](#template-versions). Over REST the response carries the new `version`, `current_version` and `template_url`.

**Errors:**

- `TooManyVersions { max }` The template already has 10 versions. Create a new template to keep editing.
- `JsonError` The JSON string is malformed.
- `UnauthorizedCall` The template does not exist, or your role in the owning organization cannot manage templates.
- `OrgSuspended` The organization is suspended.

### Delete a Template

{% openapi src="https://gateway.origyn.com/openapi.json" path="/gateway/v1/nft/{env}/delete_template" method="post" %}
https://gateway.origyn.com/openapi.json
{% endopenapi %}

**Using dfx instead**

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai delete_template '(1)'
```

Deleting removes every version. A template that a collection still uses cannot be deleted: the call returns `TemplateInUse { collection_ids }` (`409 template_in_use` over REST).

---

## Size Limits

| Limit               | Value        | Reason                                        |
| ------------------- | ------------ | --------------------------------------------- |
| Max template JSON   | ~1.5 MB      | Guidance, not enforced: the call must fit the IC 2 MB ingress message |
| Background images   | ~800 KB      | Guidance, not enforced: keeps the template inside that limit |
| Templates per organization | Configurable | Enforced by the Minting Studio canister  |
| Versions per template | 10         | Versions are never deleted                    |
| Pagination limit    | 2 per query  | Avoids exceeding the 3 MB IC response limit   |
