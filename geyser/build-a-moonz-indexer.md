# 🏗️ Build a Moonz Indexer

We have reached the engine room.

You can connect to the Moonz Geyser.

You can subscribe to transactions and accounts.

You know what the messages contain.

You know how to survive disconnects.

Now we put those pieces together into an indexer.

{% hint style="info" %}
**A Moonz indexer turns the live chain stream into structured data your application can query quickly.**
{% endhint %}

## 🌌 The Architecture

A practical Moonz indexing system can look like this:

```text
                 SOLANA MAINNET
                       |
                       ↓
                 MOONZ PROGRAM
                       |
                       ↓
                  MOONZ GEYSER
                       |
               confirmed gRPC
                       |
                       ↓
                STREAM CONSUMER
                 /           \
                ↓             ↓
        TRANSACTIONS       ACCOUNTS
                |             |
                ↓             ↓
          Event decoder   State decoder
                |             |
                └──────┬──────┘
                       ↓
                   DATABASE
                       |
               ┌───────┴───────┐
               ↓               ↓
          Checkpoints        Current state
               |               |
               └───────┬───────┘
                       ↓
                  YOUR API
                       |
                       ↓
                  YOUR PRODUCT
```

The Geyser is the ingress layer.

It should not become your database, API and application logic all at once.

## 🧱 Separate the Responsibilities

A clean system can have several logical components.

```text
Connection

Routing

Transaction decoding

Event decoding

Account decoding

Storage

Checkpointing

Reconciliation

API
```

They can begin inside one process.

You do not need nine servers.

The important part is keeping the responsibilities conceptually separate.

## 📡 1. Connection

The first responsibility is boring by design.

Connect to:

```text
grpc.moonz.fun:443
```

using TLS and open the Moonz scoped subscription.

The connection layer should concern itself with:

```text
gRPC connection

Subscription request

Heartbeats

Reconnects

Shutdown
```

It should not calculate token market caps or write complicated application logic.

## 🎯 2. Subscription

Subscribe to both supported Moonz surfaces.

```js
stream.write({
  transactions: {
    moonz_transactions: {
      vote: false,
      failed: false,

      account_include: [
        PROGRAM_ID
      ],

      account_exclude: [],
      account_required: []
    }
  },

  accounts: {
    moonz_accounts: {
      owner: [
        PROGRAM_ID
      ],

      account: [],
      filters: []
    }
  },

  commitment:
    "CONFIRMED"
});
```

Now the stream can provide protocol activity and program state.

## 🧭 3. Route Immediately

Do not try to decode every message in the gRPC callback itself.

Route first.

```js
stream.on(
  "data",
  update => {
    if (update?.transaction) {
      transactionQueue.push({
        filters:
          update.filters || [],

        update:
          update.transaction
      });

      return;
    }

    if (update?.account) {
      accountQueue.push({
        filters:
          update.filters || [],

        update:
          update.account
      });

      return;
    }

    if (update?.ping) {
      markHeartbeat();
      return;
    }

    if (update?.pong) {
      markPong(
        update.pong.id
      );
    }
  }
);
```

The exact queue technology is your choice.

For a small service it may simply be an internal work queue.

At greater scale it could become a durable message system.

## ⚡ 4. Process Transactions

A transaction worker receives:

```text
slot

signature

transaction

meta

transaction index
```

Its job is to turn that into useful Moonz history.

Conceptually:

```text
Geyser transaction
      ↓
Normalize signature
      ↓
Store transaction identity
      ↓
Inspect instructions
      ↓
Read log_messages
      ↓
Decode Moonz events
      ↓
Store events
      ↓
Commit
      ↓
Advance checkpoint
```

## 🧬 Use the Canonical Event Decoder

The Moonz SDK already knows how to parse canonical Moonz program logs.

If your Geyser transaction gives you:

```js
update.transaction.meta
  .log_messages
```

those logs are the bridge into the same Moonz event model used elsewhere in your application.

Conceptually:

```ts
const events =
  moonz.parseLogs(
    logMessages,
    {
      signature,
      slot
    }
  );
```

That lets your Geyser pipeline and ordinary Moonz SDK integrations speak the same event language.

{% hint style="success" %}
**Geyser transports the transaction.**

**The Moonz SDK can help turn its logs into Moonz events.**
{% endhint %}

## 🌙 5. Preserve Canonical Events

For full lifecycle coverage, store every canonical Moonz event you care about.

The verified event set currently includes:

```text
AmmBuyEvent

AmmSellEvent

BuyEvent

ClaimFeesEvent

CreatedTxn

LaunchEscrowFundedEvent

LaunchEscrowRefundedEvent

MigratedEvent

PoolSwitchCancelledEvent

PoolSwitchCompletedEvent

PoolSwitchStartedEvent

PoolSwitchSwapExecutedEvent

SellEvent
```

Do not reduce the indexer to only:

```text
buys

sells
```

if your product needs to understand the full Moonz lifecycle.

## 🛰️ 6. Process Program Accounts

The account worker receives program owned account updates.

Its job is different.

```text
Geyser account update
      ↓
Verify owner
      ↓
Read account discriminator
      ↓
Decode with Moonz IDL
      ↓
Identify account
      ↓
Store current state
      ↓
Commit
      ↓
Advance checkpoint
```

Transactions create history.

Account updates maintain state.

## 🌱 Current State Table

For a token oriented product, your current state layer may eventually expose values such as:

```text
mint

creator

phase

quote asset

tokens sold

remaining supply

last trade time

switch state

protocol addresses
```

The exact database schema is yours.

The important thing is that this is a current state projection.

It can be updated when newer confirmed Moonz state arrives.

## 🧾 7. Keep Transaction History Separate

Do not overwrite transaction history when current state changes.

A simple conceptual schema could separate:

```text
moonz_transactions

moonz_events

moonz_accounts

moonz_tokens

ingestion_checkpoints
```

Each table solves a different problem.

## 🗃️ Example Transaction Table

Conceptually:

```sql
CREATE TABLE moonz_transactions (
  signature TEXT PRIMARY KEY,
  slot TEXT NOT NULL,
  transaction_index TEXT,
  observed_at TEXT NOT NULL
);
```

Using text for chain integers is one simple way to avoid accidental JavaScript precision loss when the surrounding application treats those values as strings.

Your database may support a more appropriate exact integer type.

## 🧬 Example Event Table

A Moonz event table might look like:

```sql
CREATE TABLE moonz_events (
  signature TEXT NOT NULL,
  event_index INTEGER NOT NULL,
  slot TEXT NOT NULL,
  canonical_name TEXT NOT NULL,
  mint TEXT,
  payload_json TEXT NOT NULL,

  PRIMARY KEY (
    signature,
    event_index
  )
);
```

The key idea is:

```text
signature
+
event_index
```

rather than deduplicating events based on their business fields.

## 🌙 Example Current Token Table

A current state projection might look like:

```sql
CREATE TABLE moonz_tokens (
  mint TEXT PRIMARY KEY,
  launch_state TEXT NOT NULL,
  creator TEXT,
  phase TEXT,
  quote_asset TEXT,
  slot TEXT NOT NULL,
  state_json TEXT NOT NULL
);
```

This is your fast application view.

The raw canonical data can remain available elsewhere.

## 🛰️ Example Account Table

For program account provenance:

```sql
CREATE TABLE moonz_accounts (
  address TEXT PRIMARY KEY,
  owner TEXT NOT NULL,
  slot TEXT NOT NULL,
  account_type TEXT,
  data_json TEXT,
  raw_data_base64 TEXT
);
```

Whether you retain raw account bytes permanently depends on your product and storage requirements.

But retaining sufficient provenance is extremely useful during integration development.

## 📍 8. Checkpoints

Your checkpoint table can record the progress of each processor.

For example:

```sql
CREATE TABLE ingestion_checkpoints (
  consumer TEXT PRIMARY KEY,
  slot TEXT,
  signature TEXT,
  updated_at TEXT NOT NULL
);
```

Possible consumer names:

```text
moonz_transactions

moonz_accounts
```

Update a checkpoint only after the related database transaction succeeds.

## 🔐 Use Database Transactions

For event ingestion, aim for this behaviour:

```text
Begin database transaction

Insert transaction

Insert decoded events

Update checkpoint

Commit
```

If something fails:

```text
Rollback
```

Now your database does not claim progress without storing the corresponding Moonz data.

## ♻️ Make Inserts Idempotent

Suppose the same transaction is observed twice.

Your indexer should not create two trades simply because it saw the transaction twice.

For example:

```sql
INSERT INTO moonz_transactions (
  signature,
  slot,
  transaction_index,
  observed_at
)
VALUES (?, ?, ?, ?)
ON CONFLICT(signature)
DO NOTHING;
```

The exact syntax varies by database.

The principle does not.

Repeated observation should not create repeated history.

## 🧬 Events Need Their Own Identity

One Solana transaction can contain more than one Moonz event.

So using only the transaction signature as the event primary key is not enough.

Use:

```text
signature

event_index
```

together.

That allows:

```text
Signature ABC
Event 0

Signature ABC
Event 1

Signature ABC
Event 2
```

to remain distinct.

## 🌐 9. Build an Internal API

Once the indexer is storing structured Moonz data, your application no longer needs to perform expensive chain reconstruction for every request.

Your API can provide endpoints such as:

```text
Token by mint

Recent Moonz tokens

Recent trades

Token trades

Current phase

Current quote asset

PCLS history

Migration history
```

These are examples of your own product API.

They are not Moonz public Geyser RPCs.

That distinction matters.

## 📊 Example Product Flow

A token page could work like:

```text
Browser
   ↓
Your API
   ↓
moonz_tokens
   +
moonz_events
   ↓
Fast response
```

Meanwhile the indexer keeps those tables current from Geyser.

The browser does not need its own Geyser connection.

## 🚫 Do Not Expose Geyser Directly to Every Browser

For most products, this would be a poor architecture:

```text
10,000 browsers
      ↓
10,000 direct Geyser connections
```

Apart from service limits, it pushes infrastructure concerns into every client.

Prefer:

```text
Moonz Geyser
      ↓
Your indexer
      ↓
Your API or WebSocket service
      ↓
Your users
```

Your backend consumes the protocol stream.

Your product distributes the data it actually needs.

## 🌊 One Stream, Many Products

One Moonz ingestion service can feed several internal consumers.

```text
                 Moonz Geyser
                      |
                      ↓
                  Indexer
                      |
          ┌───────────┼───────────┐
          ↓           ↓           ↓
       API        WebSocket    Analytics
          |           |           |
          ↓           ↓           ↓
       Website       Bot       Dashboard
```

This is more efficient than each product opening its own protocol connection.

## 🔭 10. Reconcile With the Moonz SDK

The Geyser is live.

The Moonz SDK can read canonical state.

That makes the SDK extremely useful after a reconnect.

For a known mint:

```ts
const token =
  await moonz.getToken(
    mint
  );
```

Now compare that canonical read with the state stored in your database.

Conceptually:

```text
Database state
      +
Moonz SDK state
      ↓
Compare
      ↓
Same?
  ├── YES
  |     ↓
  |   Continue
  |
  └── NO
        ↓
     Reconcile
        ↓
     Continue
```

This repairs current state after gaps.

## ⚠️ Reconciliation Does Not Recreate History

This distinction is critical.

Suppose your indexer was offline while three trades happened.

A later SDK state read can tell you the current Moonz state.

It cannot magically prove the complete sequence of every transaction you missed.

```text
Current state recovery
≠
Historical transaction recovery
```

If complete historical recovery is a requirement, your architecture also needs an appropriate historical Solana source or retained transaction history.

The current public Moonz Geyser does not provide historical replay.

## 🧯 11. Quarantine Decode Failures

Do not simply print an error and discard unknown data.

Create a safe failure path.

```text
Decode failed
      ↓
Store source identity
      ↓
Store safe raw context
      ↓
Record decoder version
      ↓
Alert
      ↓
Continue stream
```

This allows you to inspect the problem later without stopping the entire pipeline.

## 🧪 Decoder Versioning

Your integration will evolve.

Consider recording which decoder version produced an indexed record.

For example:

```text
moonz_decoder_version
0.1.0
```

If your interpretation logic changes later, you can identify which records were created under older decoder behaviour.

## 🛰️ 12. Verify Before You Trust

A serious indexer should use the same integrity principles introduced in the Moonz SDK section.

Do not trust a string merely because somebody called it a Moonz mint.

Verify relationships.

For example:

```text
Program ownership

Launch State PDA

Stored vault addresses

Derived vault addresses

Token account ownership

Vault authority

Vault mint
```

The Moonz SDK exposes these checks through its integrity model.

## 🌙 Keep Canonical and Derived Data Separate

Suppose your product calculates:

```text
Market cap

Price change

Volume

Holder count

Trending score
```

Those are useful product fields.

But they are not the same thing as canonical Moonz program state.

A strong schema keeps the distinction clear.

```text
Canonical chain data
        ↓
Stored faithfully
        ↓
Derived product data
        ↓
Calculated separately
```

That makes future recalculation possible.

## 📈 13. Build Derived Data

Once the canonical layer is reliable, you can start building higher level datasets.

Examples:

```text
Candles

Volume

Trade counts

Unique traders

Bonding progress history

PCLS history

Migration timelines

Creator activity

Recent buys
```

This is where the indexer becomes a data platform rather than simply a transaction archive.

## 🕯️ Candles

A charting pipeline might consume decoded trades and aggregate them into intervals.

Conceptually:

```text
Trade events
      ↓
Normalize price
      ↓
Bucket by time
      ↓
Open

High

Low

Close

Volume
```

The canonical trade events provide the raw protocol information.

Your indexing layer creates the chart representation.

## 👑 Holders Need Their Own Source

Remember the account ownership limitation.

The public Moonz owner stream contains accounts owned directly by the Moonz program.

SPL Token accounts are owned by the Token Program.

So a complete holder index cannot be derived simply by assuming every token account arrives through the Moonz account subscription.

Holder tracking requires the appropriate Solana token account data source.

We will explain the Moonz holder model later in the guide.

## 💱 Quote Assets Matter

When storing market information, preserve the quote asset.

A price without its quote asset is ambiguous.

```text
0.0004 SOL

is not the same market value as

0.0004 USDC
```

Moonz supports markets quoted in SOL or USDC.

Your schema should retain that identity.

## 🔄 PCLS Is a Sequence

Do not model PCLS as merely:

```text
quote_asset changed
```

The canonical events give you a lifecycle.

```text
PoolSwitchStartedEvent
          ↓
PoolSwitchSwapExecutedEvent
          ↓
PoolSwitchCompletedEvent
```

or:

```text
PoolSwitchStartedEvent
          ↓
PoolSwitchCancelledEvent
```

An indexer can preserve that sequence and expose it cleanly to products.

## 🚀 Migration Is Also History

When:

```text
MigratedEvent
```

arrives, store it as history.

Then let the current state projection represent the new phase.

Again:

```text
Event history
and
current state
```

serve different purposes.

## 🔍 A Useful Debug Record

When investigating an integration issue, you want to be able to answer:

```text
Which slot?

Which transaction?

Which account?

Which filter?

Which canonical event?

Which decoder version?

Which database write?
```

Designing for that visibility early will save a great deal of time later.

## 📡 Example Processing Skeleton

At a high level, your service might resemble:

```js
async function onTransaction(
  update
) {
  const slot =
    update.slot;

  const info =
    update.transaction;

  const signature =
    normalizeSignature(
      info.signature
    );

  const logs =
    info.meta
      ?.log_messages || [];

  const events =
    decodeMoonzEvents({
      signature,
      slot,
      logs
    });

  await storeTransactionAndEvents({
    signature,
    slot,
    transactionIndex:
      info.index,
    events
  });
}

async function onAccount(
  update
) {
  const decoded =
    decodeMoonzAccount(
      update.account
    );

  await storeCurrentState({
    slot:
      update.slot,

    decoded
  });
}
```

The helper names here describe your application architecture.

They are not public Moonz SDK method names unless explicitly documented elsewhere.

## 🧭 The Main Loop

Your final service lifecycle becomes:

```text
START
  ↓
Load configuration
  ↓
Open database
  ↓
Load checkpoints
  ↓
GetVersion
  ↓
Open Geyser
  ↓
Subscribe
  ↓
Route updates
  ↓
┌─────────────────────────┐
│                         │
↓                         ↓
Transactions           Accounts
↓                         ↓
Decode events          Decode state
↓                         ↓
Store history          Store current state
│                         │
└─────────────┬───────────┘
              ↓
         Checkpoint
              ↓
       Monitor health
              ↓
        Stream fails?
        /          \
      no            yes
      |              |
      ↓              ↓
   continue        backoff
                     ↓
                  reconnect
                     ↓
                  reconcile
                     ↓
                  continue
```

That is a Moonz indexer.

## 🧑‍🚀 Start Small

You do not need to build the entire architecture on day one.

A sensible progression is:

```text
1
Connect and print updates

2
Decode Moonz events

3
Store transactions and events

4
Decode program accounts

5
Maintain current token state

6
Add checkpoints

7
Add reconnect logic

8
Add reconciliation

9
Add your API

10
Add derived analytics
```

Each stage gives you something testable.

## 🌌 What You Can Build From Here

Once this foundation exists, the possibilities expand quickly.

```text
Moonz explorer

DEX integration

Trading bot backend

Analytics service

Charting platform

Token discovery service

PCLS monitor

Migration tracker

Creator dashboard

Market data service

Notification engine
```

They can all begin with the same reliable ingestion layer.

{% hint style="success" %}
**You are no longer watching Moonz.**

You are indexing it.
{% endhint %}

## 🌙 Where Next?

You now know how external infrastructure can consume Moonz continuously.

The next part of the guide changes perspective.

Instead of asking how to receive Moonz data, we are going to understand what the protocol itself is doing.

Next section:

**Understanding Moonz**
