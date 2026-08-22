# 📊 Bonding Tracker

A Moonz bonding tracker answers one simple question:

**How close is this token to its AMM?**

But a useful tracker can show much more than a percentage.

You can display:

```text
Bonding progress

Tokens sold

Tokens remaining

Sale supply

SOL collected

Current price

Market cap

Virtual SOL reserve

Virtual token reserve

Latest trade activity

Migration status
```

Then the tracker can automatically change state when the token reaches its AMM.

{% hint style="success" %}
Use Moonz events to know when something changed.

Use current Moonz state to determine what is true now.
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

We will use:

```text
getToken()

getMarketData()

watchToken()
```

for this build.

## 🌙 Load the Token

Start with the mint:

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

The current lifecycle phase is:

```ts
token.phase
```

For a token currently on the curve:

```text
BONDING
```

## 📊 Bonding Progress

The SDK exposes:

```ts
token.supply
  .bondingProgress
```

The SDK derives this from:

```text
tokens sold
÷
sale supply
×
100
```

The result is a percentage.

For example:

```ts
const progress =
  token.supply
    .bondingProgress;

console.log(
  `${progress}%`
);
```

## 📏 Clamp Visual Progress

Protocol state should be displayed faithfully.

For a visual progress bar, it is still sensible to constrain the CSS width:

```ts
const progressWidth =
  Math.min(
    100,
    Math.max(
      0,
      progress
    )
  );
```

Then:

```tsx
<div
  className="bondingTrack"
>
  <div
    className="bondingFill"
    style={{
      width:
        `${progressWidth}%`
    }}
  />
</div>
```

Keep the actual protocol value separate from presentation constraints.

## 🪙 Sale Supply

The bonding sale allocation is exposed as:

```ts
token.supply.saleRaw
```

This is a raw token amount.

Moonz tokens use:

```text
6 decimals
```

## 📈 Tokens Sold

```ts
token.supply.soldRaw
```

tracks how much of the bonding sale allocation is currently sold.

This value can move:

```text
Up after bonding buys

Down after bonding sells
```

So bonding progress is not guaranteed to move in only one direction.

## 🌘 Tokens Remaining

The SDK exposes:

```ts
token.supply.remainingRaw
```

Conceptually:

```text
remaining
=
sale supply
minus
tokens sold
```

This is often one of the clearest values to place beside a progress bar.

## 🧮 Format Moonz Amounts

Because Moonz uses 6 decimals:

```ts
function formatMoonz(
  raw: string
) {
  const value =
    BigInt(raw);

  const base =
    1_000_000n;

  const whole =
    value / base;

  const fraction =
    (value % base)
      .toString()
      .padStart(
        6,
        "0"
      )
      .replace(
        /0+$/,
        ""
      );

  return fraction
    ? `${whole}.${fraction}`
    : whole.toString();
}
```

Then:

```ts
const sold =
  formatMoonz(
    token.supply.soldRaw
  );

const remaining =
  formatMoonz(
    token.supply
      .remainingRaw
  );
```

## ☀️ SOL Collected

The Launch State exposes:

```ts
token.launchState
  .solCollectedRaw
```

This is denominated in lamports.

The SDK's bonding market calculation combines it with the protocol virtual SOL reserve.

## 🧮 Format SOL

```ts
function formatSol(
  raw: string
) {
  const value =
    BigInt(raw);

  const base =
    1_000_000_000n;

  const whole =
    value / base;

  const fraction =
    (value % base)
      .toString()
      .padStart(
        9,
        "0"
      )
      .replace(
        /0+$/,
        ""
      );

  return fraction
    ? `${whole}.${fraction}`
    : whole.toString();
}
```

Then:

```ts
const collectedSol =
  formatSol(
    token.launchState
      .solCollectedRaw
  );
```

## 🌌 Get the Market View

Now read:

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

During bonding the SDK returns:

```text
phase
BONDING

market
BONDING

priceSource
VIRTUAL_CURVE

tradable
true

quoteAsset
SOL
```

provided the token passes the SDK integrity checks.

## 💰 Current Bonding Price

```ts
market.priceQuote
```

is the current spot price of one whole Moonz token in SOL while bonding.

For example:

```ts
console.log(
  market.priceQuote,
  market.quoteAsset
);
```

## 🌕 Current Market Cap

```ts
market.marketCapQuote
```

is the current total SPL supply multiplied by the spot price.

During bonding it is denominated in:

```text
SOL
```

The SDK does not convert this value to an external fiat currency.

## 🌀 Virtual Quote Reserve

During `BONDING`:

```ts
market.virtualQuoteReserve
```

contains the effective quote reserve used by the current pricing model.

The SDK calculates it from:

```text
117 virtual SOL

+

sol_collected
```

For display:

```ts
const effectiveSol =
  market
    .virtualQuoteReserve
    ?.ui ??
  null;
```

## 🌌 Virtual Token Reserve

During bonding:

```ts
market.virtualTokenReserve
```

contains the effective token side used by the pricing model.

It combines:

```text
760,000,000
virtual tokens

+

remaining sale tokens
```

For display:

```ts
const effectiveTokens =
  market
    .virtualTokenReserve
    ?.ui ??
  null;
```

{% hint style="info" %}
The effective virtual reserve is a pricing value.

It is not the same thing as an SPL token account balance.
{% endhint %}

## 🏦 Show the Real Sale Vault Too

If you want the actual remaining token account balance, use:

```ts
token.reserves
  .saleTokens
  ?.amount.ui
```

This is distinct from:

```ts
market.virtualTokenReserve
```

One is real sale inventory.

The other is the effective token reserve used by the bonding formula.

## 🧩 Create a Bonding Snapshot

A useful application model could be:

```ts
async function loadBondingSnapshot(
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

    phase:
      token.phase,

    market:
      market.market,

    tradable:
      market.tradable,

    quoteAsset:
      market.quoteAsset,

    progress:
      market.bondingProgress,

    saleSupplyRaw:
      token.supply.saleRaw,

    soldRaw:
      token.supply.soldRaw,

    remainingRaw:
      token.supply
        .remainingRaw,

    saleVaultBalance:
      token.reserves
        .saleTokens
        ?.amount.ui ??
      null,

    solCollectedRaw:
      token.launchState
        .solCollectedRaw,

    price:
      market.priceQuote,

    marketCap:
      market.marketCapQuote,

    virtualSol:
      market
        .virtualQuoteReserve
        ?.ui ??
      null,

    virtualTokens:
      market
        .virtualTokenReserve
        ?.ui ??
      null,

    integrity:
      market.integrityAll
  };
}
```

## 🧠 Two Reads Are Intentional Here

`getMarketData()` obtains its own current token snapshot internally.

The example above also calls `getToken()` because the tracker wants additional raw bonding state that is not part of `MoonzMarketData`.

For a small interface this is straightforward.

For a high volume application, design your own caching and refresh layer instead of repeatedly fetching the same mint without limits.

## 🛡️ Integrity Before Presentation

`getMarketData()` refuses to expose canonical price data when the full Moonz integrity check fails.

In that case it returns an unavailable market view.

You should also inspect:

```ts
market.integrityAll
```

before treating the tracker as verified protocol state.

## 🎛️ Example Tracker View

A simple tracker might show:

```text
BONDING

██████████████░░░░░░
72.48%

470,000,000 Moonz sold

180,000,000 Moonz remaining

SOL collected
74.381 SOL

Price
0.00000042 SOL

Market cap
420 SOL
```

The numbers above are only an interface example.

Use live SDK values for the real token.

## 📡 Make the Tracker Live

The initial snapshot is only the beginning.

Now subscribe to the token:

```ts
const stop =
  await moonz.watchToken(
    mint,
    {
      events: [
        "BONDING_BUY",
        "BONDING_SELL",
        "MIGRATED"
      ],

      async onEvent(event) {
        console.log(
          event.type
        );
      }
    }
  );
```

## 🟢 Bonding Buy

When:

```ts
event.type ===
  "BONDING_BUY"
```

the useful tracker values may have changed:

```text
Tokens sold

Tokens remaining

Bonding progress

SOL collected

Price

Market cap

Virtual reserves
```

So refresh the current snapshot.

## 🔴 Bonding Sell

When:

```ts
event.type ===
  "BONDING_SELL"
```

refresh for the same reason.

A sell can move:

```text
Bonding progress down

Tokens remaining up

SOL collected down

Price down
```

according to the resulting current curve state.

## 🧭 Events Are Refresh Triggers

A strong application pattern is:

```text
BONDING_BUY event
or
BONDING_SELL event
        ↓
Something changed
        ↓
Read current state
        ↓
Replace tracker snapshot
```

Do not make the browser maintain a second unofficial copy of the Moonz bonding curve if you do not need one.

## 🧠 Why Reread State

Events tell you what happened.

Current accounts tell you what is true after all confirmed activity observed by your RPC.

Using both gives you:

```text
Events
for responsiveness

State
for canonical current values
```

This also makes reconnect behaviour easier.

## ⚡ Debounce Rapid Refreshes

A busy token may emit several events quickly.

Instead of performing a full RPC refresh for every callback immediately, debounce updates.

For example:

```ts
let refreshTimer:
  ReturnType<
    typeof setTimeout
  > | null =
  null;

function scheduleRefresh() {
  if (refreshTimer) {
    clearTimeout(
      refreshTimer
    );
  }

  refreshTimer =
    setTimeout(
      async () => {
        const snapshot =
          await loadBondingSnapshot(
            mint
          );

        console.log(
          snapshot
        );
      },
      250
    );
}
```

Then:

```ts
const stop =
  await moonz.watchToken(
    mint,
    {
      events: [
        "BONDING_BUY",
        "BONDING_SELL",
        "MIGRATED"
      ],

      onEvent() {
        scheduleRefresh();
      }
    }
  );
```

The debounce value is an application choice, not a Moonz protocol rule.

## 🔐 Deduplicate Events Too

For persistent applications, retain the event identity:

```text
signature
+
eventIndex
```

before triggering durable downstream work.

This avoids recording the same protocol event twice if your own ingestion path sees it more than once.

## 🌕 Detect Migration

The most important tracker transition is:

```text
BONDING
↓
AMM
```

Listen for:

```text
MIGRATED
```

Then refresh immediately.

```ts
if (
  event.type ===
  "MIGRATED"
) {
  const next =
    await loadBondingSnapshot(
      mint
    );

  console.log(
    "Migration detected",
    next
  );
}
```

## 🌊 What Changes After Migration

Once migration is complete, `getMarketData()` changes from:

```text
market
BONDING

priceSource
VIRTUAL_CURVE

virtualQuoteReserve
available

virtualTokenReserve
available
```

to:

```text
market
AMM

priceSource
AMM_RESERVES

tokenReserve
available

quoteReserve
available
```

The same mint now has a different market model.

## 🧭 Do Not Keep Showing a Bonding Price

Once:

```ts
market.market ===
  "AMM"
```

your component should stop describing the current price as a bonding curve price.

A simple UI transition is:

```text
BONDING COMPLETE

100%

Migrated to Moonz AMM

Current market
AMM

Price source
AMM reserves
```

## 🧬 Switch Component Mode

For example:

```ts
function viewMode(
  snapshot
) {
  if (
    snapshot.market ===
    "BONDING"
  ) {
    return "BONDING";
  }

  if (
    snapshot.market ===
    "AMM"
  ) {
    return "MIGRATED";
  }

  return "UNAVAILABLE";
}
```

This lets one component survive the market transition.

## 📊 Progress After Migration

`bondingProgress` remains part of the market data structure after migration.

That is useful as a historical completion indicator.

But after migration, current pricing no longer comes from the bonding reserves.

Use the AMM reserve fields for the live market.

## 🚦 Handle Other Lifecycle States

A tracker can also encounter:

```text
PENDING_DEV_BUY

SWITCHING

UNKNOWN
```

In these states:

```ts
market.market
```

can be:

```text
UNAVAILABLE
```

and:

```ts
market.tradable
```

is false.

Do not force a bonding display when the current protocol state says otherwise.

## 🛡️ Safe Tracker State Machine

A useful interface mapping is:

```text
PENDING_DEV_BUY
      ↓
Launch completing

BONDING
      ↓
Show live progress

AMM_LIVE
      ↓
Bonding complete
Show AMM state

SWITCHING
      ↓
Pool switch in progress

UNKNOWN
      ↓
Protocol state unavailable
```

## 🛰️ Reconnect Correctly

If the WebSocket disconnects, do not attempt to rebuild the missed current state only from the last event you remember.

Instead:

```text
Reconnect
    ↓
Read getToken()
    ↓
Read getMarketData()
    ↓
Replace local snapshot
    ↓
Resume watchToken()
```

Current state is your recovery checkpoint.

## 🔁 A Complete Live Tracker

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

    progress:
      market.bondingProgress,

    soldRaw:
      token.supply.soldRaw,

    remainingRaw:
      token.supply
        .remainingRaw,

    solCollectedRaw:
      token.launchState
        .solCollectedRaw,

    price:
      market.priceQuote,

    marketCap:
      market.marketCapQuote,

    virtualQuoteReserve:
      market
        .virtualQuoteReserve,

    virtualTokenReserve:
      market
        .virtualTokenReserve,

    tokenReserve:
      market.tokenReserve,

    quoteReserve:
      market.quoteReserve,

    integrity:
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
            "Refresh failed",
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
        "BONDING_BUY",
        "BONDING_SELL",
        "MIGRATED"
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

## 🌌 What You Have Built

Your component now understands:

```text
How far bonding has progressed

How many tokens remain

How much SOL the curve has collected

What the current bonding price is

What the current market cap is

What virtual reserves price the curve

When buys move progress forward

When sells move progress backward

When bonding finishes

When pricing moves to AMM reserves
```

{% hint style="success" %}
The tracker does not need to predict when migration happened.

Moonz emits the migration event and the current SDK state confirms the new market.
{% endhint %}

## 🌊 Next Build

Bonding ends with a live Moonz AMM.

Next we build the dashboard for that side of the lifecycle.

Next build:

**AMM Dashboard**
