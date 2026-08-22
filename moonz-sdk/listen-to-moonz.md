# ⚡ Listen to Moonz

Until now, you have been asking Solana questions.

What is this token?

Where is it in the lifecycle?

What is its price?

What market is it using?

Useful.

But sometimes you do not want to keep asking:

**Did something happen yet?**

You want Moonz to tell you when it happens.

That is where the SDK watchers come in.

{% hint style="info" %}
**Reading tells you what Moonz looks like now.**

Watching lets your application react while Moonz moves.
{% endhint %}

## 📡 Two Ways to Listen

The public SDK gives you two main watcher methods.

```ts
moonz.watch()
```

listens across the Moonz protocol.

```ts
moonz.watchToken()
```

follows one specific Moonz token.

Think of them as two different radios.

One listens to the whole Moonz frequency.

The other tunes into one launch.

## 🌐 WebSocket Connection

Live Solana subscriptions use WebSockets.

If your RPC provider supports WebSockets through the normal RPC endpoint, the SDK can use that connection.

If your provider gives you a separate WebSocket endpoint, provide it when creating the client.

```ts
const moonz = new MoonzSDK({
  rpcUrl: "YOUR_SOLANA_RPC",
  wsEndpoint: "YOUR_SOLANA_WSS"
});
```

For simple reads, the RPC endpoint was enough.

For live subscriptions, make sure your Solana provider supports WebSocket connections.

## 🌙 Listen to the Whole Protocol

Start with:

```ts
const stop = await moonz.watch({
  onEvent(event) {
    console.log(event);
  }
});
```

If you omit the `events` filter, every decoded Moonz program event is eligible to reach your callback.

The watcher subscribes to logs from the Moonz program.

When Moonz emits events, the SDK decodes them and passes developer friendly event objects to your application.

## 🎯 Listen for What You Care About

Most applications do not need every event.

You can filter them.

For example:

```ts
const stop = await moonz.watch({
  events: [
    "TOKEN_CREATED",
    "BUY",
    "SELL",
    "MIGRATED"
  ],

  onEvent(event) {
    console.log(event);
  }
});
```

Now the application only receives events matching those filters.

## 🧬 Event Types

The current public SDK exposes these Moonz event types:

```text
TOKEN_CREATED

LAUNCH_ESCROW_FUNDED
LAUNCH_ESCROW_REFUNDED

BONDING_BUY
BONDING_SELL

AMM_BUY
AMM_SELL

FEES_CLAIMED

MIGRATED

POOL_SWITCH_STARTED
POOL_SWITCH_SWAP_EXECUTED
POOL_SWITCH_COMPLETED
POOL_SWITCH_CANCELLED
```

These are developer friendly event names.

The decoded event also keeps the exact Anchor event struct name in:

```ts
event.rawName
```

So deeper integrations can still see the canonical underlying event name.

## 🗂️ Event Categories

You do not always need to filter by one exact event.

The SDK also groups events into categories.

```text
CREATE
ESCROW
BUY
SELL
FEES
MIGRATION
POOL_SWITCH
```

That means:

```ts
events: [
  "BUY"
]
```

can match buy activity without your application separately listing bonding buys and AMM buys.

The same applies to sells.

## 🚀 Listen for Every Trade

Moonz also provides the convenient filter:

```text
TRADE
```

For example:

```ts
const stop = await moonz.watch({
  events: [
    "TRADE"
  ],

  onEvent(event) {
    console.log(event);
  }
});
```

`TRADE` matches both buy and sell categories.

That makes it useful for:

* Live trade feeds
* Trading bots
* Analytics
* Volume tracking
* Notifications
* Market terminals

## 🌌 Listen to Everything

If you explicitly want every decoded Moonz event, you can use:

```ts
events: [
  "ALL"
]
```

Omitting the filter has the same practical effect.

Every decoded Moonz event is eligible.

## 🔭 Watch One Token

Sometimes the entire protocol is too noisy.

You only care about one mint.

Use:

```ts
const stop = await moonz.watchToken(
  mint,
  {
    onEvent(event) {
      console.log(event);
    }
  }
);
```

Before the subscription begins, the SDK verifies that the supplied mint has valid Moonz Launch State.

If it does not, `watchToken()` throws an error.

For a valid Moonz token, the SDK derives its deterministic Launch State PDA and scopes the log subscription to that address.

That gives your application a focused stream for one launch.

## 🎯 Filter One Token

The token watcher supports the same event filters.

```ts
const stop = await moonz.watchToken(
  mint,
  {
    events: [
      "BUY",
      "SELL",
      "MIGRATED"
    ],

    onEvent(event) {
      console.log(event);
    }
  }
);
```

Now your application can follow the trading journey of one Moonz token.

## 🧠 What Does an Event Look Like?

Every decoded Moonz event has a common structure.

Useful fields can include:

```text
type
category
rawName
data

mint
creator
user
executor
feeMint

market
side

quoteAsset
quoteAssetCode

timestamp

signature
slot
blockTime

eventIndex
watchedMint
```

Not every event has every field.

A trade and a pool switch are different things.

The SDK exposes the fields that make sense for the decoded event.

## 📦 The Raw Event Data

Every event also exposes:

```ts
event.data
```

This contains the decoded Anchor event fields converted into JSON safe scalar values.

Large integer values such as `u64`, `u128` and `i64` may be represented as strings.

That matters when storing events in databases or passing them through APIs.

Do not automatically convert large blockchain integers into JavaScript numbers unless you know they are safe.

## 💰 Follow Trades

A token watcher can give you useful trade information.

```ts
const stop = await moonz.watchToken(
  mint,
  {
    events: [
      "BUY",
      "SELL"
    ],

    onEvent(event) {
      console.log({
        type:
          event.type,

        mint:
          event.watchedMint,

        user:
          event.user,

        market:
          event.market,

        side:
          event.side,

        quoteAsset:
          event.quoteAsset,

        signature:
          event.signature
      });
    }
  }
);
```

A trade event may also expose decoded values through `event.data`.

For example:

```ts
console.log(
  event.data.token_amount
);

console.log(
  event.data.quote_amount
);
```

Now you have the beginning of a live trade feed.

## 🌱 Bonding or AMM?

Trade events can tell your application which Moonz market produced the trade.

```ts
event.market
```

The recognised values are:

```text
BONDING
AMM
```

Trade direction is exposed as:

```ts
event.side
```

with:

```text
BUY
SELL
```

This means the same live feed can understand trades before and after bonding completes.

## 🛰️ Watch Selected Mints Across Moonz

The protocol watcher can also receive a mint whitelist.

```ts
const stop = await moonz.watch({
  mints: [
    "TOKEN_MINT_ONE",
    "TOKEN_MINT_TWO"
  ],

  events: [
    "TRADE"
  ],

  onEvent(event) {
    console.log(event);
  }
});
```

This is useful for:

* Watchlists
* Portfolio monitoring
* Selected market alerts
* Trading terminals
* Curated feeds

There is one important limitation.

Events that do not carry a launch mint cannot match this mint filter.

For example, the current `ClaimFeesEvent` payload does not carry the launch mint.

## 👀 watchedMint

`watchToken()` adds useful context to decoded events.

```ts
event.watchedMint
```

This tells you which token the watcher belongs to.

That is particularly useful for events whose raw payload does not contain the launch mint directly.

The original raw event data is left unchanged.

The watcher simply preserves the token context for your application.

## 🧾 Transaction Identity

Live systems need a way to avoid processing the same event twice.

Moonz gives you:

```ts
event.signature
event.eventIndex
```

`eventIndex` is the zero based order of the Moonz event inside the transaction logs.

The SDK documentation specifically allows:

```text
signature + eventIndex
```

to be used as a stable deduplication key.

For example:

```ts
const eventId =
  `${event.signature}:${event.eventIndex}`;
```

That is useful for:

* Bots
* Databases
* Indexers
* Notification systems
* Trade feeds

One transaction can emit more than one Moonz event.

The signature alone is not always enough to identify one specific event.

## 🚫 Failed Transactions

Failed transactions are ignored by default.

You can change that behaviour with:

```ts
includeFailedTransactions: true
```

For example:

```ts
const stop = await moonz.watch({
  includeFailedTransactions: true,

  onEvent(event) {
    console.log(event);
  }
});
```

Most applications should leave failed transactions excluded unless they have a specific reason to inspect them.

## 🧯 Handle Watcher Errors

Both watcher methods support:

```ts
onError(error) {
  console.error(error);
}
```

For example:

```ts
const stop = await moonz.watch({
  events: [
    "TRADE"
  ],

  onEvent(event) {
    console.log(event);
  },

  onError(error) {
    console.error(
      "Watcher error:",
      error
    );
  }
});
```

The SDK protects the subscription from consumer error handlers and rejected callback promises.

Your event handling code should still be written carefully.

A live feed is only as useful as the code consuming it.

## 🛑 Stop Listening

Both `watch()` and `watchToken()` return an unsubscribe function.

```ts
const stop = await moonz.watch({
  onEvent(event) {
    console.log(event);
  }
});
```

When you are finished:

```ts
await stop();
```

Calling the returned function removes the Solana log subscription.

A typical Node application might clean up when it receives `SIGINT`.

```ts
process.on(
  "SIGINT",
  async () => {
    await stop();
    process.exit(0);
  }
);
```

## ⚡ Build a Live Moonz Feed

Let us put the pieces together.

```ts
import { MoonzSDK } from "@moonz-fun/sdk";

const moonz = new MoonzSDK({
  rpcUrl:
    process.env.SOLANA_RPC,

  wsEndpoint:
    process.env.SOLANA_WSS
});

const stop = await moonz.watch({
  events: [
    "TOKEN_CREATED",
    "TRADE",
    "MIGRATED",
    "POOL_SWITCH"
  ],

  onEvent(event) {
    console.log({
      type:
        event.type,

      mint:
        event.mint,

      market:
        event.market,

      side:
        event.side,

      quoteAsset:
        event.quoteAsset,

      signature:
        event.signature,

      eventIndex:
        event.eventIndex
    });
  },

  onError(error) {
    console.error(
      "Moonz watcher error:",
      error
    );
  }
});
```

That one watcher could become:

* A live Moonz terminal
* A trade feed
* A Telegram alert bot
* A launch notification service
* A bonding activity monitor
* A migration tracker
* A PCLS activity feed
* An analytics pipeline

## 🌙 Build a Token Watcher

Want something smaller?

```ts
const stop = await moonz.watchToken(
  mint,
  {
    events: [
      "TRADE",
      "MIGRATED",
      "POOL_SWITCH"
    ],

    onEvent(event) {
      console.log({
        token:
          event.watchedMint,

        type:
          event.type,

        user:
          event.user,

        quoteAsset:
          event.quoteAsset,

        signature:
          event.signature,

        eventIndex:
          event.eventIndex
      });
    }
  }
);
```

Now your application has a live connection to the journey of one Moonz token.

{% hint style="success" %}
**Mission complete.**

You are no longer repeatedly asking Solana whether something changed.

Your application can listen while Moonz moves.
{% endhint %}

## 🕰️ Next Stop

Live events tell you what is happening now.

But what if you already have a transaction signature and want to understand what happened earlier?

Next stop:

**Historical Events**
