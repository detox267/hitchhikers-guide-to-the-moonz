# 🕰️ Historical Events

Live watchers tell you what is happening now.

But the chain remembers.

Maybe you already have a Solana transaction signature.

Maybe your application is processing transactions from another source.

Maybe you are rebuilding history.

Maybe somebody asks:

**What actually happened in this transaction?**

The Moonz SDK can help you answer that too.

{% hint style="info" %}
**Live events and historical events speak the same Moonz language.**

Whether an event arrives through a watcher or is decoded from an existing transaction, the SDK returns the same developer friendly `MoonzDecodedEvent` structure.
{% endhint %}

## 🔎 Start With a Transaction

If you already have a Solana transaction signature, use:

```ts
const events =
  await moonz.getTransactionEvents(
    signature
  );
```

The result is an array of decoded Moonz events.

```ts
for (const event of events) {
  console.log(event);
}
```

Instead of manually interpreting Moonz program logs, your application receives structured event data.

## 🌙 What Can Be Decoded?

The current public SDK understands Moonz events including:

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

The SDK maps the underlying Anchor events into these developer friendly names.

It also preserves the exact Anchor event name in:

```ts
event.rawName
```

That gives you both views.

A clean Moonz event type for application logic.

And the canonical underlying event name when you need to go deeper.

## 🧭 Understand the Event

Every decoded event contains:

```ts
event.type
event.category
event.rawName
event.data
```

Depending on the event, it may also contain information such as:

```text
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
```

Not every event contains every field.

A buy event and a pool switch event are describing different things.

The common structure gives your application one consistent way to process them.

## 📦 Keep the Original Data

The decoded Anchor event fields are available through:

```ts
event.data
```

For example:

```ts
console.log(
  event.data
);
```

This is useful when your application needs information beyond the common fields Moonz exposes at the top level.

Values are converted into JSON safe scalar values.

Large blockchain integers may be represented as strings.

That is intentional.

JavaScript numbers cannot safely represent every integer used by Solana programs.

{% hint style="warning" %}
**Treat large raw amounts carefully.**

If an event amount is returned as a string, keep it as an exact integer representation or use a suitable big integer library.

Do not automatically force large blockchain values into JavaScript numbers.
{% endhint %}

## 💰 Find Trades in a Transaction

You can inspect the decoded events and select trades.

```ts
const events =
  await moonz.getTransactionEvents(
    signature
  );

const trades =
  events.filter(
    event =>
      event.category === "BUY" ||
      event.category === "SELL"
  );

console.log(trades);
```

Now a transaction signature can become a structured Moonz trade record.

A trade event can expose information such as:

```ts
event.market
event.side
event.mint
event.user
event.quoteAsset
event.data
```

This can be useful for:

* Trading history
* Analytics
* Bots
* Explorers
* Transaction pages
* Historical databases

## 🌱 Bonding or AMM?

Trade events also tell you which Moonz market produced the activity.

```ts
console.log(
  event.market
);
```

The recognised trade markets are:

```text
BONDING
AMM
```

And the trade side is:

```text
BUY
SELL
```

That means historical data can preserve the market context too.

A buy during bonding is not silently treated as though it happened through the AMM.

## 🛰️ Already Have the Logs?

Some developers will already be processing Solana transactions themselves.

In that case, there is no reason to ask the SDK to obtain the transaction again.

If you already have the program log messages, use:

```ts
const events =
  moonz.parseLogs(
    logMessages
  );
```

The SDK will decode the Moonz events found in those logs.

## 🗺️ Add Transaction Context

You can also provide context when parsing logs.

```ts
const events =
  moonz.parseLogs(
    logMessages,
    {
      signature,
      slot,
      blockTime
    }
  );
```

That context is attached to the decoded events.

Now your event can carry both Moonz data and the Solana transaction information surrounding it.

## 🧾 Why Context Matters

Consider an analytics system.

The Moonz event tells you:

What happened.

The transaction context tells you:

Where and when you saw it.

For example:

```ts
for (const event of events) {
  console.log({
    type:
      event.type,

    mint:
      event.mint,

    signature:
      event.signature,

    slot:
      event.slot,

    blockTime:
      event.blockTime
  });
}
```

That is much more useful than an isolated program log.

## 🔢 One Transaction Can Tell More Than One Story

A Solana transaction can contain more than one Moonz event.

That is why the SDK exposes:

```ts
event.eventIndex
```

This is the zero based position of that Moonz event inside the decoded transaction logs.

So two events from the same transaction can still be distinguished.

## 🔐 A Stable Event Identity

For bots, databases and historical processing, Moonz recommends using:

```text
signature + eventIndex
```

as an event identity.

For example:

```ts
const eventId =
  `${event.signature}:${event.eventIndex}`;
```

You might end up with something conceptually like:

```text
5kTx...9Ab:0

5kTx...9Ab:1
```

Same transaction.

Different Moonz events.

That helps prevent duplicate processing without pretending that one transaction can only contain one event.

## 🧠 Build a Simple Transaction Decoder

Here is a compact example.

```ts
import { MoonzSDK } from "@moonz-fun/sdk";

const moonz = new MoonzSDK({
  rpcUrl: "YOUR_SOLANA_RPC"
});

const signature =
  "TRANSACTION_SIGNATURE";

const events =
  await moonz.getTransactionEvents(
    signature
  );

for (const event of events) {
  console.log({
    type:
      event.type,

    category:
      event.category,

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
}
```

You have just turned a Solana transaction signature into structured Moonz history.

## 🕵️ Build a Moonz Transaction Explorer

Now imagine putting that behind a search box.

A user pastes a transaction signature.

Your application decodes the Moonz events.

Then instead of showing:

```text
Program log
Program data
Instruction
Account
```

you can show:

```text
Moonz Buy

Market
BONDING

Quote Asset
SOL

Token
...

User
...

Transaction
...
```

That is the difference between displaying blockchain data and actually understanding it.

## 📚 Build Historical Records

Historical event decoding can also become the foundation of your own data system.

You could store:

* Token launches
* Buys
* Sells
* Migrations
* Pool switch activity
* Fee activity
* Escrow activity

Alongside:

* Signature
* Slot
* Block time
* Event index

From there you can build your own history around Moonz.

{% hint style="success" %}
**The chain already contains the story.**

The SDK helps you read it.
{% endhint %}

## ⚡ Live and Historical Together

The useful part is that you do not need two completely different event models.

Live:

```ts
moonz.watch()
```

Historical:

```ts
moonz.getTransactionEvents(
  signature
);
```

Existing logs:

```ts
moonz.parseLogs(
  logMessages,
  context
);
```

All of them lead back to decoded Moonz events.

That makes it much easier to build systems that understand both the past and the present.

## 🧪 A Useful Pattern

A larger application might work like this:

```text
Historical transactions
        ↓
getTransactionEvents
        ↓
Build initial Moonz history
        ↓
Start live watcher
        ↓
Process new Moonz events
        ↓
Continue from there
```

One system.

One event model.

Two directions through time.

{% hint style="success" %}
**Mission complete.**

You can listen to Moonz now.

And you can look backwards when you need to know how you got here.
{% endhint %}

## 🛰️ Next Stop

We can read Moonz state.

We can discover tokens.

We can understand markets.

We can listen to events.

We can decode history.

Now it is time to learn where Moonz actually lives.

Next stop:

**Protocol Addresses**
