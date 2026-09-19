---
icon: lightbulb
---

# How It Works

Before diving into technical setup, here is a high-level overview of how ORIGYN NFT collections work. Understanding these core concepts will make the rest of the documentation much easier to follow.

***

## Core Concepts

### Organizations

An **organization** is the account that issues certificates. It owns the templates and collections, and its **members** work in it with a role:

* **Owner** and **Admin** manage templates, collections and the team
* **Minter** mints certificates
* **Viewer** reads everything, including private content

The organization pays for collections and mints from one wallet, its **billing principal**. You get an organization by applying in the Minting Studio dashboard, or join one by invitation. [Organizations & Members →](../minting-studio/organizations.md)

### Templates

A **template** is the blueprint for your certificates. It defines:

* What **fields** appear on each certificate (name, serial number, weight, images, etc.)
* What **data types** each field accepts (text, number, date, image, video, document)
* How the certificate is **visually laid out** (sections, backgrounds, multi-column layouts)
* Which **languages** are supported for multi-language certificates
* Which fields are **private**, and who may read them

Every change to a template is saved as a new **version**, and each certificate remembers the version it was minted with.

Think of a template as a form design, it specifies the structure, but contains no actual data yet. You create a template once, then use it to mint as many certificates as you need.

Here are examples of what certificates look like once minted from different templates, each template defines its own layout, fields, and background, while the actual data is filled in during minting. Click any image to expand it.

<table data-column-title-hidden data-view="cards" data-full-width="false"><thead><tr><th></th><th data-hidden data-card-cover data-type="files"></th></tr></thead><tbody><tr><td><strong>Gold</strong></td><td><a href="../.gitbook/assets/certificate-example-gold.png">certificate-example-gold.png</a></td></tr><tr><td><strong>Diamonds</strong></td><td><a href="../.gitbook/assets/certificate-example-diamond.png">certificate-example-diamond.png</a></td></tr><tr><td><strong>Jewelry</strong></td><td><a href="../.gitbook/assets/certificate-example-jewelry.png">certificate-example-jewelry.png</a></td></tr><tr><td><strong>Art</strong></td><td><a href="../.gitbook/assets/certificate-example-art.png">certificate-example-art.png</a></td></tr><tr><td><strong>Sports</strong></td><td><a href="../.gitbook/assets/certificate-example-football.png">certificate-example-football.png</a></td></tr></tbody></table>

### Collections

A **collection** is a container for certificates. Technically, it is an ORIGYN NFT canister deployed on the Internet Computer. Each collection:

* Is created from a single **template**
* Has a **name**, **symbol**, and **description** (like a brand for your certificates)
* Lives on its own **canister** with a unique ID
* Can hold any number of **certificates**

When you create a collection, the system automatically provisions a canister, installs the ORIGYN NFT code, and uploads your template. This all happens in under a minute.

### Certificates (NFTs)

A **certificate** is an individual ORIGYN NFT within a collection. Each certificate:

* Contains **metadata** structured according to the collection's template
* Is **fully on-chain** that means all the data, images, and documents stored on the Internet Computer
* Follows the **ICRC-7 standard** making them transferable, queryable, and interoperable with any IC marketplace
* Has an **owner** who can transfer or manage it
* Can carry **private content**: fields and files encrypted on chain that only chosen readers can see

Certificates represent verified real-world assets like gold bars, diamonds, watches, and art while including the ORIGYN badge guaranteeing authenticity.

***

## The Complete Flow

```
0. Get Access
   └─ Apply in the Minting Studio dashboard, or accept an invitation
   └─ Your organization is created on approval
   └─ Create an API key (recommended)

1. Design a Template
   └─ Use the Visual Template Builder or write JSON manually
   └─ Define fields, sections, languages, and background
   └─ Mark fields private and choose their readers (optional)
   └─ Register the template → receive a template_id

2. Create a Collection
   └─ Choose your template
   └─ Provide a name, symbol, and description
   └─ Pay the creation fee (15,000 OGY, from the organization's billing principal)
   └─ Collection canister is automatically deployed

3. Mint Certificates
   └─ Upload files (images, documents) to the collection
   └─ Upload private files through the private upload endpoints (optional)
   └─ Provide JSON metadata for each certificate (validated server-side against the template)
   └─ Certificates become live ORIGYN NFTs with unique token IDs

4. View & Manage
   └─ Query certificate details and metadata
   └─ Read private content with your credential
   └─ Transfer certificates between owners
   └─ Manage your team, reader groups and billing
```

***

## Two Deployment Paths

There are two ways to launch an ORIGYN NFT collection, depending on your needs:

### Minting Studio (Recommended)

A **managed service** where ORIGYN handles all the infrastructure. Ideal for:

* Creators who want to focus on content, not canister management
* Projects that don't need custom smart contract logic
* Quick launches, new collections are ready in under a minute

**Cost:** 15,000 OGY per collection, plus a fee per certificate and its storage. ORIGYN pays the cycles, upgrades and infrastructure. See [Pricing](pricing.md).

You can drive it over HTTP with an API key, or by calling the canisters directly with `dfx`. Both reach the same canisters, and you can mix them; only private content is HTTP-only. [Getting Started](../minting-studio/getting-started.md) compares the two and sets you up.

### Custom Installation

A **self-managed** deployment using the open-source ORIGYN NFT canister. Ideal for:

* Developers who need full control over their smart contracts
* Projects requiring custom logic or deep integration
* Teams comfortable managing their own canisters and cycles

**Cost:** no OGY fees. You fund the canister's cycles and run the infrastructure.

[Get started with Custom Installation →](../custom-installation/setup.md)
