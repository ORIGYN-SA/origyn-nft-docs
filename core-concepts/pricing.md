---
icon: coins
---

# Pricing

You pay for two things: creating a collection, and minting certificates into it. ORIGYN pays the Internet Computer cycles that keep your collection running.

Everything is charged in **OGY**, the ORIGYN token, from the organization's [billing principal](../minting-studio/organizations.md#paying-the-billing-principal).

## What it costs

| | Price | Charged when |
| --- | ----- | ------------ |
| Creating a collection | **15,000 OGY** | `create_collection` |
| Each certificate | **$0.01** worth of OGY | Reserved at `initialize_mint` |
| Storage for files | **$0.164** worth of OGY per MB | Reserved at `initialize_mint` |

Mint prices are set in US dollars and converted to OGY at the time of the call, using an on-chain price oracle. The collection fee is a flat OGY amount, so what it costs in dollars moves with the OGY price.

At the OGY price of the day this page was checked ($0.00140792), creating a collection was about **$21**.

## Worked examples

Ten certificates with 5 MB of images between them:

```
base     10 × $0.01   = $0.10
storage   5 MB × $0.164 = $0.82
total                  = $0.92  (666.63 OGY)
```

Five hundred certificates with 2 MB of images each, so 1 GB in total:

```
base    500 × $0.01     = $5.00
storage 1,000 MB × $0.164 = $164.03
total                     = $169.03  (122,459 OGY)
```

Storage dominates as soon as you attach real images, so the size of your files, not the number of certificates, is what decides the bill.

## Getting the exact number

Never guess. `estimate` charges nothing and returns the same figures the real charge will use:

```bash
curl "https://gateway.origyn.com/gateway/v1/nft/production/estimate?num_mints=10&total_bytes=5000000" \
  -H "Authorization: Bearer $ORIGYN_API_KEY"
```

```json
{
  "total_ogy_e8s": "66663099337",
  "total_usd_e8s": "92015991",
  "ogy_usd_price_e8s": "140792",
  "breakdown": { "base_fee_usd_e8s": "10000000", "storage_fee_usd_e8s": "82015991" }
}
```

All amounts are **e8s**: divide by 100,000,000 to get OGY or dollars. Here that is 666.63 OGY, about $0.92.

## How paying works

1. The billing principal approves the Minting Studio to spend OGY, once, with an ICRC-2 approval. The dashboard does this when you create an API key.
2. `initialize_mint` charges the whole reservation up front: the certificates you asked for, and the bytes you said you would upload.
3. When you settle with `close_mint_request`, what you actually used is burned and **the rest is refunded** to the billing principal, minus the ledger transfer fee (0.002 OGY). A residue smaller than that fee is burned instead of paid out.

So over-reserving storage is cheap: you get the unused part back. Under-reserving is not, because uploads stop when you hit the limit and you have already paid.

An approval is a ceiling, not a payment. Approve more than one batch needs so you are not re-approving constantly, and check what is left with `GET /orgs/{org_id}/allowance`. Below 20,000 OGY the organization's owners, admins and minters get a warning email.

{% hint style="warning" %}
The approval and the balance are different things. A call can fail with `402 insufficient_funds` while the allowance still looks generous: the approval says how much may be spent, the wallet has to actually hold it.
{% endhint %}

## Private content

Encrypting a file adds 16 bytes per 1,048,560-byte chunk, so about 0.0015%. You pay storage on the encrypted size, and `estimate` reports it as `encryption_overhead_bytes` when you pass `private_file_sizes`. See [Minting Private Content](../private-content/minting.md#1-reserve-room-for-encryption).

## What you do not pay for

* **Cycles.** ORIGYN funds the collection canister and its storage canisters.
* **Reading.** Every read endpoint is free, including the public HTTP API.
* **Templates, organizations, members and reader groups.** Only collections and certificates cost money.

Running your own collection instead ([Custom Installation](../custom-installation/setup.md)) means no OGY fees and no ORIGYN involvement, but you fund the canister's cycles yourself.
