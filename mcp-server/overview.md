---
icon: plug
---

# MCP Server Overview

`origyn-nft-mcp` is a [Model Context Protocol](https://modelcontextprotocol.io) server that exposes the ORIGYN NFT lifecycle as tools an AI agent can call. Point Claude Code, Claude Desktop, or Cursor at it and the agent can create collections, mint certificates, read them back, and transfer them — paying in OGY from a wallet of its own.

It is a third way in, alongside the [REST API](../rest-api/overview.md) and direct canister calls.

| | REST API | Direct canister | MCP server |
| --- | -------- | --------------- | ---------- |
| Caller | Your backend | Your backend | An AI agent |
| Auth | API key | Your IC identity | The agent's own identity |
| Paying | Approve once in the dashboard | `icrc2_approve` yourself | Per-call approve, capped |
| Best for | Existing HTTP stacks | On-chain integrations | Agentic workflows |

{% hint style="info" %}
Collections the agent creates are ordinary ORIGYN NFT canisters. You can read and manage them over REST or with `dfx` exactly as if you had created them yourself.
{% endhint %}

## What the agent is actually doing

Every paid operation goes through the same orchestrated sequence, and it is worth understanding before you hand an agent a funded wallet:

1. Query the canister for the real cost (`estimate_mint_cost`) — free.
2. Check the request against the server's own spend policy.
3. Sign an `icrc2_approve` on the OGY ledger for **exactly** that amount, expiring in 60 seconds.
4. Make the canister call.

The agent never holds a standing allowance. Between calls, its approved amount is zero.

## Three layers of spend defense

Giving a language model a wallet is only reasonable if a single bad decision — or a single bug — cannot empty it. Three independent layers bound the damage, and each one works even if the others fail.

| Layer | Where it lives | What it stops |
| ----- | -------------- | ------------- |
| **1. Host approval** | Your MCP host | Every paid tool is marked `destructiveHint`, so Claude Code prompts you before the call |
| **2. Server policy** | The server process | Per-call cap, session budget, per-tool rate limits, canister allowlist — checked *before* anything is signed |
| **3. Ledger allowance** | The OGY ledger | The approve is for one exact amount and expires in 60 seconds; the ledger enforces it regardless of what the server does |

{% hint style="warning" %}
Layer 1 is the one you can switch off. If you enable auto-approve in your host, the agent can spend up to the **session budget** without asking again. Set `sessionBudgetOgyE8s` to an amount you are willing to lose before you do that.
{% endhint %}

{% hint style="info" %}
The session budget lives in memory and **resets when the server restarts**. It is a per-session bound, not a daily or monthly one.
{% endhint %}

## Installing

The server ships as two packages: `@origyn/nft-sdk-core` (MCP-agnostic SDK) and `@origyn/nft-mcp-server` (the stdio server and CLI). Build and link the CLI:

```bash
pnpm install
pnpm -F @origyn/nft-mcp-server build
pnpm -F @origyn/nft-mcp-server link --global
```

`origyn-nft-mcp` is now on your `PATH`.

### One-shot host setup

```bash
origyn-nft-mcp install claude-code --network mainnet
```

That single command:

* registers the server with the host's MCP config,
* writes `~/.origyn-nft/config.json` with the canister IDs for that network,
* creates `~/.origyn-nft/identity.json` (mode `600`) if none exists,
* prints the agent's principal and the OGY ledger to fund it on.

Hosts: `claude-code`, `claude-desktop`, `cursor`. Networks: `mock`, `staging`, `mainnet` (alias `live`).

Run it without `--network` on a terminal and you get an interactive prompt instead. `--force` overwrites an existing config; the identity file is never regenerated, so a funded principal cannot be lost this way.

### Try it with no money first

`--network mock` runs against a deterministic in-process stub. No canisters, no OGY, no funding. Every tool responds with plausible data so you can watch the agent walk the whole flow before any of it is real.

```bash
origyn-nft-mcp install claude-code --network mock
```

### Fund the principal

Send OGY to the principal printed during install, using any ICRC-1 client. The agent needs enough for at least one `create_collection` (~15,000 OGY) plus some mints.

```bash
origyn-nft-mcp wallet | jq .ogyBalanceE8s
```

{% hint style="warning" %}
Fund this principal with a **working balance, not a treasury**. It is a hot key on your machine; the caps bound a runaway agent, not a stolen laptop.
{% endhint %}

### Restart the host

MCP stdio servers are spawned once when the host starts. Quit and relaunch Claude Code (or your host) before the tools appear.

## Configuration

`~/.origyn-nft/config.json`, or wherever `ORIGYN_NFT_CONFIG` points. Every field has a default, so a missing file is valid and yields mainnet-pointed defaults.

```json
{
  "network": { "host": "https://icp0.io", "fetchRootKey": false },
  "canisters": {
    "claimlink":  "uasjq-dyaaa-aaaas-qdwka-cai",
    "ogyLedger":  "lkwrt-vyaaa-aaaaq-aadhq-cai",
    "origynNft":  "2evjk-yiaaa-aaaal-qct3a-cai"
  },
  "identity": { "managed": true, "path": "~/.origyn-nft/identity.json" },
  "policy": {
    "maxSpendPerCallOgyE8s": "2000000000000",
    "sessionBudgetOgyE8s":   "20000000000000",
    "rateLimits": { "create_collection": 5, "mint_nft": 500 },
    "approveExpiryMs": 60000
  }
}
```

| Field | What it does |
| ----- | ------------ |
| `policy.maxSpendPerCallOgyE8s` | Hard ceiling on any single paid call |
| `policy.sessionBudgetOgyE8s` | Total the process may spend before restart |
| `policy.rateLimits` | Per-tool call counters for the session |
| `policy.approveExpiryMs` | Lifetime of each ledger approval |
| `identity.managed` | `true` generates a key on first run; `false` requires one to already exist and never writes |

Amounts are strings in e8s (1 OGY = 100,000,000 e8s) because they exceed what JSON numbers represent safely.

Inspect what is actually in effect:

```bash
origyn-nft-mcp config print      # merged user config + defaults
origyn-nft-mcp config validate   # exit 0 if valid, 1 with the error
```

## Identity

The agent signs with an `Ed25519KeyIdentity` stored as JSON at `~/.origyn-nft/identity.json`, mode `600`. This is a distinct principal from your personal wallet, and that separation is the point: it owns only what you send it.

Override with `ORIGYN_NFT_IDENTITY_JSON` or `ORIGYN_NFT_IDENTITY_PEM` if you manage keys elsewhere.

## Driving it

Once the host has restarted, talk to the agent normally:

* *"What's my wallet state?"* → `wallet_info`
* *"Estimate the cost of minting 10 certificates totalling 4 MB"* → `estimate_cost`
* *"Create a collection called Field Reports and mint one certificate into it"* → the full flow

See the [Tool Reference](tools.md) for everything it can call, and [Agent Memory](agent-memory.md) for using a collection as versioned, on-chain agent state.

## Troubleshooting

| Error | Cause | Fix |
| ----- | ----- | --- |
| `InsufficientFunds` | The agent's principal is out of OGY | Re-fund the address from `origyn-nft-mcp wallet` |
| `SpendCapExceeded` | Per-call cap or session budget hit | Raise the limit in config, or split into smaller calls |
| `AllowlistDenied` | A canister outside `canisterAllowlist` was targeted | Add it to config if it is legitimate — otherwise investigate, this should be rare |
| Tools don't appear | Host was not restarted after install | Quit and relaunch the host |

{% hint style="warning" %}
Cost figures in the tool descriptions (`~15,000 OGY` and so on) are illustrative defaults, not quotes. Real cost comes from the canister's `estimate_mint_cost` query at call time — use `estimate_cost` if the number matters.
{% endhint %}

## Current limits

* **Stdio transport only.** No HTTP MCP endpoint yet, so the server runs locally alongside the host.
* **One identity per process.** Multi-tenant hosted wallets are a separate product.
* **Session budget is in memory.** Restarting resets it; there are no rolling daily or weekly budgets.
* **No encrypted content.** vetKeys-based private NFT content is not exposed through MCP.
* **No claim-link distribution.** The agent mints directly to recipient principals rather than generating claim links.
