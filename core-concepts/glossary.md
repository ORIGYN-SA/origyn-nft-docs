---
icon: book-open
---

# Glossary

Terms you will meet in these docs, and what they mean here.

**Allowance.** How much OGY the billing principal has approved the Minting Studio to spend. A ceiling, not a payment, and not the same as the wallet's balance. See [Pricing](pricing.md).

**API key.** The credential that lets you call the HTTP API, of the form `sk_live_` plus 32 hex characters. It identifies one principal, and only works in the environment that issued it. See [Obtaining an API Key](../rest-api/api-keys.md).

**Billing principal.** The wallet that pays for an organization's collections and mints, and receives its refunds. Defaults to the Owner. See [Organizations](../minting-studio/organizations.md#paying-the-billing-principal).

**Canister.** A smart contract on the Internet Computer. Your collection is one, and it stores its files in further canisters of its own.

**Certificate.** One ORIGYN NFT: the record of a single item, with its data, images and documents. Called a token where an id is involved, and an NFT where a standard is involved. The three words mean the same object.

**Collection.** A canister holding certificates that share one template, one name and one symbol.

**Cycles.** What computation and storage cost on the Internet Computer, paid by the canister. For Minting Studio collections, ORIGYN pays them; for a Custom Installation, you do.

**DPP.** Digital Product Passport, one of the two `certificate_type` values a collection can declare. See [Certificate types](../minting-studio/collections-and-certificates.md#certificate-types).

**e8s.** An amount scaled by 100,000,000, the way both OGY and the USD figures in the estimate are expressed. 1 OGY is `100_000_000` e8s. Divide by 100,000,000 to read it.

**ICRC-2 approval.** The ledger mechanism behind the allowance: you authorize a spender, here the Minting Studio, to take up to a set amount from your account.

**ICRC-7 and ICRC-37.** The Internet Computer NFT standards a collection implements, for ownership and transfers (ICRC-7) and for approvals and delegated transfers (ICRC-37). See [ICRC-7 and ICRC-37](../technical-reference/icrc37-icrc7.md).

**Idempotency key.** A header you choose that names one paid operation, so a retry cannot charge you twice. See [Paid Requests](../rest-api/paid-requests.md).

**The index.** ORIGYN's off-chain copy of what is on chain, which serves the fast list and search endpoints. It lags a mint by a few seconds, which is why some endpoints read the canister live instead.

**Internet Identity, NFID Wallet, OISY.** Wallets you can sign in to the dashboard with. The principal they give you is not the same as a `dfx` identity's principal.

**Mint request.** A paid reservation: how many certificates you may mint and how many bytes you may upload. Settling it refunds what you did not use. See [Minting](../minting-studio/minting.md).

**OGY.** The ORIGYN token, which all fees are paid in. Its ledger canister is `lkwrt-vyaaa-aaaaq-aadhq-cai`.

**Organization.** The account that issues certificates: it owns the templates and collections, has members with roles, and pays through its billing principal.

**Principal.** An identity on the Internet Computer, written like `ijx2n-lcz52-...-dae`. Yours comes from your wallet or from `dfx identity get-principal`. Certificates are owned by principals, and API keys belong to one.

**Private content.** Certificate fields and files that are encrypted on chain and readable only by people you choose. See [Private Content](../private-content/overview.md).

**Reader group.** A named list of principals in an organization that a template can grant read access to private fields. See [Reader Groups](../private-content/reader-groups.md).

**Session token.** A short-lived alternative to an API key, valid for at most 24 hours, used by the dashboard. See [Obtaining an API Key](../rest-api/api-keys.md#session-tokens).

**Subaccount.** An optional 32-byte extension of a principal, so one identity can hold several distinct accounts. Optional everywhere in these APIs, and absent on the HTTP surface.

**Template.** The blueprint for certificates in a collection: the fields, their types and layout, which are private, and who may read them. Every edit creates a new [version](../minting-studio/templates.md#template-versions), and a certificate records the version it was minted with.

**vetKD.** The Internet Computer feature that derives the per-collection encryption key used for private content. The key is released only to the ORIGYN gateway.
