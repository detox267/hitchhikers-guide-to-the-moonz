# 🛡️ Slippage and Minimum Output

A quote tells you what a trade is expected to return.

Slippage protection decides how far execution is allowed to move before the transaction should fail.

Moonz applies that protection through a minimum output.

{% hint style="success" %}
The expected output is a preview.

The minimum output is the execution boundary.
{% endhint %}

## 🎯 The Two Numbers

For a buy, the important values are:

```ts
quote.expectedTokensOut

quote.minTokensOut
```

For a sell:

```ts
quote.expectedQuoteOut

quote.minQuoteOut
```

They have different jobs.

```text
Expected output
      ↓
What current state predicts

Minimum output
      ↓
Lowest acceptable execution
after slippage protection
```

## 🧮 Basis Points

Moonz expresses slippage as:

```text
slippageBps
```

One basis point is:

```text
0.01 percent
```

So:

```text
50 bps
=
0.5 percent

100 bps
=
1 percent

250 bps
=
2.5 percent
```

## ✅ Default Slippage

If you omit `slippageBps`, the Trading SDK uses:

```text
100
```

which means:

```text
1 percent
```

For example:

```ts
const quote =
  await moonz.quoteBuy({
    mint,
    amount: "0.1"
  });

console.log(
  quote.slippageBps
);
```

will use the SDK default unless another value is supplied.

## 🎛️ Custom Slippage

You can provide a different value:

```ts
const quote =
  await moonz.quoteBuy({
    mint,
    amount: "0.1",
    slippageBps: 50
  });
```

That requests:

```text
0.5 percent
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

## 🚦 Valid Range

The Trading SDK requires `slippageBps` to be an integer from:

```text
0
through
9999
```

Values outside that range are rejected.

The SDK also rejects non integer basis point values.

Conceptually:

```text
0
=
no slippage tolerance

100
=
1 percent

9999
=
99.99 percent
```

A technically valid setting is not automatically a sensible setting for your users.

## ⚠️ Extremely High Slippage

A very high slippage tolerance dramatically lowers the protected minimum.

That can allow a trade to execute at a much worse output than the original quote.

Your interface should choose defaults and limits appropriate for its users.

{% hint style="warning" %}
Do not confuse the largest value accepted by the SDK with a recommended trading setting.
{% endhint %}

## 🧠 How the Minimum Is Calculated

The Trading SDK applies the slippage floor to the expected output.

Conceptually:

```text
minimum output
=
expected output
×
(10000 - slippageBps)
÷
10000
```

The calculation uses integer arithmetic.

The resulting minimum is rounded down to the protocol integer amount.

## 🟢 Buy Protection

Suppose a buy quote expects:

```text
100000 Moonz tokens
```

with:

```text
1 percent slippage
```

The protected minimum is conceptually:

```text
99000 Moonz tokens
```

If execution would produce less than the encoded minimum, the trade should not succeed.

## 🔴 Sell Protection

Suppose a sell expects:

```text
1 SOL
```

and the user allows:

```text
1 percent slippage
```

The protected minimum is conceptually:

```text
0.99 SOL
```

The actual protocol calculation works with raw integer units rather than floating point numbers.

## 🔢 Raw Values Matter

For buys, the exact protected amount is:

```ts
quote.minTokensOutRaw
```

For sells:

```ts
quote.minQuoteOutRaw
```

These raw strings represent exact blockchain integer amounts.

They are the values relevant to transaction construction.

## 🌍 Human Values Are for Display

The corresponding display fields are:

```ts
quote.minTokensOut

quote.minQuoteOut
```

A useful rule is:

```text
Human value
    ↓
Display

Raw value
    ↓
Protocol precision
```

Avoid converting large raw values into ordinary JavaScript numbers.

## 🧱 Buy Transaction Protection

When the SDK builds a buy transaction, it takes:

```ts
quote.minTokensOutRaw
```

and encodes it into the selected Moonz buy instruction.

For the SOL route:

```text
amountIn

minTokensOut
```

For the USDC route:

```text
usdcIn

minTokensOut
```

The minimum is therefore part of the transaction itself.

## 🔴 Sell Transaction Protection

For sells, the builder takes:

```ts
quote.minQuoteOutRaw
```

and encodes it as the minimum acceptable quote output.

For SOL:

```text
tokensIn

minWsolOut
```

For USDC:

```text
tokensIn

minUsdcOut
```

Again, the slippage boundary is carried into the actual protocol instruction.

## 💸 Slippage Is Not the Trading Fee

These are separate concepts.

```text
Trading fee
      ↓
Protocol fee applied to trade

Slippage
      ↓
Execution tolerance
```

The current Moonz trading fee is:

```text
125 basis points
=
1.25 percent
```

Changing `slippageBps` does not change that fee.

## 📉 Slippage Does Not Change Expected Output

Suppose current state produces an expected output of:

```text
1000 tokens
```

Changing slippage does not tell the SDK to expect fewer tokens.

Instead it changes the protected floor.

```text
Expected
1000

1 percent slippage
minimum 990

2 percent slippage
minimum 980
```

The expected market calculation and the execution tolerance remain distinct.

## 🛰️ Why Slippage Exists

Moonz reads live Solana state.

That state can change.

Between quoting and execution:

```text
Another buy may execute

Another sell may execute

Reserves may move

Bonding may progress

Migration may occur

The active route may change
```

A quote cannot freeze the chain.

Slippage protection gives the transaction an acceptable boundary.

## 🔄 Quote State Versus Execution State

Consider:

```text
12:00:00
quoteBuy()

Expected:
100000 tokens

Minimum:
99000 tokens

        ↓

Another trade executes

        ↓

12:00:03
Your transaction executes
```

If the current execution can still satisfy:

```text
at least 99000 tokens
```

the protected output remains acceptable.

If not, the transaction should fail instead of accepting a worse result.

## 🧱 Builders Use a Fresh Quote

This is an important Moonz behaviour.

Calling:

```ts
quoteBuy(...)
```

for your interface does not permanently bind a later transaction to that earlier quote.

When you call:

```ts
buildBuyTransaction(...)
```

the builder quotes current state again.

The same applies to sells.

```text
Display quote
      ↓
Time passes
      ↓
Builder runs
      ↓
Fresh state
      ↓
Fresh quote
      ↓
Fresh minimum output
      ↓
Transaction
```

## 📦 Use built.quote

Because the builder creates a fresh quote, the quote directly associated with the transaction is:

```ts
built.quote
```

For a buy:

```ts
const built =
  await moonz.buildBuyTransaction({
    mint,
    buyer: wallet.publicKey,
    amount,
    slippageBps: 100
  });

console.log(
  built.quote.minTokensOut
);
```

For a sell:

```ts
const built =
  await moonz.buildSellTransaction({
    mint,
    seller: wallet.publicKey,
    amount,
    slippageBps: 100
  });

console.log(
  built.quote.minQuoteOut
);
```

## 👛 High Level Trades Preserve the Quote

The high level methods also return the quote used during construction.

For example:

```ts
const result =
  await moonz.buy({
    mint,
    wallet,
    amount,
    slippageBps: 100
  });

console.log(
  result.quote.expectedTokensOut
);

console.log(
  result.quote.minTokensOut
);
```

For a sell:

```ts
const result =
  await moonz.sell({
    mint,
    wallet,
    amount,
    slippageBps: 100
  });

console.log(
  result.quote.expectedQuoteOut
);

console.log(
  result.quote.minQuoteOut
);
```

## 🖥️ Show Both Values

A trading interface should usually show both:

```text
Expected output

Minimum output
```

For example:

```text
Expected
125420.55 MOONZ

Minimum received
124166.34 MOONZ

Slippage
1 percent
```

That gives the user a clearer picture than displaying expected output alone.

## 💬 Show the Quote Asset Too

For sells, do not display:

```text
Expected output
12.4
```

without identifying the asset.

Instead use:

```ts
quote.quoteAsset
```

so the interface can display something like:

```text
Expected output
12.4 USDC
```

or:

```text
Expected output
0.084 SOL
```

## 🌉 Curve Crossing Still Uses Protection

A buy can cross from bonding into the new AMM.

In that case:

```ts
quote.crossesCurve
```

can be true and:

```ts
quote.market
```

can be:

```text
BONDING_TO_AMM
```

The combined expected output still receives slippage protection.

The caller does not need to calculate separate minimums for each leg.

## 🧬 Keep Protocol Math in the SDK

The Trading SDK already handles:

```text
Unit conversion

Bonding curve math

AMM math

Trading fees

Fee splits

Curve crossing

Slippage floor
```

Do not recreate those calculations in application code unless you have a compelling protocol engineering reason.

For ordinary integrations:

```ts
const quote =
  await moonz.quoteBuy({
    mint,
    amount,
    slippageBps
  });
```

is the supported public calculation surface.

## 🚫 Do Not Calculate With Floating Point Raw Values

Avoid patterns like:

```ts
const raw =
  Number(
    quote.expectedTokensOutRaw
  );
```

for protocol arithmetic.

Raw blockchain values are integer quantities and may exceed safe JavaScript integer precision.

Prefer:

```ts
const raw =
  BigInt(
    quote.expectedTokensOutRaw
  );
```

when exact arithmetic is required.

## 🧪 Inspect Slippage During Development

For buys:

```ts
const quote =
  await moonz.quoteBuy({
    mint,
    amount: "0.1",
    slippageBps: 100
  });

console.log({
  expected:
    quote.expectedTokensOut,

  minimum:
    quote.minTokensOut,

  expectedRaw:
    quote.expectedTokensOutRaw,

  minimumRaw:
    quote.minTokensOutRaw,

  slippageBps:
    quote.slippageBps
});
```

For sells:

```ts
const quote =
  await moonz.quoteSell({
    mint,
    amount: "100000",
    slippageBps: 100
  });

console.log({
  expected:
    quote.expectedQuoteOut,

  minimum:
    quote.minQuoteOut,

  expectedRaw:
    quote.expectedQuoteOutRaw,

  minimumRaw:
    quote.minQuoteOutRaw,

  slippageBps:
    quote.slippageBps
});
```

## 🛑 Zero Minimum Protection

The transaction builders refuse to deliberately build a trade whose calculated protected minimum is zero.

The buy path checks:

```text
input > 0

minTokensOut > 0
```

The sell path checks:

```text
tokensIn > 0

minimum quote output > 0
```

That prevents meaningless zero output instructions from being intentionally constructed by the SDK.

## 🔁 Refreshing Quotes

Your application decides how often to refresh its displayed quote.

A sensible interaction pattern is:

```text
Amount changes
      ↓
Request quote
      ↓
Display output
      ↓
User waits
      ↓
Optional refresh
      ↓
User confirms
      ↓
Builder obtains fresh quote
```

The exact interface refresh strategy is an application decision.

The builder's fresh quote is the transaction construction boundary.

## 🧭 What Slippage Protects

Slippage protection helps defend against:

```text
Market movement

Reserve movement

Price movement between
quote and execution
```

It does not guarantee:

```text
A transaction will succeed

A wallet will sign

An RPC will respond

The token state will remain unchanged

Liquidity will remain available
```

Those are separate parts of transaction execution.

## 🌌 The Protection Layer

The Moonz trade path now looks like:

```text
Current Solana state
        ↓
Expected output
        ↓
Trading fee
        ↓
Expected user output
        ↓
Slippage tolerance
        ↓
Minimum output
        ↓
Encode into transaction
        ↓
Wallet signs
        ↓
Execute
        ↓
Minimum satisfied?
     /       \
   YES       NO
    ↓         ↓
 Success    Fail
```

{% hint style="success" %}
A Moonz quote tells you where the market is.

Minimum output protects how far the transaction is willing to follow it.
{% endhint %}

## 🧱 Next Stop

We have quoted the trade, selected the route and protected the output.

Now we will look at the unsigned Solana transaction itself.

Next stop:

**Build Transactions**
