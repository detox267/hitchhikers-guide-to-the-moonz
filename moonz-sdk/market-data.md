# 📊 Market Data

You found the token.

You know where it is in the Moonz lifecycle.

Now comes the question everyone eventually asks.

**What is it worth right now?**

Moonz markets do not always price themselves the same way.

During bonding, price comes from the Moonz virtual curve.

Once the token reaches the Moonz AMM, price comes from real protocol reserves.

The SDK understands the difference for you.

```ts
const market = await moonz.getMarketData(mint);
```

One call.

The correct Moonz market model for the current state.

{% hint style="info" %}
**Moonz market data follows protocol state.**

The SDK does not assume that every token is always trading through the same market.

It looks at where the token actually is.
{% endhint %}

## 🧭 The Market Object

A successful market read returns `MoonzMarketData`.

Important fields include:

```text
mint

phase
phaseCode

market
priceSource
tradable

quoteAsset
quoteAssetCode

priceQuote
marketCapQuote

totalSupply
bondingProgress

virtualQuoteReserve
virtualTokenReserve

tokenReserve
quoteReserve

integrityAll
```

The interesting part is that some of these fields only make sense during certain parts of the Moonz journey.

So let us follow the token.

## 🌱 While the Token Is Bonding

During `BONDING`, Moonz prices the token using the protocol virtual curve.

The SDK identifies this market as:

```text
market      BONDING
priceSource VIRTUAL_CURVE
tradable    true
quoteAsset  SOL
```

The market is quoted in SOL during bonding.

## 🧮 The Bonding Reserves

Moonz uses immutable virtual curve values.

The virtual quote reserve begins with:

```text
117 SOL
```

Then the SOL collected by the launch is added.

So:

```text
virtual quote reserve

117 SOL
+
SOL collected
```

The virtual token side begins with:

```text
760,000,000 tokens
```

Then the remaining sale tokens are added.

So:

```text
virtual token reserve

760,000,000 tokens
+
remaining sale tokens
```

The SDK exposes both values.

```ts
console.log(
  market.virtualQuoteReserve
);

console.log(
  market.virtualTokenReserve
);
```

Both are returned as `MoonzAmount` values.

That means you have:

```ts
market.virtualQuoteReserve?.raw
market.virtualQuoteReserve?.decimals
market.virtualQuoteReserve?.ui
```

and the same structure for the virtual token reserve.

## 💰 Bonding Price

The bonding price is derived from the ratio between the virtual reserves.

Conceptually:

```text
price

virtual SOL reserve
÷
virtual token reserve
```

The SDK handles the decimal differences between SOL and Moonz tokens before producing the result.

Read it with:

```ts
console.log(
  "Price:",
  market.priceQuote
);
```

`priceQuote` represents the price of one whole token in the current quote asset.

During bonding, that quote asset is SOL.

So if:

```text
priceQuote = 0.00000042
quoteAsset = SOL
```

the meaning is:

```text
1 token = 0.00000042 SOL
```

## 📈 Market Cap During Bonding

Moonz also exposes:

```ts
market.marketCapQuote
```

The SDK calculates this as:

```text
price per whole token
×
current total token supply
```

There is an important word in the field name.

**Quote.**

`marketCapQuote` is denominated in the active quote asset.

During bonding:

```text
quoteAsset = SOL
```

So `marketCapQuote` is denominated in SOL.

It is not automatically converted into dollars.

{% hint style="warning" %}
**Do not label a SOL market cap as USD.**

If your application wants USD market cap while the token is quoted in SOL, you must perform that external conversion yourself.
{% endhint %}

## 🌡️ Bonding Progress Comes With It

The market object also contains:

```ts
market.bondingProgress
```

That makes it easy to display price and bonding progress together.

```ts
console.log({
  price:
    market.priceQuote,

  quote:
    market.quoteAsset,

  marketCap:
    market.marketCapQuote,

  bonding:
    market.bondingProgress
});
```

Now your market card can answer two questions at once.

**What is the current price?**

and:

**How far through bonding is the token?**

## 🚀 Welcome to the AMM

Eventually the pricing model changes.

When the token reaches:

```text
AMM_LIVE
```

the SDK stops using the bonding virtual curve.

Now the market is represented as:

```text
market      AMM
priceSource AMM_RESERVES
tradable    true
```

The price now comes from actual Moonz AMM reserves.

## 🏦 AMM Price

The token side comes from the Moonz LP token reserve.

The quote side depends on the active quote asset.

For a SOL market:

```text
LP token reserve
+
WSOL reserve
```

For a USDC market:

```text
LP token reserve
+
USDC reserve
```

The price is then derived from:

```text
active quote reserve
÷
token reserve
```

with the appropriate token decimal adjustment.

The SDK exposes those reserves directly.

```ts
console.log(
  "Token Reserve:",
  market.tokenReserve
);

console.log(
  "Quote Reserve:",
  market.quoteReserve
);
```

During an AMM market, the virtual reserve fields are `null`.

## 💱 SOL or USDC?

The same AMM market representation works with either supported Moonz quote asset.

Read:

```ts
console.log(
  market.quoteAsset
);
```

You may receive:

```text
SOL
```

or:

```text
USDC
```

The market price and market cap are then denominated in that asset.

For example:

```text
quoteAsset = USDC
priceQuote = 0.0042
```

means:

```text
1 token = 0.0042 USDC
```

And:

```text
marketCapQuote
```

is then denominated in USDC.

## 🧠 Ask Where the Price Came From

One of the most useful market fields is:

```ts
market.priceSource
```

It tells your application how Moonz arrived at the current canonical price.

Possible values are:

```text
VIRTUAL_CURVE
AMM_RESERVES
UNAVAILABLE
```

That means an analytics platform can display the same Moonz token correctly before and after it reaches the AMM.

No guessing.

No hard coded assumption that every market has already bonded.

## 🚦 Is the Market Tradable?

Check:

```ts
market.tradable
```

When Moonz can establish a canonical active market during `BONDING` or `AMM_LIVE`, this is:

```text
true
```

Otherwise it is:

```text
false
```

This gives applications a clean signal for whether Moonz currently considers that market representation tradable.

## 🌑 When the Market Goes Dark

Not every protocol state should produce a price.

Moonz deliberately returns an unavailable market for:

```text
PENDING_DEV_BUY
SWITCHING
CANCELLED
UNKNOWN
```

An unavailable market looks like:

```text
market      UNAVAILABLE
priceSource UNAVAILABLE
tradable    false

priceQuote      null
marketCapQuote  null
```

This is intentional.

Moonz would rather tell your application:

**There is no canonical market here right now**

than manufacture a price that does not represent the current protocol state.

## 🔄 What Happens During PCLS?

When the token is in:

```text
SWITCHING
```

canonical market data becomes unavailable.

That means:

```ts
market.tradable
```

is false.

And:

```ts
market.priceQuote
```

is null.

This matters for PCLS aware applications.

Do not keep presenting the previous market as though nothing is happening.

The protocol has entered a transition state.

We will explore that journey properly in the PCLS section later.

## 🛡️ Integrity Comes First

Before Moonz exposes canonical market data, the SDK checks:

```ts
token.integrity.all
```

If that combined integrity result is false, market data becomes unavailable.

That means the SDK will not expose canonical pricing when the important Moonz account relationships fail its integrity checks.

The market object exposes the result too:

```ts
market.integrityAll
```

{% hint style="success" %}
**Price follows verified Moonz state.**

If the protocol relationships do not pass the SDK integrity checks, Moonz does not pretend everything is fine.
{% endhint %}

## 🧯 Missing AMM Reserves

AMM market data also becomes unavailable when the required reserves cannot establish a valid market.

For example:

The LP token account is missing.

The active quote reserve is missing.

The token reserve is zero.

The quote reserve is zero.

In those situations:

```text
market = UNAVAILABLE
tradable = false
```

Again, no invented price.

## 🔎 Read One Market

Here is a compact market read.

```ts
const market = await moonz.getMarketData(
  mint
);

if (!market) {
  console.log("Not a Moonz token");
  process.exit(0);
}

console.log({
  phase:
    market.phase,

  market:
    market.market,

  source:
    market.priceSource,

  tradable:
    market.tradable,

  quoteAsset:
    market.quoteAsset,

  price:
    market.priceQuote,

  marketCap:
    market.marketCapQuote,

  bonding:
    market.bondingProgress,

  integrity:
    market.integrityAll
});
```

One response now tells you:

Where the token is.

Which market model applies.

Whether the market is tradable.

What asset it is quoted against.

What the current canonical price is.

How the price was derived.

What the market cap is in the quote asset.

How far through bonding the token is.

Whether the SDK integrity checks passed.

## 📊 Build a Market Card

Now turn that into something a user can actually see.

```ts
const market = await moonz.getMarketData(
  mint
);

if (!market) {
  console.log("Not Moonz");
  process.exit(0);
}

const card = {
  phase:
    market.phase,

  market:
    market.market,

  priceSource:
    market.priceSource,

  price:
    market.priceQuote,

  quote:
    market.quoteAsset,

  marketCap:
    market.marketCapQuote,

  bondingProgress:
    market.bondingProgress,

  tradable:
    market.tradable
};

console.log(card);
```

That object could become:

A market card.

A trading terminal row.

A token page.

A Telegram response.

An explorer panel.

An analytics record.

## 🛰️ Build a Market Model That Follows Moonz

The important idea is not the code.

It is the state transition.

```text
BONDING

Price source
VIRTUAL_CURVE

        ↓

AMM_LIVE

Price source
AMM_RESERVES
```

Your application does not need two unrelated integrations.

It can ask Moonz what market exists now.

That is the advantage of canonical market data.

{% hint style="success" %}
**Mission complete.**

Your application can now follow a Moonz token from virtual curve pricing into reserve based AMM pricing without pretending those markets are the same thing.
{% endhint %}

## ⚡ Next Stop

We know how to read Moonz.

We know how to discover Moonz.

We know how to understand the current market.

Now let us stop asking what happened and start listening while it happens.

Next stop:

**Listen to Moonz**
