---
icon: lightbulb
---

# How It Works

Before diving into technical setup, here is a high-level overview of how ORIGYN NFT collections work. Understanding these core concepts will make the rest of the documentation much easier to follow.

---

## Core Concepts

### Templates

A **template** is the blueprint for your certificates. It defines:

- What **fields** appear on each certificate (name, serial number, weight, images, etc.)
- What **data types** each field accepts (text, number, date, image, video, document)
- How the certificate is **visually laid out** (sections, backgrounds, multi-column layouts)
- Which **languages** are supported for multi-language certificates

Think of a template as a form design, it specifies the structure, but contains no actual data yet. You create a template once, then use it to mint as many certificates as you need.

Here are examples of what certificates look like once minted from different templates, each template defines its own layout, fields, and background, while the actual data is filled in during minting. Click any image to expand it.

<table data-view="cards"><thead><tr><th></th><th data-hidden data-card-cover data-type="files"></th></tr></thead><tbody><tr><td><strong>Gold</strong></td><td><a href="../.gitbook/assets/certificate-example-gold.png">certificate-example-gold.png</a></td></tr><tr><td><strong>Diamonds</strong></td><td><a href="../.gitbook/assets/certificate-example-diamond.png">certificate-example-diamond.png</a></td></tr><tr><td><strong>Jewelry</strong></td><td><a href="../.gitbook/assets/certificate-example-jewelry.png">certificate-example-jewelry.png</a></td></tr><tr><td><strong>Art</strong></td><td><a href="../.gitbook/assets/certificate-example-art.png">certificate-example-art.png</a></td></tr><tr><td><strong>Sports</strong></td><td><a href="../.gitbook/assets/certificate-example-football.png">certificate-example-football.png</a></td></tr></tbody></table>

### Collections

A **collection** is a container for certificates. Technically, it is an ORIGYN NFT canister deployed on the Internet Computer. Each collection:

- Is created from a single **template**
- Has a **name**, **symbol**, and **description** (like a brand for your certificates)
- Lives on its own **canister** with a unique ID
- Can hold any number of **certificates**

When you create a collection, the system automatically provisions a canister, installs the ORIGYN NFT code, and uploads your template. This all happens in under a minute.

### Certificates (NFTs)

A **certificate** is an individual ORIGYN NFT within a collection. Each certificate:

- Contains **metadata** structured according to the collection's template
- Is **fully on-chain** that means all the data, images, and documents stored on the Internet Computer
- Follows the **ICRC-7 standard** making them transferable, queryable, and interoperable with any IC marketplace
- Has an **owner** who can transfer or manage it

Certificates represent verified real-world assets like gold bars, diamonds, watches, and art while including the ORIGYN badge guaranteeing authenticity.

---

## The Complete Flow

```
1. Design a Template
   └─ Use the Visual Template Builder or write JSON manually
   └─ Define fields, sections, languages, and background
   └─ Register the template → receive a template_id

2. Create a Collection
   └─ Choose your template
   └─ Provide a name, symbol, and description
   └─ Pay the creation fee (15,000 OGY via Minting Studio)
   └─ Collection canister is automatically deployed

3. Mint Certificates
   └─ Upload files (images, documents) to the collection
   └─ Provide metadata for each certificate
   └─ Certificates become live ORIGYN NFTs with unique token IDs

4. View & Manage
   └─ Query certificate details and metadata
   └─ Transfer certificates between owners
   └─ Manage collection permissions
```

---

## Two Deployment Paths

There are two ways to launch an ORIGYN NFT collection, depending on your needs:

### Minting Studio (Recommended)

A **managed service** where ORIGYN handles all the infrastructure. Ideal for:

- Creators who want to focus on content, not canister management
- Projects that don't need custom smart contract logic
- Quick launches, new collections are ready in under a minute

**Cost:** 15,000 OGY tokens per collection. ORIGYN manages cycles (gas), upgrades, and infrastructure.

[Get started with Minting Studio →](../minting-studio/getting-started.md)

### Custom Installation

A **self-managed** deployment using the open-source ORIGYN NFT canister. Ideal for:

- Developers who need full control over their smart contracts
- Projects requiring custom logic or deep integration
- Teams comfortable managing their own canisters and cycles

**Cost:** You manage your own cycles. No OGY fee, but you are responsible for all infrastructure.

[Get started with Custom Installation →](../custom-installation/setup.md)
