# 🌙 Meet the SDK

You have established communications.

Now it is time to meet the toolkit.

The Moonz SDK is the public read only TypeScript and JavaScript toolkit for understanding Moonz directly from Solana.

It knows how Moonz accounts are derived.

It knows how Moonz state is encoded.

It knows how to decode the protocol.

And it turns that information into objects your application can actually use.

{% hint style="info" %}
**The Moonz SDK does not replace Solana.**

It helps your application understand what Moonz means on Solana.
{% endhint %}

## 🧭 What Is the Moonz SDK?

At its simplest, the SDK sits between your application and Solana.

```text
Your Application
       ↓
   Moonz SDK
       ↓
   Solana RPC
       ↓
Moonz Program State
```

Your application provides a Solana RPC connection.

The SDK handles the Moonz specific work.

That includes understanding Launch State accounts, deriving protocol addresses, decoding metadata, reading reserves, calculating bonding progress and interpreting lifecycle state.

Without the SDK, you could still build all of this yourself.

You would just have considerably more decoding to do.

## 📦 The Package

The public package is:

```text
@moonz-fun/sdk
```

Current version:

```text
0.1.2
```

Install it with:

```bash
npm install @moonz-fun/sdk
```

The SDK supports Node.js 18 or newer.

## 🔭 One Token

If you already know a token mint, the main entry point is:

```ts
const token = await moonz.getToken(mint);
```

This gives your application a Moonz aware view of that token.

You can read things such as:

* Creator
* Lifecycle phase
* Quote asset
* Metadata
* Launch State
* Protocol vaults
* Protocol reserves
* Supply
* Bonding progress
* Switch state
* Integrity checks

This is where most integrations should begin.

## 🔎 Is It Moonz?

Sometimes you do not need everything.

You just need an answer.

```ts
const isMoonz = await moonz.isMoonzToken(mint);
```

This gives you a simple boolean while the SDK verifies the expected Moonz Launch State underneath.

That makes it useful for:

* Wallets
* Token scanners
* Trading bots
* Portfolio tools
* Explorers
* Analytics platforms

A mint goes in.

Moonz program state decides the answer.

## 🧬 Go Deeper

Some developers will want the decoded Launch State itself.

```ts
const state = await moonz.getLaunchState(mint);
```

This is where you start getting closer to the actual protocol.

The Launch State describes where the token currently sits inside Moonz and contains the state the rest of the SDK builds upon.

We will explore it properly later in the guide.

## 🖼️ Read Metadata

Need the name, symbol or metadata information?

```ts
const metadata = await moonz.getMetadata(mint);
```

This lets applications combine protocol state with the human friendly identity of the token.

Useful for:

* Token pages
* Wallets
* Explorers
* Trading interfaces
* Discovery tools

## 🏦 Understand the Vaults

Moonz uses deterministic protocol accounts throughout the token lifecycle.

The SDK can identify the relevant vaults for a token.

```ts
const vaults = await moonz.getVaults(mint);
```

It can also read their reserves.

```ts
const reserves = await moonz.getReserves(mint);
```

This becomes important when you are building analytics, explorers or holder systems that need to understand the difference between a normal wallet and a Moonz protocol account.

## 🔭 Discover the Moonz

You do not need to already know a mint.

The SDK can discover Moonz tokens directly from Moonz program accounts.

```ts
const tokens = await moonz.getTokens();
```

You can narrow that search.

For example:

```ts
const tokens = await moonz.getTokens({
  phase: "BONDING"
});
```

Or:

```ts
const tokens = await moonz.getTokens({
  quoteAsset: "SOL"
});
```

Or find launches from one creator.

```ts
const tokens = await moonz.getTokensByCreator(
  "CREATOR_WALLET"
);
```

This is where scanners, discovery pages and creator dashboards start becoming possible.

## 📊 Read Market Data

Applications often want the market view rather than every raw protocol field.

The SDK exposes:

```ts
const market = await moonz.getMarketData(mint);
```

This converts the current Moonz token state into canonical Moonz market data.

Later we will look at exactly what that object contains and how it changes between bonding and the Moonz AMM.

## ⚡ Listen to Moonz

The SDK is not limited to asking questions.

It can listen.

Watch one token:

```ts
const stop = await moonz.watchToken(
  mint,
  {
    events: [
      "BUY",
      "SELL",
      "MIGRATED"
    ],

    onEvent(event) {
      console.log(event);
    }
  }
);
```

Or watch Moonz activity across the protocol:

```ts
const stop = await moonz.watch({
  events: [
    "TOKEN_CREATED",
    "BUY",
    "SELL",
    "MIGRATED"
  ],

  onEvent(event) {
    console.log(event);
  }
});
```

When you are finished listening:

```ts
await stop();
```

This is the beginning of live feeds, alert bots, trading terminals and realtime analytics.

## 🕰️ Look Back in Time

Already have a Solana transaction signature?

The SDK can decode Moonz events from it.

```ts
const events = await moonz.getTransactionEvents(
  signature
);
```

If your infrastructure already has Solana logs, you can decode those directly too.

```ts
const events = moonz.parseLogs(
  logMessages,
  {
    signature,
    slot,
    blockTime
  }
);
```

This is useful for historical indexing and integrations that already process Solana transactions themselves.

## 🛰️ Derive Protocol Addresses

The SDK also exposes Moonz PDA helpers.

Developers can derive addresses such as:

* Launch State
* Sale Vault
* LP Vault
* Treasury WSOL Vault
* Treasury USDC Vault
* Escrow SOL
* Mint Authority
* Creator Fees
* Metadata

For example:

```ts
const addresses = deriveMoonzAddresses(mint);
```

This is especially useful for explorers, analytics platforms and integrations that want to verify protocol relationships themselves.

## 🛡️ Trust, Then Verify

The SDK does not only decode Moonz.

It also exposes integrity information.

When you call:

```ts
const token = await moonz.getToken(mint);
```

you can inspect:

```ts
token.integrity
```

and the combined result:

```ts
token.integrity.all
```

These checks cover relationships such as program ownership, expected PDAs, vault authorities and vault mints.

That gives integrations more context than simply trusting that an account looks right.

## 🛠️ What Can You Build From Here?

You now have several paths.

### Want to inspect one token?

Use `getToken()`.

### Want to recognise Moonz?

Use `isMoonzToken()`.

### Want to find tokens?

Use `getTokens()`.

### Want market information?

Use `getMarketData()`.

### Want live activity?

Use `watch()` or `watchToken()`.

### Want historical activity?

Use `getTransactionEvents()` or `parseLogs()`.

### Want deeper protocol access?

Use the Launch State and PDA helpers.

{% hint style="success" %}
**You do not need to use every part of the SDK.**

Use the smallest part that solves the problem you are building.
{% endhint %}

## 🚀 Where We Go Next

Now we stop looking at the whole toolkit and start using it properly.

Next stop:

**Read a Moonz Token**

We will take one mint, pull its Moonz information from Solana and break down exactly what comes back.
