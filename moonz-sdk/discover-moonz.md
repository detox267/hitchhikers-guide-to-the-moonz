# 🔭 Discover Moonz

Until now, you needed to know where you were going.

You had a mint.

You pointed the SDK at it.

You read the token.

Useful.

But the Moonz are bigger than one mint.

Time to get a telescope.

`getTokens()` lets your application discover Moonz tokens directly from Moonz program state on Solana.

No private token directory.

No Moonz database required.

Just the chain.

{% hint style="info" %}
**Discovery starts with the Moonz program itself.**

The SDK reads Moonz Launch State accounts from Solana, verifies the expected deterministic Launch State addresses, decodes them and turns them into developer friendly token summaries.
{% endhint %}

## 🌌 Find the Moonz

The simplest discovery request is:

```ts
const tokens = await moonz.getTokens();
```

That returns an array of:

```text
MoonzTokenSummary
```

Each summary gives you enough information to understand and organise a Moonz launch without performing the heavier `getToken()` read for every mint.

## 🧭 What Comes Back?

A discovery result includes information such as:

* Mint
* Creator
* Lifecycle phase
* Quote asset
* Pending quote asset
* Launch State address
* Sale Vault
* LP Vault
* Treasury vaults
* Escrow vault
* Metadata address
* Sale supply
* Tokens sold
* Tokens remaining
* Bonding progress
* Last trade timestamp
* Metadata state
* Mint finalisation state
* Switching state
* PDA integrity

That makes `getTokens()` a strong starting point for scanners, discovery pages and market lists.

## 🕰️ The Default View

If you provide no sorting option, Moonz uses:

```text
LAST_TRADE_DESC
```

That means the most recently traded Moonz tokens appear first.

```ts
const tokens = await moonz.getTokens();
```

Think of it as asking:

**What has been moving most recently?**

## 🌱 Find Bonding Tokens

Want to see only tokens currently bonding?

```ts
const tokens = await moonz.getTokens({
  phase: "BONDING"
});
```

Now your application has the foundation of a bonding discovery page.

You could display:

* Mint
* Creator
* Bonding progress
* Quote asset
* Last trade time

And suddenly you have a scanner.

## 🚀 Find Multiple Phases

You can also request more than one lifecycle phase.

```ts
const tokens = await moonz.getTokens({
  phase: [
    "BONDING",
    "AMM_LIVE"
  ]
});
```

This is useful when your application wants active Moonz markets but does not care whether they are still bonding or already trading through the Moonz AMM.

## 💱 Search by Quote Asset

Want SOL markets?

```ts
const tokens = await moonz.getTokens({
  quoteAsset: "SOL"
});
```

Want USDC markets?

```ts
const tokens = await moonz.getTokens({
  quoteAsset: "USDC"
});
```

You can also provide multiple accepted quote assets.

```ts
const tokens = await moonz.getTokens({
  quoteAsset: [
    "SOL",
    "USDC"
  ]
});
```

This becomes useful for market explorers and PCLS aware applications.

## 👤 Find One Creator

Want to see launches belonging to one creator?

Use:

```ts
const tokens = await moonz.getTokensByCreator(
  "CREATOR_WALLET"
);
```

This is equivalent to using the creator filter through `getTokens()`.

```ts
const tokens = await moonz.getTokens({
  creator: "CREATOR_WALLET"
});
```

That could power:

* Creator profiles
* Launch history
* Creator dashboards
* Portfolio tools
* Analytics

One wallet in.

Its Moonz launches out.

## 🎯 Search Selected Mints

Already have a set of mints you care about?

You can provide a mint list.

```ts
const tokens = await moonz.getTokens({
  mints: [
    "TOKEN_MINT_ONE",
    "TOKEN_MINT_TWO",
    "TOKEN_MINT_THREE"
  ]
});
```

This is useful for:

* Watchlists
* Portfolio views
* Curated token lists
* Monitoring systems

## 🌡️ Find the Tokens Closest to Bonding

Now we can do something interesting.

Sort by bonding progress.

```ts
const tokens = await moonz.getTokens({
  phase: "BONDING",
  sort: "BONDING_DESC"
});
```

The tokens with the highest bonding progress appear first.

Add a limit:

```ts
const tokens = await moonz.getTokens({
  phase: "BONDING",
  sort: "BONDING_DESC",
  limit: 20
});
```

You have just built the data layer for:

**Top 20 Moonz tokens closest to completing bonding.**

That is a real product idea from a few lines of code.

{% hint style="success" %}
**Your telescope just became a scanner.**

Filter by `BONDING`.

Sort by `BONDING_DESC`.

Now you can watch the front of the pack.
{% endhint %}

## 🧭 Sorting the Moonz

The SDK currently supports:

```text
LAST_TRADE_DESC
LAST_TRADE_ASC

BONDING_DESC
BONDING_ASC

MINT_ASC
MINT_DESC
```

### Most Recently Traded

```ts
const tokens = await moonz.getTokens({
  sort: "LAST_TRADE_DESC"
});
```

### Least Recently Traded

```ts
const tokens = await moonz.getTokens({
  sort: "LAST_TRADE_ASC"
});
```

### Highest Bonding Progress

```ts
const tokens = await moonz.getTokens({
  sort: "BONDING_DESC"
});
```

### Lowest Bonding Progress

```ts
const tokens = await moonz.getTokens({
  sort: "BONDING_ASC"
});
```

### Mint Order

```ts
const tokens = await moonz.getTokens({
  sort: "MINT_ASC"
});
```

or:

```ts
const tokens = await moonz.getTokens({
  sort: "MINT_DESC"
});
```

## 📚 Offset and Limit

Discovery results can be sliced using `offset` and `limit`.

For example:

```ts
const tokens = await moonz.getTokens({
  offset: 0,
  limit: 25
});
```

Then the next group:

```ts
const tokens = await moonz.getTokens({
  offset: 25,
  limit: 25
});
```

The SDK applies filtering and sorting first.

Then it applies the offset.

Then it applies the limit.

If `limit` is omitted, every matching result after the offset is returned.

## ⚠️ A Note About RPC Work

There is an important difference between:

**How many results your application receives**

and:

**How many Launch State accounts the RPC initially returns**

The current SDK asks Solana for Moonz Launch State program accounts first.

Filters such as phase, quote asset, creator and mint are then applied by the SDK.

Sorting happens after filtering.

`offset` and `limit` are applied last.

That means:

```ts
limit: 10
```

limits the final result returned to your application.

It does not tell Solana to fetch only ten Moonz Launch State accounts.

{% hint style="warning" %}
**Building at larger scale?**

Be aware of your RPC provider limits and `getProgramAccounts` capabilities.

For ordinary applications this discovery method is extremely convenient.

For large scale indexing infrastructure, we will later look at realtime tooling designed for continuous protocol ingestion.
{% endhint %}

## 🛡️ Discovery Has Integrity Checks Too

Each `MoonzTokenSummary` includes PDA integrity information.

You can read:

```ts
token.integrity
```

The current discovery integrity checks include:

* Launch State PDA
* Sale Vault PDA
* LP Vault PDA
* Treasury WSOL PDA
* Treasury USDC PDA
* Escrow SOL PDA

And the combined result:

```ts
token.integrity.allPdas
```

That gives a discovery interface a quick way to understand whether the deterministic Moonz protocol addresses line up.

## 🔎 Print a Useful Discovery Result

Here is a compact example.

```ts
const tokens = await moonz.getTokens({
  phase: [
    "BONDING",
    "AMM_LIVE"
  ],

  limit: 50
});

for (const token of tokens) {
  console.log({
    mint:
      token.mint,

    creator:
      token.creator,

    phase:
      token.phase,

    quoteAsset:
      token.quoteAsset,

    bondingProgress:
      token.bondingProgress,

    pdaIntegrity:
      token.integrity.allPdas
  });
}
```

This is based on the public Moonz SDK discovery example.

You now have enough information to populate a useful market list.

## 🧪 Build a Bonding Watchlist

Let us make the idea more focused.

```ts
const tokens = await moonz.getTokens({
  phase: "BONDING",
  sort: "BONDING_DESC",
  limit: 10
});

for (const token of tokens) {
  console.log({
    mint: token.mint,
    creator: token.creator,
    progress: token.bondingProgress,
    quoteAsset: token.quoteAsset,
    lastTrade: token.lastTradeTimestamp
  });
}
```

You have now asked Moonz:

**Show me the ten bonding tokens closest to completion.**

That could become:

* A bonding leaderboard
* A discovery page
* A Telegram alert source
* A trading terminal
* A watchlist
* A market scanner

The SDK gives you the data.

The interface is yours.

## 🌌 One More Step

Discovery gives you summaries.

When somebody selects one of those tokens, you can move from the lightweight discovery view to the complete token view.

```ts
const selected = tokens[0];

const token = await moonz.getToken(
  selected.mint
);
```

That creates a useful application pattern:

```text
Discover many tokens
        ↓
Show summaries
        ↓
User selects one
        ↓
Load full token information
```

Simple.

Efficient.

And completely driven by Moonz state on Solana.

{% hint style="success" %}
**Mission complete.**

You no longer need somebody to hand you a Moonz mint.

Your application can go looking for the Moonz itself.
{% endhint %}

## 📊 Next Stop

We can find the markets.

Now let us understand them.

Next stop:

**Market Data**
