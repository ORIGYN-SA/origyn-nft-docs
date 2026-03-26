---
icon: puzzle-piece
metaLinks:
  alternates:
    - https://app.gitbook.com/s/yE16Xb3IemPxJWydtPOj/getting-started/templates
---

# Templates

Templates define the structure, layout, and field types of your certificates (ORIGYN NFTs). Every collection is created from a template, and every certificate minted in that collection follows the template's structure.

## Visual Template Builder (Recommended)

> **The fastest and easiest way to create templates** is the [Minting Studio Template Builder](https://ahegaoburger.github.io/claimlink-template-builder/). It provides a visual drag-and-drop interface for designing templates, previewing them in real-time, and downloading the ready-to-use JSON file. This approach eliminates manual JSON editing, reduces errors significantly, and is the recommended starting point for all users.

With the Template Builder you can:

* Design certificate layouts visually with drag-and-drop
* Add fields, images, sections, and backgrounds
* Preview your certificate in real-time
* Configure multi-language support
* Export the final JSON file for registration via the API

For most users, the Template Builder is all you need. The sections below cover the underlying JSON structure for advanced use cases or programmatic template creation.

***

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

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Template display name |
| `description` | string | Brief description of the template's purpose |
| `category` | string | One of `manual`, `ai`, `existing`, or `preset` |
| `structure` | object | The full template definition (sections, languages, background) |
| `thumbnail` | string (optional) | Base64 data URI for a preview thumbnail |

### Sections

Templates are organized into sections. The two standard sections are:

* **Certificate** — The visual certificate tab. Contains the fields displayed prominently on the certificate.
* **Information** — The detailed data tab. Contains additional metadata and supporting information.

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

| Type | Description | Use Case |
|------|-------------|----------|
| `title` | Heading text | Section headings (h1–h4), supports alignment |
| `input` | Data entry field | Text, number, date, email, URL, textarea |
| `badge` | Tag/badge display | Status indicators, categories, predefined values |
| `image` | Image upload | Single image with configurable aspect ratio and max size |
| `video` | Video content | Video with duration, autoplay, and loop controls |
| `document` | File upload | PDFs, Word documents, and other attachments |
| `readonly` | Static text | Immutable content that cannot be edited during minting |

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

| Property | Type | Description |
|----------|------|-------------|
| `id` | string | Unique identifier for this field |
| `type` | string | Field type (see table above) |
| `label` | string | Display label |
| `order` | number | Position within the section (for drag-and-drop ordering) |
| `required` | boolean | Whether the field must be filled during minting |
| `immutable` | boolean | If true, the value cannot be changed after minting |
| `description` | string | Helper text shown below the field |
| `validation` | object | Validation rules (minLength, maxLength, pattern, errorMessage) |
| `size` | string | Display size: `sm`, `md`, or `lg` |

### Tree Format (Advanced)

Templates also support a tree-based node format for more complex layouts. This format uses nested nodes with 18 node types:

**Layout nodes:** `columns`, `elements`, `section`
**Text nodes:** `title`, `subTitle`, `text`, `valueField`, `field`
**Media nodes:** `image`, `mainImage`, `multiImage`, `collectionImage`, `gallery`, `video`, `attachments`
**Special nodes:** `separator`, `history`, `certificate`

The tree format is used internally and can be created through the Visual Template Builder.

***

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
      "fr": { "label": "Numero de serie", "placeholder": "Entrez le numero de serie" },
      "it": { "label": "Numero di serie", "placeholder": "Inserisci il numero di serie" }
    }
  }
}
```

When certificates are minted with multi-language templates, the certificate viewer displays a language toggle to switch between translations.

***

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

| Option | Description |
|--------|-------------|
| `type: "standard"` | Uses the default ORIGYN certificate background |
| `type: "custom"` | Uses a custom image or video as background |

**Size limit:** Background images should be under **800 KB**. The total template JSON must stay under **1.5 MB** to fit within the Internet Computer's 2 MB ingress message limit.

***

## Template API Reference

All commands below use the Minting Studio canister ID `uasjq-dyaaa-aaaas-qdwka-cai`.

### Register a Template

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai create_template '(record {
  template_json = "<your_template_json_string>"
})'
```

**Returns:** `template_id` (nat) on success.

**Errors:**
* `LimitExceeded` — You have reached the maximum number of templates per owner.
* `JsonError` — The JSON string is malformed.

### Get a Template by ID

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai get_template_by_id '(record {
  template_id = 1
})'
```

**Returns:** `Template` record with `template_id` and `template_json`.

### List Your Template IDs

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai get_template_ids_by_owner '(record {
  owner = principal "<your_principal>"
})'
```

**Returns:** `template_ids` (vec nat) and `total_count`.

### List Your Templates (with JSON)

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai get_templates_by_owner '(record {
  owner = principal "<your_principal>";
  pagination = record { offset = opt 0; limit = opt 2 }
})'
```

**Note:** Due to the 3 MB response limit on the Internet Computer, `limit` should be kept at 2 or less when templates contain large backgrounds. For large collections, fetch IDs first with `get_template_ids_by_owner`, then fetch individually with `get_template_by_id`.

### Update a Template

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai update_template '(record {
  template_id = 1;
  new_tempalte_json = "<updated_json_string>"
})'
```

**Note:** Only the template owner can update their templates.

### Delete a Template

```bash
dfx canister --network ic call uasjq-dyaaa-aaaas-qdwka-cai delete_template '(1)'
```

**Note:** Only the template owner can delete their templates.

***

## Size Limits

| Limit | Value | Reason |
|-------|-------|--------|
| Max template JSON | 1.5 MB | Fits within the IC 2 MB ingress message limit |
| Background images | ~800 KB | Keeps template size manageable |
| Templates per owner | Configurable | Enforced by the Minting Studio canister |
| Pagination limit | 2 per query | Avoids exceeding the 3 MB IC response limit |
