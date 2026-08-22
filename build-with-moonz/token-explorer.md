# 🔭 Build a Token Explorer

Time to build something.

A Moonz token explorer can read a token directly from Solana and turn the protocol state into information a user can understand.

You can show:

```text
Token name

Symbol

Mint

Creator

Lifecycle phase

Quote asset

Bonding progress

Current price

Market cap

Sale reserve

LP reserve

Quote reserve

PCLS state

Protocol integrity
```

All of this can be built from the public Moonz SDK.

{% hint style="success" %}
A basic Moonz explorer does not require the Moonz website database or private indexer infrastructure.

The public SDK reads canonical protocol state from Solana.
{% endhint %}

## 📦 Install the SDK

```bash
npm install @moonz-fun/sdk
```

The SDK supports TypeScript and JavaScript.

You also need access to a Solana RPC endpoint.

## 🌙 Create the Client

```ts
import {
  MoonzSDK
} from "@moonz-fun/sdk";

const moonz =
  new MoonzSDK({
    rpcUrl:
      process.env.SOLANA_RPC_URL
  });
```

If your RPC provider uses a separate WebSocket endpoint:

```ts
const moonz =
  new MoonzSDK({
    rpcUrl:
      process.env.SOLANA_RPC_URL,

    wsEndpoint:
      process.env.SOLANA_WSS_URL
  });
```

The default commitment is:

```text
confirmed
```

## 🔎 Start With a Mint

Your explorer can accept a Solana mint address from:

```text
Search input

URL parameter

QR code

Wallet activity

External integration
```

For example:

```ts
const mint =
  "TOKEN_MINT";
```

## 🛡️ Check Whether It Is Moonz

Before building a Moonz specific interface:

```ts
const isMoonz =
  await moonz.isMoonzToken(
    mint
  );

if (!isMoonz) {
  throw new Error(
    "Not a Moonz token"
  );
}
```

`isMoonzToken()` checks whether the canonical Moonz Launch State can be resolved for the mint.

This is much safer than identifying a Moonz token from its name, symbol or image.

## 🧬 Read the Complete Token

Now load the token:

```ts
const token =
  await moonz.getToken(
    mint
  );

if (!token) {
  throw new Error(
    "Moonz token not found"
  );
}
```

`getToken()` returns a normalized:

```text
MoonzTokenInfo
```

object.

## 🪪 Identity

The top level identity fields are:

```ts
console.log(
  token.mint
);

console.log(
  token.creator
);

console.log(
  token.programId
);
```

These give your explorer the canonical token mint, launch creator and Moonz Program ID.

## 🖼️ Metadata

If metadata is available:

```ts
console.log(
  token.metadata?.name
);

console.log(
  token.metadata?.symbol
);

console.log(
  token.metadata?.uri
);
```

Useful metadata fields include:

```text
name

symbol

uri

metadata address

update authority

mutable

validOwner

validMint

matchesLaunchState

matchesDerivedPda
```

The `uri` is the metadata URI recorded in the Metaplex metadata account.

If your application needs the image, description or other JSON properties, fetch the metadata document referenced by that URI.

## 🌗 Lifecycle Phase

Display:

```ts
token.phase
```

Possible normalized SDK values include:

```text
PENDING_DEV_BUY

BONDING

AMM_LIVE

SWITCHING

CANCELLED

UNKNOWN
```

For most interfaces, the text label is more useful than the numeric phase code.

The raw normalized code is also available:

```ts
token.phaseCode
```

## 💱 Quote Asset

The current quote asset is:

```ts
token.quoteAsset
```

Normally:

```text
SOL

or

USDC
```

You should use this field when labelling price and liquidity.

Do not permanently label every Moonz token as SOL quoted because an AMM can move between SOL and USDC through PCLS.

## 📈 Bonding Progress

The SDK exposes:

```ts
token.supply.bondingProgress
```

This is a percentage value derived from the sale allocation and tokens sold.

For example:

```ts
const progress =
  token.supply
    .bondingProgress;

console.log(
  `${progress}%`
);
```

You can use that value for a progress bar.

## 🪙 Supply

The current SPL token supply is available as:

```ts
token.supply.total
```

Moonz amounts have three useful representations:

```text
raw

decimals

ui
```

For display:

```ts
console.log(
  token.supply.total.ui
);
```

For exact machine accounting:

```ts
console.log(
  token.supply.total.raw
);
```

## 🌒 Bonding Inventory

The SDK also exposes:

```ts
token.supply.saleRaw

token.supply.soldRaw

token.supply.remainingRaw
```

These are raw token units.

For a human interface, the actual sale vault reserve already includes a formatted amount:

```ts
const saleReserve =
  token.reserves
    .saleTokens
    ?.amount.ui;
```

## 🌊 LP Reserve

The current Moonz token reserve in the LP vault is:

```ts
const lpReserve =
  token.reserves
    .lpTokens
    ?.amount.ui;
```

This becomes particularly important once the token reaches:

```text
AMM_LIVE
```

## ☀️ WSOL Reserve

```ts
const wsolReserve =
  token.reserves
    .wsol
    ?.amount.ui;
```

The SDK formats WSOL using 9 decimals.

## 💵 USDC Reserve

```ts
const usdcReserve =
  token.reserves
    .usdc
    ?.amount.ui;
```

The SDK formats USDC using 6 decimals.

## 🧭 Select the Active Quote Reserve

A simple explorer can choose the display reserve from the current quote asset:

```ts
const quoteReserve =
  token.quoteAsset === "SOL"
    ? token.reserves.wsol
        ?.amount.ui
    : token.quoteAsset === "USDC"
      ? token.reserves.usdc
          ?.amount.ui
      : null;
```

This keeps the interface aligned with the current protocol state.

## 📊 Get Current Market Data

Token structure and market pricing are deliberately separate concepts.

Load the canonical market view with:

```ts
const market =
  await moonz.getMarketData(
    mint
  );

if (!market) {
  throw new Error(
    "Market data unavailable"
  );
}
```

The returned object includes:

```text
phase

market

priceSource

tradable

quoteAsset

priceQuote

marketCapQuote

totalSupply

bondingProgress

virtual reserves

AMM reserves

integrityAll
```

## 💰 Current Price

```ts
const price =
  market.priceQuote;
```

`priceQuote` is the spot price of one whole Moonz token denominated in the active quote asset.

So if:

```text
quoteAsset = SOL
```

the price is in SOL.

If:

```text
quoteAsset = USDC
```

the price is in USDC.

## 🌌 Market Cap

```ts
const marketCap =
  market.marketCapQuote;
```

This is:

```text
Current whole token price

×

Current total SPL supply
```

and is denominated in the active quote asset.

{% hint style="info" %}
The Moonz SDK does not fetch an external fiat price feed.

For a SOL quoted market, `marketCapQuote` is in SOL.

For a USDC quoted market, it is in USDC.
{% endhint %}

## 🧮 Price Source

Your explorer can show where the current price came from:

```ts
market.priceSource
```

Possible values are:

```text
VIRTUAL_CURVE

AMM_RESERVES

UNAVAILABLE
```

During bonding:

```text
VIRTUAL_CURVE
```

means the SDK used the Moonz bonding reserve model.

During the live AMM:

```text
AMM_RESERVES
```

means the SDK used the actual protocol vault balances.

## 🛣️ Market Type

```ts
market.market
```

can be:

```text
BONDING

AMM

UNAVAILABLE
```

This is different from lifecycle phase.

For example:

```text
phase
=
SWITCHING

market
=
UNAVAILABLE
```

because PCLS is a lifecycle state but is not a trading market.

## 🚦 Tradable

Use:

```ts
market.tradable
```

when deciding whether your interface should present a live trading state.

Do not guess tradability from the token name or from stale cached information.

## 🔀 Show PCLS State

The token object exposes:

```ts
token.switching
```

Useful fields are:

```text
active

currentQuoteAsset

pendingQuoteAsset

startedAt

lastCompletedAt

feeEscrowedLamports

amountInRaw

minAmountOutRaw

swapExecuted
```

For example:

```ts
if (
  token.switching.active
) {
  console.log(
    "Switching from",
    token.switching
      .currentQuoteAsset,
    "to",
    token.switching
      .pendingQuoteAsset
  );
}
```

An explorer can turn that into a useful status such as:

```text
Pool switch in progress

SOL → USDC
```

## 🛡️ Display Protocol Integrity

One of the strongest features you can add to a Moonz explorer is protocol verification.

The SDK exposes:

```ts
token.integrity
```

with checks for:

```text
Program owner

Launch State PDA

Sale vault PDA

LP vault PDA

WSOL treasury PDA

USDC treasury PDA

Escrow SOL PDA

Token Program owners

Vault authorities

Vault mints
```

The combined result is:

```ts
token.integrity.all
```

## ✅ A Simple Verified Badge

```ts
const protocolStatus =
  token.integrity.all
    ? "Verified Moonz state"
    : "Integrity check failed";
```

{% hint style="warning" %}
Do not hide failed integrity checks.

If `token.integrity.all` is false, surface that state clearly and avoid presenting derived market information as trusted Moonz protocol data.
{% endhint %}

## 🧩 Build One Explorer Object

Here is a useful application layer model:

```ts
async function loadExplorerToken(
  mint: string
) {
  const token =
    await moonz.getToken(
      mint
    );

  if (!token) {
    return null;
  }

  const market =
    await moonz.getMarketData(
      mint
    );

  if (!market) {
    return null;
  }

  const quoteReserve =
    token.quoteAsset === "SOL"
      ? token.reserves.wsol
          ?.amount.ui ?? null
      : token.quoteAsset === "USDC"
        ? token.reserves.usdc
            ?.amount.ui ?? null
        : null;

  return {
    mint:
      token.mint,

    creator:
      token.creator,

    name:
      token.metadata?.name ??
      "Unknown",

    symbol:
      token.metadata?.symbol ??
      "",

    metadataUri:
      token.metadata?.uri ??
      null,

    phase:
      token.phase,

    quoteAsset:
      token.quoteAsset,

    tradable:
      market.tradable,

    market:
      market.market,

    priceSource:
      market.priceSource,

    price:
      market.priceQuote,

    marketCap:
      market.marketCapQuote,

    bondingProgress:
      market.bondingProgress,

    totalSupply:
      market.totalSupply.ui,

    saleReserve:
      token.reserves
        .saleTokens
        ?.amount.ui ??
      null,

    lpReserve:
      token.reserves
        .lpTokens
        ?.amount.ui ??
      null,

    quoteReserve,

    switching:
      token.switching,

    integrity:
      token.integrity.all
  };
}
```

Now the rest of your application does not need to understand raw Solana account layouts.

It can consume one clean explorer object.

## 🖥️ A Simple Token Header

Your interface could render:

```text
MOONZ TOKEN NAME
SYMBOL

Mint
Creator

Phase
BONDING

Quote
SOL

Price
0.000000123 SOL

Market Cap
123 SOL

Bonding
72.48%

Protocol
✓ Verified
```

When the market changes, the same layout can adapt automatically.

## 🌊 After Migration

Once the token becomes:

```text
AMM_LIVE
```

your explorer can change the market section from:

```text
Bonding progress

Sale reserve
```

to:

```text
LP reserve

Quote reserve

AMM price
```

without changing how the token itself is identified.

## 🔀 During PCLS

If:

```ts
token.phase ===
  "SWITCHING"
```

show the lifecycle state instead of pretending there is a normal current market price.

For example:

```text
POOL SWITCH IN PROGRESS

Current quote
SOL

Pending quote
USDC

Trading
Temporarily unavailable
```

`getMarketData()` reports the market as unavailable during non tradable lifecycle states.

## 🧠 Handle Null Values Properly

Several fields can legitimately be null.

Examples include:

```text
Metadata not available

Price unavailable

Market cap unavailable

AMM reserve not applicable

Bonding virtual reserve not applicable
```

Null does not automatically mean the SDK failed.

It can mean the value does not apply to the token's current lifecycle state.

## 🧱 Keep Raw Amounts for Machines

For user display, use:

```ts
amount.ui
```

For exact accounting, indexing or calculations, keep:

```ts
amount.raw
```

The SDK represents raw protocol amounts as strings so large integer values are not silently damaged by JavaScript number precision.

## 🔐 Keep RPC Credentials Server Side When Needed

If your RPC URL contains a private API key, do not expose that credential in public browser code.

A frontend can use:

```text
A browser safe RPC endpoint
```

or send read requests through infrastructure you control.

The Moonz SDK itself does not require a Moonz API key.

## 🚀 What You Have Built

With two core calls:

```ts
moonz.getToken(
  mint
);

moonz.getMarketData(
  mint
);
```

you have enough public protocol data to build a useful Moonz token page.

You can display:

```text
Identity

Metadata

Creator

Lifecycle

Bonding progress

Price

Market cap

Liquidity

PCLS state

Integrity
```

without depending on Moonz private infrastructure.

{% hint style="success" %}
A Moonz explorer can verify what a token is, where it is in the lifecycle, how its market is currently priced and whether its protocol accounts pass the SDK integrity checks.
{% endhint %}

## 📡 Next Build

A token explorer gives you the current state.

Next we make it move.

Next build:

**Live Trade Feed**
