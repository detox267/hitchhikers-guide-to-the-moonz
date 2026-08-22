# 📡 Live Trade Feed

A token explorer tells you what a Moonz token looks like now.

A live trade feed tells you what it is doing.

The public Moonz SDK can subscribe to protocol events directly from Solana.

You can react to:

```text
Bonding buys

Bonding sells

AMM buys

AMM sells

Migration

PCLS activity

Fee claims

Launch activity
```

without polling the Moonz website.

{% hint style="success" %}
Moonz events come from program execution on Solana.

Your application can consume the same protocol events used to describe the token lifecycle.
{% endhint %}

## 📦 Start With the SDK

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

A WebSocket capable Solana RPC endpoint is useful for live subscriptions.

## 🌙 Watch One Token

For a token specific feed:

```ts
const mint =
  "TOKEN_MINT";

const stop =
  await moonz.watchToken(
    mint,
    {
      events: [
        "TRADE"
      ],

      onEvent(event) {
        console.log(event);
      }
    }
  );
```

The returned `stop` function removes the subscription.

```ts
await stop();
```

## 🔎 Moonz Verifies the Token First

`watchToken()` does not blindly subscribe to an arbitrary mint.

Before opening the subscription it resolves the Moonz Launch State.

If no Launch State exists, it throws an error indicating that the mint is not a Moonz token.

The subscription is then scoped to the deterministic:

```text
Launch State PDA
```

for that mint.

## 📡 Watch All Trades

The easiest trade filter is:

```text
TRADE
```

For example:

```ts
const stop =
  await moonz.watchToken(
    mint,
    {
      events: [
        "TRADE"
      ],

      onEvent(event) {
        console.log(
          event.side,
          event.market,
          event.user
        );
      }
    }
  );
```

`TRADE` matches events whose category is:

```text
BUY

or

SELL
```

## 🟢 Watch Only Buys

Use the category:

```ts
events: [
  "BUY"
]
```

That includes both:

```text
BONDING_BUY

AMM_BUY
```

## 🔴 Watch Only Sells

```ts
events: [
  "SELL"
]
```

That includes:

```text
BONDING_SELL

AMM_SELL
```

## 🎯 Watch Exact Event Types

You can be more specific:

```ts
events: [
  "BONDING_BUY",
  "BONDING_SELL",
  "AMM_BUY",
  "AMM_SELL"
]
```

This is useful when your application wants different behaviour for the bonding curve and AMM.

## 🧬 Trade Market

Trade events expose:

```ts
event.market
```

For trade events this is:

```text
BONDING

or

AMM
```

So one feed can distinguish the two Moonz market stages.

## 🧭 Trade Side

Trade events also expose:

```ts
event.side
```

as:

```text
BUY

or

SELL
```

A basic display can therefore use:

```ts
function tradeLabel(
  event
) {
  return [
    event.market,
    event.side
  ].join(" ");
}
```

which can produce:

```text
BONDING BUY

BONDING SELL

AMM BUY

AMM SELL
```

## 💱 Quote Asset

Where the raw event contains a quote asset, the SDK normalizes it to:

```ts
event.quoteAsset
```

with values such as:

```text
SOL

USDC
```

The numeric form is also available as:

```ts
event.quoteAssetCode
```

This is especially useful after PCLS because an AMM trade may be quoted in either asset.

## 👤 Trader

Trade events expose the protocol user as:

```ts
event.user
```

You can use it for:

```text
Recent trader display

Wallet activity pages

Trade history

Bot notifications

Analytics
```

## 🪪 Event Type

Every decoded event has:

```ts
event.type
```

Possible trade types are:

```text
BONDING_BUY

BONDING_SELL

AMM_BUY

AMM_SELL
```

This is the normalized Moonz SDK event name.

## 🧬 Raw Anchor Event Name

The SDK also preserves:

```ts
event.rawName
```

For example:

```text
BuyEvent

SellEvent

AmmBuyEvent

AmmSellEvent
```

Use `type` for most application logic.

Use `rawName` when you specifically need the canonical Anchor event struct name.

## 📦 Event Data

The original event fields are exposed through:

```ts
event.data
```

Trade event data includes protocol values corresponding to fields such as:

```text
mint

user

quote asset

input amount

input mint

output amount

output mint

quote amount

token amount

trade fee

creator fee

platform fee

LP fee

tokens sold total

quote collected total

timestamp
```

The exact fields depend on the underlying event.

## 🔢 Large Integers Stay Safe

Moonz event values such as:

```text
u64

u128

i64
```

are converted into JSON safe scalar values.

Large Anchor integer values are generally represented as strings.

That means you should not immediately convert every protocol amount into a JavaScript `number`.

For example:

```ts
const quoteAmount =
  event.data.quoteAmount ??
  event.data.quote_amount;

console.log(
  quoteAmount
);
```

Keep raw integer amounts as strings or convert them to `BigInt` when doing exact arithmetic.

## 💰 Format Quote Amounts

SOL uses:

```text
9 decimals
```

USDC uses:

```text
6 decimals
```

A simple exact formatter can use `BigInt`.

```ts
function formatRaw(
  raw: string,
  decimals: number
) {
  const value =
    BigInt(raw);

  const base =
    10n ** BigInt(decimals);

  const whole =
    value / base;

  const fraction =
    (value % base)
      .toString()
      .padStart(
        decimals,
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
const decimals =
  event.quoteAsset === "USDC"
    ? 6
    : 9;
```

## 🪙 Moonz Token Amounts

Moonz launch tokens use:

```text
6 decimals
```

So a raw Moonz token amount can be displayed with:

```ts
const tokenUi =
  formatRaw(
    String(
      event.data.tokenAmount ??
      event.data.token_amount ??
      "0"
    ),
    6
  );
```

## 🛡️ Prefer the Event Meaning Over Guessing Direction

Trade events already tell you:

```text
market

side

quote asset
```

Do not infer whether a transaction was a buy or sell by looking only at token balance movement.

Use the normalized protocol event.

## 📜 Transaction Signature

Live subscription events include:

```ts
event.signature
```

This is useful for linking the feed entry back to the Solana transaction.

## 🛰️ Slot

The subscription also provides:

```ts
event.slot
```

which identifies the Solana slot associated with the observed log notification.

## ⏱️ Protocol Timestamp

Where the underlying Moonz event contains a timestamp, the SDK exposes:

```ts
event.timestamp
```

The full raw value remains available through:

```ts
event.data
```

## 🧷 Event Index

A single Solana transaction can emit more than one Moonz event.

The SDK assigns:

```ts
event.eventIndex
```

as the zero based order of each decoded Moonz event inside that transaction's logs.

For example:

```text
Transaction ABC

eventIndex 0

eventIndex 1

eventIndex 2
```

## 🔐 Deduplicate Correctly

Do not use only:

```text
transaction signature
```

as the unique identity of a Moonz event.

One transaction can contain multiple Moonz events.

The SDK specifically supports:

```text
signature
+
eventIndex
```

as a stable deduplication key.

For example:

```ts
function eventKey(
  event
) {
  if (
    event.signature === undefined ||
    event.eventIndex === undefined
  ) {
    return null;
  }

  return [
    event.signature,
    event.eventIndex
  ].join(":");
}
```

A stored key might look like:

```text
5abc...xyz:0
```

## 🧠 Why This Matters at Migration

One transaction can contain multiple meaningful Moonz events.

For example, a buy reaching the end of bonding can cause lifecycle activity around migration.

Treat events individually instead of reducing the whole transaction to one database row.

## 🌉 Watch Migration Too

A live token page should usually know when bonding ends.

Add:

```ts
events: [
  "TRADE",
  "MIGRATED"
]
```

Then:

```ts
onEvent(event) {
  if (
    event.type ===
    "MIGRATED"
  ) {
    console.log(
      "Moonz entered AMM"
    );
  }
}
```

When this happens your application can refresh:

```text
Phase

Price source

LP reserve

Quote reserve

Market data
```

## 🔀 Watch PCLS

The event category:

```text
POOL_SWITCH
```

matches Moonz PCLS activity.

So a useful token subscription is:

```ts
const stop =
  await moonz.watchToken(
    mint,
    {
      events: [
        "TRADE",
        "MIGRATED",
        "POOL_SWITCH"
      ],

      onEvent(event) {
        console.log(event);
      }
    }
  );
```

## 🧬 PCLS Event Types

The normalized PCLS event types are:

```text
POOL_SWITCH_STARTED

POOL_SWITCH_SWAP_EXECUTED

POOL_SWITCH_COMPLETED

POOL_SWITCH_CANCELLED
```

These let your interface react to the complete switching lifecycle.

## ⏸️ When PCLS Starts

If:

```ts
event.type ===
  "POOL_SWITCH_STARTED"
```

your application can refresh the token state and display:

```text
Pool switch in progress

Trading temporarily unavailable
```

## 🌕 When PCLS Completes

If:

```ts
event.type ===
  "POOL_SWITCH_COMPLETED"
```

refresh:

```text
Quote asset

AMM reserves

Market price

Market cap

Trading state
```

The token may now be trading against a different quote asset.

## ↩️ When PCLS Is Cancelled

```ts
if (
  event.type ===
  "POOL_SWITCH_CANCELLED"
) {
  // Refresh the current token state.
}
```

The original AMM market is active again after successful recovery.

## 🧩 Build a Feed Entry

A practical feed does not need to store the entire SDK event as its display model.

For example:

```ts
function toFeedEntry(
  event
) {
  return {
    id:
      event.signature !== undefined &&
      event.eventIndex !== undefined
        ? `${event.signature}:${event.eventIndex}`
        : null,

    type:
      event.type,

    market:
      event.market ?? null,

    side:
      event.side ?? null,

    mint:
      event.mint ??
      event.watchedMint ??
      null,

    user:
      event.user ?? null,

    quoteAsset:
      event.quoteAsset ??
      null,

    timestamp:
      event.timestamp ??
      null,

    signature:
      event.signature ??
      null,

    slot:
      event.slot ??
      null,

    data:
      event.data
  };
}
```

This gives your interface one predictable event shape.

## 📚 Keep a Small In Memory Feed

For a browser interface:

```ts
const recent = [];

function addEvent(
  event
) {
  const item =
    toFeedEntry(
      event
    );

  recent.unshift(
    item
  );

  if (
    recent.length > 100
  ) {
    recent.pop();
  }
}
```

For persistent history, store the events in a database instead.

## 🛡️ Deduplicate Before Inserting

```ts
const seen =
  new Set<string>();

function addOnce(
  event
) {
  const key =
    eventKey(
      event
    );

  if (
    key &&
    seen.has(key)
  ) {
    return;
  }

  if (key) {
    seen.add(key);
  }

  addEvent(
    event
  );
}
```

For a production database, put the same uniqueness rule at the storage layer.

## 🚦 Failed Transactions

By default:

```text
Failed transactions
are ignored
```

That is normally what you want for a public activity feed.

If you explicitly need failed transaction events for diagnostics:

```ts
includeFailedTransactions:
  true
```

Do not enable that merely for ordinary trade history.

## 🚨 Handle Subscription Errors

Supply:

```ts
onError(error) {
  console.error(
    "Moonz subscription error",
    error
  );
}
```

For example:

```ts
const stop =
  await moonz.watchToken(
    mint,
    {
      events: [
        "TRADE",
        "MIGRATED",
        "POOL_SWITCH"
      ],

      onEvent(event) {
        addOnce(
          event
        );
      },

      onError(error) {
        console.error(
          error
        );
      }
    }
  );
```

## ⚙️ Keep Event Processing Lightweight

A live subscription callback should not perform large blocking jobs.

A good pattern is:

```text
Receive event
      ↓
Validate dedup key
      ↓
Queue or store event
      ↓
Update interface
```

If you need expensive analytics, process them outside the immediate display path.

## 🧭 Preserve Ordering When It Matters

`onEvent` may return a promise.

The SDK catches asynchronous callback failures and forwards them to `onError`, but application work should not assume long asynchronous handlers will automatically serialize every event for you.

If strict processing order matters, place events into your own queue.

## 📡 Watch the Whole Moonz Protocol

`watchToken()` is ideal for one token.

For a protocol wide feed, use:

```ts
const stop =
  await moonz.watch({
    events: [
      "TRADE",
      "MIGRATED",
      "POOL_SWITCH"
    ],

    onEvent(event) {
      console.log(
        event
      );
    }
  });
```

`watch()` subscribes to the Moonz Program ID rather than one Launch State.

## 🌎 New Token Feed

A protocol wide application can also watch:

```ts
const stop =
  await moonz.watch({
    events: [
      "TOKEN_CREATED"
    ],

    onEvent(event) {
      console.log(
        "New Moonz token",
        event.mint
      );
    }
  });
```

This can power:

```text
New launch feeds

Discovery screens

Launch notifications

Analytics ingestion
```

## 🎯 Filter Protocol Wide Watching by Mint

`watch()` also supports a mint whitelist.

```ts
const stop =
  await moonz.watch({
    mints: [
      mintA,
      mintB
    ],

    events: [
      "TRADE"
    ],

    onEvent(event) {
      console.log(
        event
      );
    }
  });
```

This is useful when one service follows several selected Moonz tokens.

## 🌙 watchToken Versus watch

Use:

```text
watchToken
```

when your application is focused on one known Moonz token.

Use:

```text
watch
```

when you need program wide events or a selected mint list.

## 📜 Live Events Are Not Historical Replay

A live subscription begins watching from the point at which you connect.

It is not a historical event database.

If you already know a transaction signature, historical Moonz events can be recovered with:

```ts
const events =
  await moonz
    .getTransactionEvents(
      signature
    );
```

That reads the transaction and parses its Moonz logs.

## 🔄 Reconnect Strategy

A production live feed should assume WebSocket connections can disconnect.

A robust architecture is:

```text
Persist processed event keys

Connect subscription

Process new events

Connection lost

Reconnect

Recover any known missing
transactions when possible

Continue live processing
```

Do not treat one WebSocket connection as permanent storage.

## 🧱 RPC Is Transport, Not History

`watch()` and `watchToken()` use Solana log subscriptions.

They are excellent for live application behaviour.

If your goal is a durable protocol indexer with replay and long term storage, the Moonz Geyser interface is the stronger ingestion surface.

## 🛰️ Live Feed Architecture

A simple application can look like:

```text
Solana RPC
WebSocket
    ↓
Moonz SDK
watchToken
    ↓
Decoded Moonz events
    ↓
Deduplicate
signature + eventIndex
    ↓
Application state
    ↓
Live trade feed
```

A larger system can use:

```text
Moonz Geyser
    ↓
Indexer
    ↓
Database
    ↓
API or WebSocket
    ↓
Many clients
```

## 🚀 Complete Token Feed Example

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

const seen =
  new Set<string>();

const stop =
  await moonz.watchToken(
    mint,
    {
      events: [
        "TRADE",
        "MIGRATED",
        "POOL_SWITCH"
      ],

      onEvent(event) {
        const key =
          event.signature !== undefined &&
          event.eventIndex !== undefined
            ? `${event.signature}:${event.eventIndex}`
            : null;

        if (
          key &&
          seen.has(key)
        ) {
          return;
        }

        if (key) {
          seen.add(key);
        }

        if (
          event.category === "BUY" ||
          event.category === "SELL"
        ) {
          console.log({
            type:
              event.type,

            market:
              event.market,

            side:
              event.side,

            trader:
              event.user,

            quoteAsset:
              event.quoteAsset,

            data:
              event.data,

            signature:
              event.signature,

            eventIndex:
              event.eventIndex
          });

          return;
        }

        if (
          event.type ===
          "MIGRATED"
        ) {
          console.log(
            "Token migrated to AMM"
          );

          return;
        }

        if (
          event.category ===
          "POOL_SWITCH"
        ) {
          console.log(
            "PCLS event",
            event.type
          );
        }
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

That is enough to power a live token activity panel.

{% hint style="success" %}
The same normalized event interface works across bonding, AMM migration and PCLS.

Your feed can follow a Moonz token as its market changes without changing event infrastructure.
{% endhint %}

## 📊 Next Build

Now we have current state and live activity.

Next we combine them into a focused market component.

Next build:

**Bonding Tracker**
