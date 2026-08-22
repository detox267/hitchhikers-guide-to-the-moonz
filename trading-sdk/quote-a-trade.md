# 💬 Quote a Trade

Before a wallet signs anything, know what the trade is expected to do.

The Moonz Trading SDK exposes two quote methods:

```text
quoteBuy()

quoteSell()
```

Both read current Moonz state directly from Solana.

Neither method signs a transaction.

Neither method submits a transaction.

Neither method requires the Moonz API.

{% hint style="success" %}
**Quote first. Sign second.**

The quote gives your application the expected output, minimum protected output, active market, quote asset and fee information before the wallet is asked to approve anything.
{% endhint %}

## 🟢 Quote a Buy

A basic buy quote looks like:

```ts
const quote =
  await moonz.quoteBuy({
    mint: "TOKEN_MINT",
    amount: "0.1",
    slippageBps: 100
  });
```

The `amount` is the amount of the token's current quote asset you want to spend.

If the market uses SOL:

```text
amount: "0.1"

means

0.1 SOL
```

If the market uses USDC:

```text
amount: "25.50"

means

25.50 USDC
```

The SDK reads the current Moonz state and determines which one applies.

## 🔴 Quote a Sell

A sell quote looks like:

```ts
const quote =
  await moonz.quoteSell({
    mint: "TOKEN_MINT",
    amount: "100000",
    slippageBps: 100
  });
```

For sells, `amount` is the human readable quantity of Moonz tokens being sold.

Moonz launch tokens use:

```text
6 decimals
```

So:

```text
amount: "100000"

means

100000 Moonz tokens
```

It does not mean 100000 raw token units.

## 🛰️ Quotes Read Current Solana State

Internally, the Trading SDK uses the Moonz information SDK to obtain the current token snapshot.

Conceptually:

```text
Token mint
    ↓
Moonz state
    ↓
Current phase
    ↓
Current quote asset
    ↓
Current reserves
    ↓
Trade calculation
    ↓
Quote
```

This matters because Moonz markets can change during their lifecycle.

A quote should reflect the market that exists now.

## 🌱 Bonding Buy

If the token is currently:

```text
BONDING
```

the buy quote uses the Moonz bonding curve.

The quote asset is:

```text
SOL
```

The returned quote reports:

```text
phase
BONDING

market
BONDING

quoteAsset
SOL
```

The same `quoteBuy()` method is used later when that token has reached its AMM.

## 🚀 AMM Buy

If the token is:

```text
AMM_LIVE
```

the SDK reads the active quote asset.

The market may be:

```text
AMM_LIVE + SOL

or

AMM_LIVE + USDC
```

The quote calculation uses the corresponding live AMM reserves.

Your application does not need separate public quote methods for SOL and USDC.

## 🛣️ The Quote Selects the Market

The public API stays simple:

```ts
await moonz.quoteBuy({
  mint,
  amount
});
```

Underneath, the SDK determines:

```text
BONDING
        ↓
Bonding curve quote

AMM_LIVE + SOL
        ↓
SOL AMM quote

AMM_LIVE + USDC
        ↓
USDC AMM quote
```

This routing is based on current Moonz state.

Do not duplicate that route selection in your frontend unless you have a separate product reason to display it.

## 💱 Quote Asset

Every trade quote exposes:

```ts
quote.quoteAsset
```

The possible public values are:

```text
SOL

USDC
```

Your interface should display this value alongside quote amounts.

For example:

```ts
console.log(
  `${quote.amountIn} ${quote.quoteAsset}`
);
```

A numeric value without its quote asset is incomplete market information.

## 🔢 Quote Decimals

The quote also exposes:

```ts
quote.quoteDecimals
```

For SOL:

```text
9
```

For USDC:

```text
6
```

Moonz launch token decimals are exposed as:

```text
6
```

through:

```ts
quote.tokenDecimals
```

## 🌍 Human Values and Raw Values

Moonz quote objects expose both display values and raw blockchain values.

For example, a buy quote contains:

```ts
quote.amountIn
quote.amountInRaw

quote.expectedTokensOut
quote.expectedTokensOutRaw

quote.minTokensOut
quote.minTokensOutRaw
```

This gives applications two useful representations.

```text
Human value
    ↓
Display to users

Raw value
    ↓
Exact protocol amount
```

## 🧠 Keep Raw Integers Precise

Raw values are returned as strings.

That is deliberate.

Do not convert large raw blockchain values into ordinary JavaScript numbers unless you know they are within the safe integer range.

For protocol calculations, prefer:

```ts
const raw =
  BigInt(
    quote.expectedTokensOutRaw
  );
```

For user interfaces, use the already formatted human value.

## 🟢 The Buy Quote

The public `MoonzBuyQuote` contains:

```ts
{
  mint,
  side,
  phase,
  market,
  quoteAsset,
  quoteDecimals,
  tokenDecimals,

  amountInRaw,
  amountIn,

  expectedTokensOutRaw,
  expectedTokensOut,

  minTokensOutRaw,
  minTokensOut,

  tradeFeeRaw,
  creatorFeeRaw,
  platformFeeRaw,
  lpFeeRaw,

  slippageBps,

  crossesCurve,
  migratesToAmm
}
```

Each field answers a different part of the trade preview.

## 🎯 Expected Token Output

For a buy:

```ts
quote.expectedTokensOut
```

is the calculated expected amount of Moonz tokens from the current state.

For example:

```ts
console.log(
  "Expected:",
  quote.expectedTokensOut
);
```

This is the estimate.

It is not the protected minimum.

## 🛡️ Minimum Token Output

The protected value is:

```ts
quote.minTokensOut
```

and its exact raw equivalent:

```ts
quote.minTokensOutRaw
```

The minimum output incorporates the requested slippage tolerance.

Conceptually:

```text
Expected output
      ↓
Apply slippage tolerance
      ↓
Minimum acceptable output
```

The transaction builder uses the protected minimum.

## 🔴 The Sell Quote

The public `MoonzSellQuote` contains:

```ts
{
  mint,
  side,
  phase,
  market,
  quoteAsset,
  quoteDecimals,
  tokenDecimals,

  tokensInRaw,
  tokensIn,

  grossQuoteOutRaw,
  grossQuoteOut,

  expectedQuoteOutRaw,
  expectedQuoteOut,

  minQuoteOutRaw,
  minQuoteOut,

  tradeFeeRaw,
  creatorFeeRaw,
  platformFeeRaw,
  lpFeeRaw,

  slippageBps
}
```

The sell quote distinguishes the gross output from the output expected after the trading fee.

## 💰 Gross Quote Output

For sells:

```ts
quote.grossQuoteOut
```

represents the calculated quote output before the trading fee is removed.

Then:

```ts
quote.expectedQuoteOut
```

represents the expected user output after the trading fee.

Conceptually:

```text
Gross quote output
        ↓
Trading fee
        ↓
Expected quote output
```

## 🛡️ Minimum Sell Output

After slippage protection is applied:

```ts
quote.minQuoteOut
```

becomes the minimum acceptable output for the sell.

Its exact raw representation is:

```ts
quote.minQuoteOutRaw
```

This is the amount the transaction protects.

## 🧮 Slippage Basis Points

Moonz uses basis points for slippage input.

```text
100 basis points
=
1 percent

50 basis points
=
0.5 percent

250 basis points
=
2.5 percent
```

If `slippageBps` is omitted, the Trading SDK defaults to:

```text
100
```

which is:

```text
1 percent
```

## 🧱 Example With Default Slippage

This:

```ts
const quote =
  await moonz.quoteBuy({
    mint,
    amount: "0.1"
  });
```

uses:

```text
100 basis points
```

unless you explicitly provide another supported value.

## 🎛️ Example With Custom Slippage

For half a percent:

```ts
const quote =
  await moonz.quoteBuy({
    mint,
    amount: "0.1",
    slippageBps: 50
  });
```

For two percent:

```ts
const quote =
  await moonz.quoteBuy({
    mint,
    amount: "0.1",
    slippageBps: 200
  });
```

The SDK validates the slippage value before using it.

## ⚠️ Slippage Is Not a Fee

Slippage tolerance and the trading fee are different things.

```text
Trading fee
    ↓
Protocol charge

Slippage tolerance
    ↓
Protection against the execution
moving beyond an acceptable output
```

Increasing slippage does not reduce the trading fee.

It widens the amount of execution movement your transaction is willing to accept.

## 💸 Trading Fee

The current Moonz trading fee is:

```text
125 basis points
```

which is:

```text
1.25 percent
```

The quote exposes the calculated fee through:

```ts
quote.tradeFeeRaw
```

and also exposes the protocol fee components:

```ts
quote.creatorFeeRaw

quote.platformFeeRaw

quote.lpFeeRaw
```

The exact fee composition depends on the current market.

## 🌱 Bonding Fee Behaviour

During bonding, the quote reports its applicable fee split.

For bonding sells, for example:

```text
creator fee

platform fee

LP fee = 0
```

The liquidity provider component becomes relevant to AMM trading.

Applications should consume the quote fields rather than recreating the fee split independently.

## 🚀 AMM Fee Behaviour

In an AMM market, the quote can report:

```text
creator fee

platform fee

LP fee
```

These values are already calculated by the Trading SDK from current protocol rules.

If your user interface wants to display a fee breakdown, use the quote.

## 🔬 Do Not Rebuild the Math in the Frontend

The Trading SDK repository contains internal calculations for:

```text
Bonding curve output

AMM output

Fee calculation

Fee splitting

Slippage floors

Unit conversion
```

Those helpers are implementation details.

They are not exported from the public package entry point.

Instead of copying them into your application:

```ts
const quote =
  await moonz.quoteBuy({
    mint,
    amount,
    slippageBps
  });
```

Let the SDK own protocol trading math.

{% hint style="warning" %}
Duplicating protocol math creates another implementation that must remain exactly synchronized with Moonz.

Use the public quote interface whenever possible.
{% endhint %}

## 🚦 Minimum SOL Trade

The current on chain minimum for a SOL trade input is:

```text
10000 lamports
```

Since one SOL is:

```text
1000000000 lamports
```

the SDK rejects SOL buy amounts below that protocol minimum.

## 💵 Minimum USDC Trade

The current on chain minimum for a USDC trade input is:

```text
10000 base units
```

USDC uses:

```text
6 decimals
```

so the SDK validates a USDC buy against that raw minimum before quoting the AMM trade.

## 🌙 Minimum Moonz Sell

The current minimum Moonz token sell input is:

```text
1000 base units
```

Moonz launch tokens use:

```text
6 decimals
```

The SDK converts the human sell amount into raw token units and rejects values below the protocol minimum.

## 🚫 Untradeable Phases

A quote is not just arithmetic.

The SDK first checks whether the token is currently tradeable.

The public quote surface supports:

```text
BONDING

AMM_LIVE
```

If the Moonz token is in a non tradeable lifecycle state, quoting fails rather than manufacturing a price for a market that cannot execute.

## 🔍 Reserve Validation

AMM quotes require real reserve accounts.

The SDK validates that the required reserve data exists.

For an AMM it needs:

```text
Active quote reserve

LP token reserve
```

If the required reserve account is missing, the quote fails.

That is preferable to pretending a valid trade can be constructed.

## 💧 Liquidity Validation

Sell quotes also protect against impossible output.

For example, a bonding sell checks whether the WSOL treasury has sufficient liquidity for the calculated gross output.

AMM sells verify the current quote reserve can support the required outflow.

If the liquidity is insufficient, quoting fails.

## 🔴 Bonding Sell Limit

During bonding, a seller cannot quote more tokens than the amount currently sold through the bonding curve.

The SDK checks:

```text
tokens being sold
≤
tokens currently sold
```

If not, it rejects the quote.

That prevents a quote from assuming bonding liquidity for tokens outside the valid bonding state.

## 🌉 Crossing From Bonding Into the AMM

A Moonz buy has one particularly interesting case.

The buy may be large enough to finish the remaining bonding allocation.

If so, the quote can identify:

```ts
quote.crossesCurve
```

A value of:

```text
true
```

means one SOL buy spans the bonding curve and the new AMM.

The quote market can become:

```text
BONDING_TO_AMM
```

## 🚀 Migration Flag

The same buy quote also exposes:

```ts
quote.migratesToAmm
```

When true, the trade finishes the bonding allocation and migrates the token into:

```text
AMM_LIVE
```

That makes it possible for a trading interface to tell the user that their trade is at the Moonz lifecycle boundary.

## 🧭 Market Values

A buy quote can expose these market values:

```text
BONDING

AMM

BONDING_TO_AMM
```

A sell quote exposes:

```text
BONDING

AMM
```

The curve crossing case is specific to a buy that consumes the remaining bonding allocation and continues through migration.

## 🧪 Inspect a Buy Quote

A useful development check is:

```ts
const quote =
  await moonz.quoteBuy({
    mint: "TOKEN_MINT",
    amount: "0.1",
    slippageBps: 100
  });

console.log({
  phase:
    quote.phase,

  market:
    quote.market,

  quoteAsset:
    quote.quoteAsset,

  amountIn:
    quote.amountIn,

  expectedTokensOut:
    quote.expectedTokensOut,

  minTokensOut:
    quote.minTokensOut,

  slippageBps:
    quote.slippageBps,

  crossesCurve:
    quote.crossesCurve,

  migratesToAmm:
    quote.migratesToAmm
});
```

That immediately tells you which Moonz market the SDK sees.

## 🧪 Inspect a Sell Quote

For sells:

```ts
const quote =
  await moonz.quoteSell({
    mint: "TOKEN_MINT",
    amount: "100000",
    slippageBps: 100
  });

console.log({
  phase:
    quote.phase,

  market:
    quote.market,

  quoteAsset:
    quote.quoteAsset,

  tokensIn:
    quote.tokensIn,

  grossQuoteOut:
    quote.grossQuoteOut,

  expectedQuoteOut:
    quote.expectedQuoteOut,

  minQuoteOut:
    quote.minQuoteOut
});
```

This is enough information to build a clear trade confirmation screen.

## 🖥️ What a Trading Interface Should Show

Before requesting a signature, a useful trading interface can display:

```text
Input amount

Expected output

Minimum output

Quote asset

Slippage

Trading fee

Current market
```

For a curve crossing buy, it may also display:

```text
This trade crosses from
bonding into the AMM
```

The quote already provides the information required to make that decision.

## 🔁 Quotes Can Become Stale

A quote is calculated from current Solana state.

That state can change after the quote is returned.

Another user may trade.

Reserves may move.

The token may reach migration.

PCLS may change the active quote asset later in the lifecycle.

That is why the transaction carries a minimum output.

```text
Quote
    ↓
Market moves
    ↓
Transaction executes
    ↓
Minimum output still satisfied?
    |
    ├── YES
    |     ↓
    |   Execute
    |
    └── NO
          ↓
        Fail
```

Slippage protection exists because a quote is not a promise that the market will remain frozen.

## 🛡️ Quote Again Before Important Actions

If your interface leaves a quote sitting on screen for a long time, consider refreshing it before the user signs.

A common application flow is:

```text
User enters amount
      ↓
Request quote
      ↓
Display result
      ↓
User changes amount?
      |
      ├── YES
      |     ↓
      |   Quote again
      |
      └── NO
            ↓
        Build trade
```

For highly active markets, your interface may choose a suitable quote refresh strategy.

## 🧱 Quote and Builder Stay Together

When you later call:

```ts
await moonz.buildBuyTransaction(...)
```

the SDK quotes from current state again as part of transaction construction.

The returned built transaction contains its own:

```ts
built.quote
```

That quote is the one associated with the transaction that was actually constructed.

## 👛 High Level Trades Also Return the Quote

The same is true for:

```ts
await moonz.buy(...)
```

and:

```ts
await moonz.sell(...)
```

The successful result includes:

```ts
result.quote
```

so your application can retain the quote associated with the submitted trade.

## 🌌 The Quote Pipeline

The quote layer can now be summarized as:

```text
User amount
      ↓
Moonz token
      ↓
Read current state
      ↓
Determine phase
      ↓
Determine quote asset
      ↓
Read reserves
      ↓
Calculate output
      ↓
Calculate fees
      ↓
Apply slippage
      ↓
Return structured quote
```

No wallet signature has happened.

No transaction has been submitted.

We simply know what the trade should look like.

{% hint style="success" %}
**A quote is the bridge between Moonz protocol state and your trading interface.**

Use it before asking the wallet to sign.
{% endhint %}

## 🟢 Next Stop

We know how much we are spending.

We know what we expect back.

Time to make the transaction.

Next stop:

**Buy Moonz**
