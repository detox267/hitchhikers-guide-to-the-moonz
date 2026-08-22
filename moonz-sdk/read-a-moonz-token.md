# 🔎 Read a Moonz Token

You know the mint.

Now let us find out what is actually behind it.

The easiest way to read one Moonz token is:

```ts
const token = await moonz.getToken(mint);
```

That single call gives your application a Moonz aware view of the token.

Creator.

Lifecycle.

Quote asset.

Metadata.

Vaults.

Reserves.

Supply.

Bonding progress.

Switch state.

Integrity.

A lot is hiding behind one little line of code.

## 🌙 Start With a Mint

We will begin with the same Moonz client from the quick start.

```ts
import { MoonzSDK } from "@moonz-fun/sdk";

const moonz = new MoonzSDK({
  rpcUrl: "YOUR_SOLANA_RPC"
});

const mint = "TOKEN_MINT";

const token = await moonz.getToken(mint);
```

If the mint cannot be recognised as a valid Moonz token, the result is:

```text
null
```

So always check the result before using it.

```ts
if (!token) {
  console.log("Not a Moonz token");
  process.exit(0);
}
```

From here on, we have Moonz.

## 🪪 Identity

Let us begin with the obvious stuff.

```ts
console.log("Mint:", token.mint);
console.log("Creator:", token.creator);
console.log("Program:", token.programId);
```

`mint` is the token mint you are reading.

`creator` is the creator recorded by the Moonz Launch State.

`programId` identifies the Moonz program the SDK is reading.

Simple.

Now things get more interesting.

## 🧬 Where Is This Token?

Every Moonz token has lifecycle state.

Read it with:

```ts
console.log("Phase:", token.phase);
```

The current public SDK understands:

```text
PENDING_DEV_BUY
BONDING
AMM_LIVE
SWITCHING
CANCELLED
UNKNOWN
```

For most applications, the readable `phase` value is what you want.

The numeric value is also available:

```ts
console.log(
  "Phase Code:",
  token.phaseCode
);
```

This means your application does not have to treat every Moonz token as though it is in exactly the same state.

It can react to where that token currently is.

## 💱 What Is It Quoted Against?

Read the active quote asset with:

```ts
console.log(
  "Quote Asset:",
  token.quoteAsset
);
```

The current recognised values are:

```text
SOL
USDC
UNKNOWN
```

The numeric code is also available:

```ts
console.log(
  "Quote Asset Code:",
  token.quoteAssetCode
);
```

This becomes important for market displays, trading integrations and PCLS aware applications.

## 🖼️ Give the Token a Name

Blockchain addresses are useful.

Humans generally prefer names.

Moonz exposes decoded token metadata at:

```ts
token.metadata
```

Because metadata may be unavailable, use optional access when reading it.

```ts
console.log(
  "Name:",
  token.metadata?.name
);

console.log(
  "Symbol:",
  token.metadata?.symbol
);

console.log(
  "Metadata URI:",
  token.metadata?.uri
);
```

The SDK also exposes metadata verification information.

```ts
console.log(
  "Valid Owner:",
  token.metadata?.validOwner
);

console.log(
  "Valid Mint:",
  token.metadata?.validMint
);

console.log(
  "Matches Launch State:",
  token.metadata?.matchesLaunchState
);

console.log(
  "Matches Derived PDA:",
  token.metadata?.matchesDerivedPda
);
```

That gives explorers and analytics platforms extra context about the metadata they are displaying.

## 🌡️ How Far Through Bonding?

This one is useful immediately.

```ts
console.log(
  "Bonding Progress:",
  token.supply.bondingProgress
);
```

You also have the underlying supply information.

```ts
console.log(
  "Sale Supply:",
  token.supply.saleRaw
);

console.log(
  "Tokens Sold:",
  token.supply.soldRaw
);

console.log(
  "Tokens Remaining:",
  token.supply.remainingRaw
);
```

The complete token supply is represented as a `MoonzAmount`.

```ts
console.log(
  token.supply.total
);
```

A `MoonzAmount` contains:

```ts
{
  raw,
  decimals,
  ui
}
```

That means applications can keep exact blockchain values for calculations while still having a human readable representation available for display.

For example:

```ts
console.log(
  "Total Supply:",
  token.supply.total.ui
);
```

## 🏦 Where Are the Protocol Accounts?

Moonz uses deterministic protocol accounts throughout the token lifecycle.

The SDK exposes two views.

```ts
token.vaults.stored
token.vaults.derived
```

`stored` contains the addresses recorded by the Moonz Launch State.

`derived` contains the addresses the SDK deterministically derives for the supplied mint.

You can inspect them directly.

```ts
console.log(
  "Stored Sale Vault:",
  token.vaults.stored.saleVault
);

console.log(
  "Derived Sale Vault:",
  token.vaults.derived.saleVault
);
```

The same structure covers:

* Launch State
* Sale Vault
* LP Vault
* Treasury WSOL Vault
* Treasury USDC Vault
* Escrow SOL Vault

This is especially useful when you are building an explorer or verifying that an account really belongs where it claims to.

## 💰 What Is Sitting in the Vaults?

Addresses tell you where the accounts are.

Reserves tell you what is in them.

```ts
const reserves = token.reserves;
```

The current SDK exposes:

```text
saleTokens
lpTokens
wsol
usdc
escrowSol
```

Token account reserves may be unavailable, so handle them safely.

```ts
console.log(
  "Sale Tokens:",
  reserves.saleTokens?.amount.ui
);

console.log(
  "LP Tokens:",
  reserves.lpTokens?.amount.ui
);

console.log(
  "WSOL:",
  reserves.wsol?.amount.ui
);

console.log(
  "USDC:",
  reserves.usdc?.amount.ui
);

console.log(
  "Escrow SOL:",
  reserves.escrowSol.ui
);
```

Now your application is not only aware of protocol addresses.

It can understand their balances too.

## 🔄 Is the Market Switching?

Moonz exposes a dedicated switch state.

```ts
console.log(
  "Switch Active:",
  token.switching.active
);

console.log(
  "Current Quote:",
  token.switching.currentQuoteAsset
);

console.log(
  "Pending Quote:",
  token.switching.pendingQuoteAsset
);
```

More detailed information is also available.

```ts
console.log(
  "Started At:",
  token.switching.startedAt
);

console.log(
  "Last Completed:",
  token.switching.lastCompletedAt
);

console.log(
  "Swap Executed:",
  token.switching.swapExecuted
);
```

You do not need to understand the full PCLS process yet.

For now, just know that your application can see when Moonz is moving through that state.

We will return to this later.

## 🛡️ Check the Integrity

Reading data is useful.

Knowing whether the important protocol relationships line up is even better.

Moonz exposes:

```ts
token.integrity
```

The individual checks include:

```text
programOwner

launchStatePda

saleVaultPda
lpVaultPda

treasuryWsolPda
treasuryUsdcPda

escrowSolPda

tokenProgramOwners
vaultAuthorities
vaultMints
```

And then there is the combined result:

```ts
console.log(
  "Integrity:",
  token.integrity.all
);
```

When every integrity check represented by the SDK passes, `all` is `true`.

{% hint style="info" %}
**Integrity is useful context, not decoration.**

A third party application can use these checks to verify important relationships between the token and the Moonz protocol instead of blindly displaying whatever account addresses it encounters.
{% endhint %}

## 🛰️ Go Straight to Launch State

`getToken()` gives most applications the easier combined view.

Some integrations will want the decoded Launch State itself.

```ts
const state = await moonz.getLaunchState(
  mint
);
```

The Launch State contains deeper protocol information including:

* Creator
* Current lifecycle phase
* Metadata state
* Sale Vault
* LP Vault
* Treasury vaults
* Escrow vault
* Sale supply
* Tokens sold
* SOL collected
* Current quote asset
* Pending quote asset
* Switch information
* Last trade timestamp

We will explore the Moonz lifecycle and Launch State in much more detail later.

## 🧪 Build a Tiny Token Profile

Let us turn everything we have learned into something useful.

```ts
import { MoonzSDK } from "@moonz-fun/sdk";

const moonz = new MoonzSDK({
  rpcUrl: "YOUR_SOLANA_RPC"
});

const mint = "TOKEN_MINT";

const token = await moonz.getToken(mint);

if (!token) {
  console.log("Not a Moonz token");
  process.exit(0);
}

const profile = {
  mint: token.mint,

  name:
    token.metadata?.name ?? null,

  symbol:
    token.metadata?.symbol ?? null,

  creator:
    token.creator,

  phase:
    token.phase,

  quoteAsset:
    token.quoteAsset,

  bondingProgress:
    token.supply.bondingProgress,

  integrity:
    token.integrity.all
};

console.log(profile);
```

You have just turned one mint address into a small Moonz token profile.

That same idea can become:

* A token card
* A scanner result
* A wallet integration
* A trading bot response
* An explorer page
* A portfolio entry
* An analytics record

One mint.

One call.

A lot more context.

{% hint style="success" %}
**Mission complete.**

Your application can now take a Moonz mint and understand who created it, where it is in the protocol, what market it uses, how far it has travelled through bonding and whether its important protocol relationships pass the SDK integrity checks.
{% endhint %}

## 🔭 Next Stop

Reading one Moonz token is useful.

Finding Moonz tokens you did not already know about is where things really start moving.

Next stop:

**Discover Moonz**
