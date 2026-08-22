# 🌊 AMM Dashboard

Once bonding finishes, a Moonz token enters its live AMM.

The market is now priced from real protocol controlled reserves.

A useful AMM dashboard can show:

```text
Current quote asset

Moonz reserve

Quote reserve

Spot price

Market cap

Last trade time

Trading status

PCLS status

Protocol integrity

Live buys and sells
```

The same dashboard can follow the market if PCLS changes the quote asset between SOL and USDC.

{% hint style="success" %}
Do not hardcode the quote reserve.

Read the current Moonz state and let the active quote asset determine which treasury represents the AMM.
{% endhint %}

## 📦 Create the SDK

```ts
import {
  MoonzSDK
} from "@moonz-fun/sdk";

const moonz =
  new MoonzSDK({
    rpcUrl:
      process.env.SOLANA_RPC_URL,

    wsEndpoint:
      process.env.SOLANA_WSS_URL
  });
```

For this dashboard we will use:

```text
getToken()

getMarketData()

watchToken()
```

## 🌙 Load Current State

```ts
const mint =
  "TOKEN_MINT";

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

The first field to inspect is:

```ts
token.phase
```

A normal live Moonz AMM has:

```text
AMM_LIVE
```

## 🌊 Load the Market

```ts
const market =
  await moonz.getMarketData(
    mint
  );

if (!market) {
  throw new Error(
    "Moonz market unavailable"
  );
}
```

For a valid live AMM, the SDK reports:

```text
market
AMM

priceSource
AMM_RESERVES

tradable
true
```

## 🧬 Actual AMM Reserves

During `AMM_LIVE`, the SDK uses the actual protocol token accounts.

The Moonz side is:

```ts
token.reserves.lpTokens
```

The quote side depends on:

```ts
token.quoteAsset
```

If the current asset is SOL, the quote reserve is:

```ts
token.reserves.wsol
```

If the current asset is USDC, it is:

```ts
token.reserves.usdc
```

## 🪙 Moonz Reserve

The public market view exposes the active Moonz reserve directly as:

```ts
market.tokenReserve
```

For display:

```ts
const moonzReserve =
  market.tokenReserve
    ?.ui ??
  null;
```

For exact calculations:

```ts
const moonzReserveRaw =
  market.tokenReserve
    ?.raw ??
  null;
```

Moonz token amounts use:

```text
6 decimals
```

## 💱 Quote Reserve

The active quote reserve is:

```ts
market.quoteReserve
```

For display:

```ts
const quoteReserve =
  market.quoteReserve
    ?.ui ??
  null;
```

The decimals depend on the active quote asset.

```text
SOL
9 decimals

USDC
6 decimals
```

The `MoonzAmount` returned by the SDK already includes:

```text
raw

decimals

ui
```

so your interface does not have to guess.

## ☀️ SOL Market

When:

```ts
market.quoteAsset ===
  "SOL"
```

the AMM is priced from:

```text
WSOL treasury

×

Moonz LP vault
```

The SDK presents WSOL as the developer friendly quote asset:

```text
SOL
```

## 💵 USDC Market

When:

```ts
market.quoteAsset ===
  "USDC"
```

the AMM uses:

```text
USDC treasury

×

Moonz LP vault
```

No change to your dashboard architecture is required.

The quote asset and reserve simply change with current state.

## 💰 Spot Price

The current spot price is:

```ts
market.priceQuote
```

This represents the price of:

```text
1 whole Moonz token
```

in the current quote asset.

For example:

```ts
const priceLabel =
  market.priceQuote === null
    ? "Unavailable"
    : `${market.priceQuote} ${market.quoteAsset}`;
```

## 🧮 Where the Price Comes From

For a live AMM the SDK calculates price from:

```text
Active quote reserve
÷
Moonz token reserve
```

while accounting for the different token decimals.

Conceptually:

```text
Quote reserve

─────────────

Moonz reserve

=

Quote asset
per Moonz token
```

The SDK performs this calculation from canonical reserve balances.

## 📈 Price Source

Check:

```ts
market.priceSource
```

A live AMM should report:

```text
AMM_RESERVES
```

This gives your interface a useful provenance label.

For example:

```text
Price source

Moonz AMM reserves
```

## 🌕 Market Cap

The SDK also returns:

```ts
market.marketCapQuote
```

This is:

```text
Current spot price

×

Current total SPL supply
```

The result is denominated in:

```text
SOL
```

for a SOL quoted market and:

```text
USDC
```

for a USDC quoted market.

## ⚠️ Market Cap Is Not Fiat Conversion

If the current quote asset is SOL:

```text
marketCapQuote
=
SOL
```

It is not automatically USD.

If your application wants a fiat conversion, that requires an external price source outside the Moonz SDK market calculation.

## 🧭 Active Quote Asset

Your dashboard heading should use:

```ts
market.quoteAsset
```

instead of a fixed label.

For example:

```ts
const pair =
  `${token.metadata?.symbol ?? "MOONZ"} / ${market.quoteAsset}`;
```

This could render:

```text
TOKEN / SOL
```

and later:

```text
TOKEN / USDC
```

after PCLS.

## 🏦 Inspect the Underlying Accounts

For a detailed dashboard you can also show the canonical account addresses.

Moonz LP vault:

```ts
token.vaults.stored
  .lpVault
```

WSOL treasury:

```ts
token.vaults.stored
  .treasuryWsolVault
```

USDC treasury:

```ts
token.vaults.stored
  .treasuryUsdcVault
```

This is useful for advanced explorers and protocol verification tools.

## 🛡️ Stored Versus Derived Addresses

The SDK exposes:

```ts
token.vaults.stored
```

and:

```ts
token.vaults.derived
```

The integrity layer checks whether the addresses stored in Launch State match the deterministic Moonz addresses derived from the mint.

For an ordinary dashboard, use the normalized reserve data.

For a verification interface, exposing this comparison can be valuable.

## ✅ Integrity

Check:

```ts
market.integrityAll
```

and:

```ts
token.integrity.all
```

before presenting the AMM as verified Moonz protocol state.

If integrity fails, `getMarketData()` returns an unavailable market view rather than canonical price data.

{% hint style="warning" %}
Do not calculate your own trusted AMM price from accounts that failed the Moonz integrity checks.

Surface the failure instead.
{% endhint %}

## 🕒 Last Trade

The token object exposes:

```ts
token.timestamps
  .lastTrade
```

This is useful for a simple activity indicator.

For example:

```ts
const lastTrade =
  token.timestamps
    .lastTrade;
```

Treat it as protocol timestamp data rather than a guarantee that no other network activity occurred afterward.

## 🚦 Tradable State

The canonical market view exposes:

```ts
market.tradable
```

For a healthy `AMM_LIVE` market with valid reserves:

```text
true
```

For states such as:

```text
PENDING_DEV_BUY

SWITCHING

CANCELLED

UNKNOWN
```

the SDK returns an unavailable market view with:

```text
tradable
false
```

## 🔀 PCLS State

The token object exposes:

```ts
token.switching
```

Important fields include:

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

## ⏸️ Dashboard During Switching

When:

```ts
token.switching.active
```

is true, your dashboard should not pretend the token is in a normal live AMM.

A useful state is:

```text
POOL SWITCH IN PROGRESS

Current asset
SOL

Pending asset
USDC

Trading
Unavailable
```

The exact assets should come from current SDK state.

## 🌕 After PCLS Completes

After a successful switch:

```text
state
returns to
AMM_LIVE

quote asset
changes

market
returns to
AMM

price source
returns to
AMM_RESERVES
```

Calling `getMarketData()` again automatically selects the newly active quote treasury.

## 🧠 No Manual Reserve Swap Logic Needed

Your dashboard does not need code such as:

```ts
// Avoid maintaining a
// permanent assumption that
// this mint uses SOL.
```

Instead:

```ts
const market =
  await moonz.getMarketData(
    mint
  );
```

already resolves the active reserve according to current Launch State.

## 🧩 Build an AMM Snapshot

A practical application model could be:

```ts
async function loadAmmSnapshot(
  mint: string
) {
  const [
    token,
    market
  ] =
    await Promise.all([
      moonz.getToken(
        mint
      ),

      moonz.getMarketData(
        mint
      )
    ]);

  if (
    !token ||
    !market
  ) {
    return null;
  }

  return {
    mint:
      token.mint,

    name:
      token.metadata?.name ??
      "Unknown",

    symbol:
      token.metadata?.symbol ??
      "",

    phase:
      token.phase,

    market:
      market.market,

    tradable:
      market.tradable,

    quoteAsset:
      market.quoteAsset,

    priceSource:
      market.priceSource,

    price:
      market.priceQuote,

    marketCap:
      market.marketCapQuote,

    moonzReserve:
      market.tokenReserve,

    quoteReserve:
      market.quoteReserve,

    lpVault:
      token.vaults
        .stored.lpVault,

    wsolTreasury:
      token.vaults
        .stored
        .treasuryWsolVault,

    usdcTreasury:
      token.vaults
        .stored
        .treasuryUsdcVault,

    lastTrade:
      token.timestamps
        .lastTrade,

    switching:
      token.switching,

    integrity:
      token.integrity.all &&
      market.integrityAll
  };
}
```

## 🖥️ Example SOL Dashboard

Your interface might display:

```text
TOKEN / SOL

Market
AMM

Trading
Live

Moonz reserve
341,250,000 TOKEN

SOL reserve
108.42 SOL

Spot price
0.0000003177 SOL

Market cap
317.7 SOL

Price source
AMM reserves

Protocol
Verified
```

These numbers are only an interface example.

Use the live SDK values for the real market.

## 💵 Example USDC Dashboard

After PCLS the same component could display:

```text
TOKEN / USDC

Market
AMM

Trading
Live

Moonz reserve
338,900,000 TOKEN

USDC reserve
24,850 USDC

Spot price
0.00007332 USDC

Market cap
73,320 USDC

Price source
AMM reserves

Protocol
Verified
```

Again, these are example display values only.

## 📡 Make the Dashboard Live

Subscribe to:

```text
AMM_BUY

AMM_SELL

POOL_SWITCH
```

For example:

```ts
const stop =
  await moonz.watchToken(
    mint,
    {
      events: [
        "AMM_BUY",
        "AMM_SELL",
        "POOL_SWITCH"
      ],

      onEvent(event) {
        console.log(
          event.type
        );
      }
    }
  );
```

## 🟢 AMM Buy

When:

```ts
event.type ===
  "AMM_BUY"
```

both reserves may have changed.

So refresh:

```text
Moonz reserve

Quote reserve

Price

Market cap

Last trade
```

## 🔴 AMM Sell

When:

```ts
event.type ===
  "AMM_SELL"
```

refresh the same state.

The actual vault balances are the canonical current reserve source.

## 🔀 PCLS Events

The category:

```text
POOL_SWITCH
```

covers:

```text
POOL_SWITCH_STARTED

POOL_SWITCH_SWAP_EXECUTED

POOL_SWITCH_COMPLETED

POOL_SWITCH_CANCELLED
```

These are ideal dashboard refresh triggers.

## ⏸️ Pool Switch Started

When:

```ts
event.type ===
  "POOL_SWITCH_STARTED"
```

refresh current state.

The dashboard should transition from a normal AMM display to a switching state.

## 🧬 Swap Executed

When:

```ts
event.type ===
  "POOL_SWITCH_SWAP_EXECUTED"
```

the reserve conversion has executed, but the lifecycle completion step is still distinct.

Refresh the current Launch State rather than assuming the new market is already active.

## 🌕 Pool Switch Completed

When:

```ts
event.type ===
  "POOL_SWITCH_COMPLETED"
```

reload:

```text
Phase

Quote asset

Moonz reserve

Quote reserve

Price

Market cap
```

The dashboard can then render the newly active market.

## ↩️ Pool Switch Cancelled

If:

```ts
event.type ===
  "POOL_SWITCH_CANCELLED"
```

reload current state.

The original quote market should be reflected by the canonical Launch State after recovery.

## 🧭 Events Trigger Reads

The recommended model is:

```text
AMM or PCLS event
       ↓
Something changed
       ↓
getToken()
+
getMarketData()
       ↓
Replace local snapshot
       ↓
Render current truth
```

Events provide responsiveness.

Current protocol accounts provide the latest state.

## ⚡ Debounce Busy Markets

A busy pool can emit multiple trades quickly.

Use a small application level debounce if necessary:

```ts
let refreshTimer:
  ReturnType<
    typeof setTimeout
  > | null =
  null;

function refreshSoon() {
  if (refreshTimer) {
    clearTimeout(
      refreshTimer
    );
  }

  refreshTimer =
    setTimeout(
      async () => {
        const next =
          await loadAmmSnapshot(
            mint
          );

        console.log(
          next
        );
      },
      250
    );
}
```

The timing is an application choice.

It is not a Moonz protocol parameter.

## 🔐 Persistent Systems Should Deduplicate

If your dashboard also stores event history, use:

```text
signature
+
eventIndex
```

as the event identity.

That keeps the live display and stored activity history separate concerns.

## 🌌 Do Not Invent Total Fiat Liquidity

The Moonz SDK gives you:

```text
Moonz reserve

Quote reserve
```

in their native assets.

It does not provide a fiat valuation oracle for both sides of the pool.

If your interface wants:

```text
Total liquidity in USD
```

you need an appropriate external price source and must clearly identify that additional data source.

## 🛡️ Do Not Mix Bonding and AMM Reserves

During bonding:

```text
virtualQuoteReserve

virtualTokenReserve
```

are meaningful for market pricing.

During `AMM_LIVE`:

```text
tokenReserve

quoteReserve
```

are the live pricing reserves.

Do not keep using the virtual curve after migration.

## 🧬 One Component Across the Lifecycle

You can combine the Bonding Tracker and AMM Dashboard into one market component.

Conceptually:

```text
market = BONDING
      ↓
Render bonding tracker

market = AMM
      ↓
Render AMM dashboard

market = UNAVAILABLE
      ↓
Render lifecycle status
```

This allows one token page to follow the full Moonz market lifecycle.

## 🚀 Complete Live AMM Example

```ts
import {
  MoonzSDK
} from "@moonz-fun/sdk";

const moonz =
  new MoonzSDK({
    rpcUrl:
      process.env.SOLANA_RPC_URL,

    wsEndpoint:
      process.env.SOLANA_WSS_URL
  });

const mint =
  "TOKEN_MINT";

async function loadSnapshot() {
  const [
    token,
    market
  ] =
    await Promise.all([
      moonz.getToken(
        mint
      ),

      moonz.getMarketData(
        mint
      )
    ]);

  if (
    !token ||
    !market
  ) {
    throw new Error(
      "Moonz state unavailable"
    );
  }

  return {
    phase:
      token.phase,

    market:
      market.market,

    tradable:
      market.tradable,

    quoteAsset:
      market.quoteAsset,

    price:
      market.priceQuote,

    marketCap:
      market.marketCapQuote,

    tokenReserve:
      market.tokenReserve,

    quoteReserve:
      market.quoteReserve,

    switching:
      token.switching,

    lastTrade:
      token.timestamps
        .lastTrade,

    integrity:
      token.integrity.all &&
      market.integrityAll
  };
}

let snapshot =
  await loadSnapshot();

console.log(
  snapshot
);

let timer:
  ReturnType<
    typeof setTimeout
  > | null =
  null;

function refreshSoon() {
  if (timer) {
    clearTimeout(
      timer
    );
  }

  timer =
    setTimeout(
      async () => {
        try {
          snapshot =
            await loadSnapshot();

          console.log(
            snapshot
          );
        } catch (
          error
        ) {
          console.error(
            "AMM refresh failed",
            error
          );
        }
      },
      250
    );
}

const stop =
  await moonz.watchToken(
    mint,
    {
      events: [
        "AMM_BUY",
        "AMM_SELL",
        "POOL_SWITCH"
      ],

      onEvent(event) {
        console.log(
          event.type,
          event.signature
        );

        refreshSoon();
      },

      onError(error) {
        console.error(
          "Moonz watch error",
          error
        );
      }
    }
  );

// Later:
// await stop();
```

## 🌊 What You Have Built

Your dashboard now understands:

```text
Whether the market is a live AMM

Which quote asset is active

Which quote treasury is active

The real Moonz reserve

The real quote reserve

Current spot price

Current market cap

Current trading state

Current PCLS state

Live AMM trades

Quote asset changes
```

{% hint style="success" %}
The dashboard follows Moonz state rather than assuming what the market should look like.

If PCLS changes SOL to USDC, the same component can reload the protocol and continue from the new reserve pair.
{% endhint %}

## 🔀 Next Build

We can already show PCLS inside the AMM dashboard.

Next we make it the focus of its own monitoring tool.

Next build:

**PCLS Monitor**
