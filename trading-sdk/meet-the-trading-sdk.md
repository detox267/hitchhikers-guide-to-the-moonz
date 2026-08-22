# ⚡ Meet the Trading SDK

Reading Moonz is useful.

Eventually somebody is going to press the button.

The Moonz Trading SDK is the public TypeScript and JavaScript interface for quoting, building and executing Moonz trades directly against Solana.

Install it with:

```bash
npm install @moonz-fun/trading-sdk
```

The current public package version is:

```text
0.1.1
```

{% hint style="info" %}
The Trading SDK handles the trading path.

For token information, metadata, reserves, protocol addresses and events, use `@moonz-fun/sdk`.
{% endhint %}

## 🌙 Two SDKs, Different Jobs

Moonz exposes two complementary developer SDKs.

```text
@moonz-fun/sdk

Read Moonz
Inspect state
Read reserves
Read metadata
Decode events
Verify protocol relationships

        +

@moonz-fun/trading-sdk

Quote trades
Build transactions
Buy
Sell
Create tokens
```

The Trading SDK actually uses the information SDK internally.

Every `MoonzTradingSDK` instance exposes:

```ts
moonz.info
```

which is a `MoonzSDK` instance using the same Solana connection and commitment.

## 🚀 Create a Client

The simplest setup is:

```ts
import {
  MoonzTradingSDK
} from "@moonz-fun/trading-sdk";

const moonz =
  new MoonzTradingSDK({
    rpcUrl: "YOUR_SOLANA_RPC"
  });
```

That is enough to start quoting Moonz trades.

## 🛰️ Bring Your Own Connection

If your application already has a Solana `Connection`, pass it directly.

```ts
import {
  Connection
} from "@solana/web3.js";

import {
  MoonzTradingSDK
} from "@moonz-fun/trading-sdk";

const connection =
  new Connection(
    "YOUR_SOLANA_RPC",
    "confirmed"
  );

const moonz =
  new MoonzTradingSDK({
    connection
  });
```

You must provide either:

```text
rpcUrl

or

connection
```

If neither is supplied, construction fails.

## ✅ Commitment

The default commitment is:

```text
confirmed
```

You can provide another Solana commitment through the constructor:

```ts
const moonz =
  new MoonzTradingSDK({
    rpcUrl: "YOUR_SOLANA_RPC",
    commitment: "confirmed"
  });
```

The selected commitment is shared with the internal Moonz information SDK.

## 🧰 Constructor Options

The public constructor accepts:

```ts
interface MoonzTradingSDKOptions {
  rpcUrl?: string;
  connection?: Connection;
  commitment?: Commitment;
  apiUrl?: string;
}
```

For ordinary trading, the important fields are:

```text
rpcUrl

connection

commitment
```

The optional `apiUrl` belongs to flows that use the Moonz API, such as token creation.

Normal trading quote and transaction methods read directly from Solana.

## 🧭 The Public Trading Surface

The public trading methods are:

```text
quoteBuy()

quoteSell()

buildBuyTransaction()

buildSellTransaction()

buy()

sell()
```

There is also:

```text
createToken()
```

which is a separate public creation flow that we will cover independently.

## 💬 Quotes

Use:

```ts
await moonz.quoteBuy(...)
```

or:

```ts
await moonz.quoteSell(...)
```

when you want to calculate the trade without constructing or sending a transaction.

These methods read current Solana state.

No Moonz API call is made.

## 🧱 Transaction Builders

Use:

```ts
await moonz.buildBuyTransaction(...)
```

or:

```ts
await moonz.buildSellTransaction(...)
```

when your application wants control over the signing and submission flow.

The builder returns an unsigned Solana `Transaction`.

Nothing is signed.

Nothing is submitted.

## 👛 High Level Trading

For the shortest application flow, use:

```ts
await moonz.buy(...)
```

and:

```ts
await moonz.sell(...)
```

The high level methods perform the full sequence:

```text
Read current Moonz state
        ↓
Quote the trade
        ↓
Choose the correct Moonz route
        ↓
Build the transaction
        ↓
Ask the wallet to sign
        ↓
Submit to Solana
        ↓
Confirm
        ↓
Return the result
```

## 🔐 The Wallet Signs Locally

The Trading SDK does not accept a private key.

Its high level wallet interface is deliberately small:

```ts
interface MoonzWalletSigner {
  publicKey:
    | PublicKey
    | null;

  signTransaction?: (
    transaction: Transaction
  ) => Promise<Transaction>;
}
```

That is compatible with the standard pattern used by Solana wallet adapters.

The wallet signs the transaction locally.

Moonz never receives the private key.

{% hint style="success" %}
Your integration provides a wallet capable of signing a Solana transaction.

It does not hand wallet custody to Moonz.
{% endhint %}

## 🛣️ Routing Is Automatic

This is one of the most important Trading SDK features.

Your application does not need to decide whether a token is still bonding, trading in a SOL AMM or trading in a USDC AMM.

For buys:

```text
BONDING
    ↓
buy

AMM_LIVE + SOL
    ↓
buy

AMM_LIVE + USDC
    ↓
amm_buy_usdc
```

For sells:

```text
BONDING
    ↓
sell

AMM_LIVE + SOL
    ↓
sell

AMM_LIVE + USDC
    ↓
amm_sell_usdc
```

The SDK reads the current Moonz state and chooses the correct route.

## 🌱 Bonding

While a token is in:

```text
BONDING
```

its trading quote asset is SOL.

A buy can be as simple as:

```ts
const quote =
  await moonz.quoteBuy({
    mint: "TOKEN_MINT",
    amount: "0.1"
  });
```

Here:

```text
0.1
```

means:

```text
0.1 SOL
```

The same public `quoteBuy()` method will later work when that token is in the AMM.

## 🚀 AMM Live

Once the token is:

```text
AMM_LIVE
```

the Trading SDK reads the active quote asset from current Moonz state.

If it is SOL:

```text
amount = SOL
```

If it is USDC:

```text
amount = USDC
```

Your interface can therefore call the same public method while the SDK selects the correct market path underneath.

## 💱 SOL and USDC

A buy amount represents the current quote asset.

For example:

```ts
await moonz.quoteBuy({
  mint: "TOKEN_MINT",
  amount: "0.25"
});
```

For a SOL market that means:

```text
0.25 SOL
```

For a USDC market it means:

```text
0.25 USDC
```

The returned quote tells you which asset was used through:

```ts
quote.quoteAsset
```

which is:

```text
SOL

or

USDC
```

Do not assume every Moonz market is permanently quoted in SOL.

## 🔴 Sell Amounts Are Moonz Tokens

Sell inputs work differently.

For:

```ts
await moonz.quoteSell({
  mint: "TOKEN_MINT",
  amount: "100000"
});
```

the amount is the human readable Moonz token quantity being sold.

Moonz launch tokens use:

```text
6 decimals
```

The resulting quote tells you how much of the active quote asset is expected back.

## 🛡️ Slippage

Trading methods accept:

```ts
slippageBps
```

The default is:

```text
100 basis points
```

which is:

```text
1%
```

For example:

```ts
const quote =
  await moonz.quoteBuy({
    mint: "TOKEN_MINT",
    amount: "0.1",
    slippageBps: 100
  });
```

The quote does not merely report an expected output.

It also calculates the minimum output that should be accepted by the transaction.

## 💬 Quote Before You Trade

A strong user interface normally quotes first.

```ts
const quote =
  await moonz.quoteBuy({
    mint,
    amount: "0.1",
    slippageBps: 100
  });

console.log(
  quote.expectedTokensOut
);

console.log(
  quote.minTokensOut
);
```

Then display the important information to the user before requesting a wallet signature.

## 🧬 Buy Quote Information

A `MoonzBuyQuote` includes:

```text
mint

side

phase

market

quoteAsset

quoteDecimals

tokenDecimals

amountIn

expectedTokensOut

minTokensOut

tradeFeeRaw

creatorFeeRaw

platformFeeRaw

lpFeeRaw

slippageBps

crossesCurve

migratesToAmm
```

Raw integer versions are also provided for the important token and quote amounts.

## 🌉 One Buy Can Cross the Curve

Moonz exposes a special market value:

```text
BONDING_TO_AMM
```

A buy can consume the remaining bonding allocation and continue into the newly created AMM within the same trade path.

The buy quote exposes:

```ts
quote.crossesCurve
```

to tell you whether the buy crosses that boundary.

It also exposes:

```ts
quote.migratesToAmm
```

when the trade finishes the bonding allocation and causes migration into `AMM_LIVE`.

This is useful information for trading interfaces.

## 🔴 Sell Quote Information

A `MoonzSellQuote` includes:

```text
mint

side

phase

market

quoteAsset

quoteDecimals

tokenDecimals

tokensIn

grossQuoteOut

expectedQuoteOut

minQuoteOut

tradeFeeRaw

creatorFeeRaw

platformFeeRaw

lpFeeRaw

slippageBps
```

Again, raw integer versions are available alongside the human readable values.

## 🔢 Keep Raw Values Raw

The quote structures expose raw amounts as strings.

For example:

```ts
quote.expectedTokensOutRaw

quote.minTokensOutRaw

quote.tradeFeeRaw
```

That is intentional.

Blockchain integer amounts can exceed the safe precision of ordinary JavaScript numbers.

If your application performs protocol accounting, preserve the raw integer strings or convert them to `bigint`.

Use the human readable fields for display.

## 💸 Trading Fee

The current protocol trading fee is:

```text
125 basis points
```

or:

```text
1.25%
```

The public quote objects expose the calculated fee components.

The split depends on the active Moonz market.

We will examine this properly in the quote page rather than asking applications to recreate the internal trading math.

## 🧱 Build Without Sending

Suppose your application uses a custom transaction pipeline.

You can build a buy:

```ts
const built =
  await moonz.buildBuyTransaction({
    mint: "TOKEN_MINT",
    buyer: wallet.publicKey,
    amount: "0.1",
    slippageBps: 100
  });

const transaction =
  built.transaction;
```

The result also contains:

```text
quote

instruction

blockhash

lastValidBlockHeight

resolved account addresses
```

Nothing has been signed or submitted yet.

## 🛰️ See Which Instruction Was Selected

A built buy tells you which Moonz instruction was selected:

```ts
built.instruction
```

The value is:

```text
buy

or

amm_buy_usdc
```

A built sell returns:

```text
sell

or

amm_sell_usdc
```

This is useful for debugging and transaction inspection.

Your application still does not need to choose the route itself.

## 🟢 High Level Buy

The simplest complete buy is:

```ts
const result =
  await moonz.buy({
    mint: "TOKEN_MINT",
    wallet,
    amount: "0.1",
    slippageBps: 100
  });

console.log(
  result.signature
);
```

The SDK:

```text
Quotes

Builds

Requests wallet signature

Submits

Confirms
```

and returns the confirmed transaction information.

## 🔴 High Level Sell

Selling follows the same model:

```ts
const result =
  await moonz.sell({
    mint: "TOKEN_MINT",
    wallet,
    amount: "100000",
    slippageBps: 100
  });

console.log(
  result.signature
);
```

The amount is the Moonz token quantity to sell.

The output asset is determined by the token's current market.

## ✅ Trade Results

Successful high level buy and sell methods return:

```text
signature

confirmationSlot

instruction

quote
```

For example:

```ts
console.log(
  result.signature
);

console.log(
  result.confirmationSlot
);

console.log(
  result.instruction
);

console.log(
  result.quote.quoteAsset
);
```

That gives the application both chain confirmation and the quote used to construct the transaction.

## 🚫 What Is Not Public API

The package contains internal trade math and transaction construction helpers.

Examples include implementation functions for:

```text
curve calculations

AMM calculations

fee splitting

direct SOL builders

direct USDC builders
```

They are not exported by the package root.

Do not build integrations against internal source files simply because they exist in the repository.

The supported public surface comes from:

```ts
@moonz-fun/trading-sdk
```

and its package exports.

{% hint style="warning" %}
If a function is not exported by the package entry point, treat it as implementation detail rather than public SDK contract.
{% endhint %}

## 🧭 The Trading Journey

The rest of this section will follow the path a real integration takes.

```text
Moonz Trading SDK
        ↓
Quote
        ↓
Understand route
        ↓
Apply slippage protection
        ↓
Build
        ↓
Wallet signs
        ↓
Submit
        ↓
Confirm
```

We will keep the route logic inside the SDK where it belongs.

Your application gets to focus on the user.

## 💬 Next Stop

Before we buy anything, we should know what we are about to receive.

Next stop:

**Quote a Trade**
