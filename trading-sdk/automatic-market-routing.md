# 🛣️ Automatic Market Routing

Moonz tokens do not stay in one market forever.

A token can begin on the bonding curve.

It can move into an AMM.

Its AMM can use SOL.

Its AMM can use USDC.

Your trading integration should not need a different button for every possible route.

That is why the Moonz Trading SDK handles route selection automatically.

{% hint style="success" %}
Your application chooses **buy** or **sell**.

The SDK determines which Moonz instruction should execute.
{% endhint %}

## 🌌 One Public Trading Interface

The public methods remain:

```text
quoteBuy()

quoteSell()

buildBuyTransaction()

buildSellTransaction()

buy()

sell()
```

You do not call separate public methods such as:

```text
buyBonding()

buySolAmm()

buyUsdcAmm()
```

Those are not required.

The SDK reads current Moonz state and routes the trade.

## 🌱 Route One

A token begins its tradeable lifecycle in:

```text
BONDING
```

The quote asset is:

```text
SOL
```

The canonical trading instructions are:

```text
buy

sell
```

So:

```ts
await moonz.buy({
  mint,
  wallet,
  amount: "0.1"
});
```

routes to:

```text
buy
```

And:

```ts
await moonz.sell({
  mint,
  wallet,
  amount: "100000"
});
```

routes to:

```text
sell
```

## 🚀 Route Two

After bonding completes, Moonz can reach:

```text
AMM_LIVE
```

If the active quote asset is:

```text
SOL
```

the canonical trade instructions remain:

```text
buy

sell
```

The market has changed.

The public trading call has not.

## 💵 Route Three

An `AMM_LIVE` Moonz market can also use:

```text
USDC
```

In that state, the SDK selects:

```text
amm_buy_usdc

amm_sell_usdc
```

Your application still calls:

```ts
moonz.buy(...)
```

or:

```ts
moonz.sell(...)
```

## 🧭 The Routing Table

The current public routing contract is:

```text
BUY

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

And:

```text
SELL

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

That table is implemented inside the SDK.

## 🔎 How a Buy Route Is Selected

`buildBuyTransaction()` first reads the token's current Launch State.

Conceptually:

```text
Mint
  ↓
getLaunchState()
  ↓
Does state exist?
  |
  ├── NO
  |     ↓
  |   Reject
  |
  └── YES
        ↓
AMM_LIVE + USDC?
  |
  ├── YES
  |     ↓
  | amm_buy_usdc
  |
  └── NO
        ↓
     SOL builder
        ↓
Validate BONDING
or AMM_LIVE + SOL
```

The final builder performs its own validation as well.

## 🔴 How a Sell Route Is Selected

`buildSellTransaction()` follows the same pattern.

```text
Mint
  ↓
getLaunchState()
  ↓
Does state exist?
  |
  ├── NO
  |     ↓
  |   Reject
  |
  └── YES
        ↓
AMM_LIVE + USDC?
  |
  ├── YES
  |     ↓
  | amm_sell_usdc
  |
  └── NO
        ↓
     SOL builder
        ↓
Validate BONDING
or AMM_LIVE + SOL
```

This keeps buy and sell route behaviour consistent.

## 🛰️ Launch State Is the Source of Truth

The route is chosen from current Moonz Launch State.

Important fields include:

```text
phase

quoteAsset
```

The SDK does not decide the route from:

```text
A cached frontend label

A token symbol

A remembered market

A hardcoded token list
```

It rereads protocol state.

## 🚫 Do Not Hardcode Bonding Forever

This would be fragile:

```ts
if (mint === KNOWN_TOKEN) {
  useBondingBuy();
}
```

A Moonz token can leave bonding.

Once it does, that assumption is stale.

Prefer:

```ts
await moonz.buy({
  mint,
  wallet,
  amount
});
```

and let the SDK inspect current state.

## 🚫 Do Not Hardcode SOL Forever

This is also unsafe:

```ts
const quoteAsset =
  "SOL";
```

An `AMM_LIVE` Moonz token can operate with:

```text
SOL

or

USDC
```

Your interface should read:

```ts
quote.quoteAsset
```

from the current quote.

## 💱 Amount Meaning Depends on the Route

For buys, `amount` always means:

```text
amount of current quote asset
```

So:

```ts
amount: "1"
```

can mean:

```text
1 SOL
```

or:

```text
1 USDC
```

depending on the current market.

That is why the interface should show the quote asset clearly before the wallet signs.

## 🔴 Sell Input Does Not Change

For sells, the input amount remains:

```text
Moonz token quantity
```

regardless of whether the market returns SOL or USDC.

For example:

```ts
await moonz.quoteSell({
  mint,
  amount: "25000"
});
```

always means:

```text
sell 25000 Moonz tokens
```

The returned quote tells you whether the expected proceeds are SOL or USDC.

## 🌉 The Special Buy Route

Moonz has one important transition case.

A SOL buy can begin while the token is still in bonding and be large enough to finish the remaining bonding allocation.

The quote can report:

```text
BONDING_TO_AMM
```

This is a market value exposed through the buy quote.

## 🌱 ➜ 🚀 One Buy, Two Markets

Conceptually:

```text
SOL buy begins
      ↓
Bonding allocation remains
      ↓
Trade consumes remaining
bonding allocation
      ↓
Migration occurs
      ↓
Remaining part of buy
continues into AMM
```

The SDK calls this condition:

```ts
quote.crossesCurve
```

## 🌉 crossesCurve

When:

```ts
quote.crossesCurve === true
```

one SOL buy uses both:

```text
Bonding curve

and

new AMM
```

The public application does not need to split that into two separate user trades.

## 🚀 migratesToAmm

The buy quote also exposes:

```ts
quote.migratesToAmm
```

When true, the buy finishes the bonding allocation and causes the token to enter:

```text
AMM_LIVE
```

This can be useful information to show in a confirmation interface.

## 🧬 Market Is Not the Same as Phase

This distinction is useful.

The Moonz state phase can be:

```text
BONDING

AMM_LIVE
```

But the buy quote market can be:

```text
BONDING

AMM

BONDING_TO_AMM
```

`BONDING_TO_AMM` describes how that particular buy is expected to execute.

It is not a separate long lived Launch State phase.

## 🧭 Example Route Inspection

You can inspect a quote before trading:

```ts
const quote =
  await moonz.quoteBuy({
    mint,
    amount: "0.5",
    slippageBps: 100
  });

console.log({
  phase:
    quote.phase,

  market:
    quote.market,

  quoteAsset:
    quote.quoteAsset,

  crossesCurve:
    quote.crossesCurve,

  migratesToAmm:
    quote.migratesToAmm
});
```

That gives your interface a clear view of the route the quote expects.

## 🧱 The Builder Checks Again

Do not assume that because a preview quote said:

```text
BONDING
```

the token must still be bonding when the user finally presses Buy.

The builder rereads current state.

Conceptually:

```text
quoteBuy()
      ↓
User reads quote
      ↓
Other transactions occur
      ↓
buildBuyTransaction()
      ↓
Read Launch State again
      ↓
Choose current route
```

This protects the transaction builder from relying on stale UI state.

## 🔄 Why This Matters

Imagine your application quotes a token while it is close to the end of bonding.

Before the user approves the trade, another wallet completes bonding.

Your old frontend state may still say:

```text
BONDING
```

but the chain now says:

```text
AMM_LIVE
```

The builder reads the current state and constructs the transaction for the current route.

## 💵 PCLS Makes Route Awareness More Important

Once an AMM is live, the active quote asset matters.

A Moonz market can use:

```text
SOL

or

USDC
```

PCLS is the protocol mechanism that allows the pool quote asset to evolve.

We will explain the protocol mechanics later.

For the Trading SDK, the important rule is simpler:

```text
Read current quote asset

Do not assume it
```

## 🛡️ Switching Is Not a Trade Route

Not every Moonz phase is tradeable.

The public quote and builder surfaces support active trading during:

```text
BONDING

AMM_LIVE
```

A lifecycle state such as:

```text
SWITCHING
```

is not treated as another trade route.

If trading is unavailable, the SDK rejects the quote or builder rather than guessing what should execute.

## 🎯 Route Selection Versus Slippage

Automatic routing solves:

```text
Which market should execute?
```

Slippage protection solves:

```text
How much execution movement
will the transaction accept?
```

They are related but separate concerns.

A transaction can choose the correct route and still fail because the market moved beyond its protected minimum output.

## 🧠 Keep Application Logic Simple

A strong integration can reduce its decision logic to:

```text
User wants BUY?
      ↓
quoteBuy()
      ↓
buy()

User wants SELL?
      ↓
quoteSell()
      ↓
sell()
```

The SDK handles the protocol route.

Your interface handles the user experience.

## 🖥️ Recommended Display Logic

For a buy interface:

```ts
const quote =
  await moonz.quoteBuy({
    mint,
    amount,
    slippageBps
  });

show({
  market:
    quote.market,

  quoteAsset:
    quote.quoteAsset,

  expectedTokens:
    quote.expectedTokensOut,

  minimumTokens:
    quote.minTokensOut
});
```

If:

```ts
quote.crossesCurve
```

is true, you can additionally explain that the trade crosses the bonding boundary.

## 🔴 Recommended Sell Display

For sells:

```ts
const quote =
  await moonz.quoteSell({
    mint,
    amount,
    slippageBps
  });

show({
  market:
    quote.market,

  quoteAsset:
    quote.quoteAsset,

  expectedOutput:
    quote.expectedQuoteOut,

  minimumOutput:
    quote.minQuoteOut
});
```

Again, the current quote asset comes from Moonz state.

## 🛰️ Debugging the Route

When using a builder, you can inspect:

```ts
built.instruction
```

For buys:

```text
buy

amm_buy_usdc
```

For sells:

```text
sell

amm_sell_usdc
```

This gives you the exact canonical instruction selected by the Trading SDK.

## ✅ High Level Results Preserve the Route

The high level trade result also returns:

```ts
result.instruction
```

So after confirmation your application can record which Moonz instruction actually executed.

For example:

```ts
const result =
  await moonz.buy({
    mint,
    wallet,
    amount
  });

console.log(
  result.instruction
);
```

## 🌌 Route Ownership

The design principle is:

```text
MOONZ PROGRAM STATE
        ↓
TRADING SDK
        ↓
ROUTE SELECTION
        ↓
TRANSACTION
        ↓
WALLET
```

Not:

```text
FRONTEND GUESS
      ↓
HARDCODED ROUTE
      ↓
TRANSACTION
```

The chain decides what state exists.

The SDK translates that state into the correct trading route.

## 🧭 Routing Rules to Remember

```text
Do not hardcode bonding

Do not hardcode SOL

Read quoteAsset

Use quote market

Let the builder reread state

Let the SDK choose instruction

Treat BONDING_TO_AMM
as a trade path

Do not treat SWITCHING
as a trade route
```

{% hint style="success" %}
The Moonz market can evolve without forcing your integration to rewrite its buy and sell interface.
{% endhint %}

## 🛡️ Next Stop

The SDK now knows where to send the trade.

Next we look at the protection that decides whether the trade should still execute once it gets there.

Next stop:

**Slippage and Minimum Output**
