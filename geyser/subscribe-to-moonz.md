# 🎯 Subscribe to Moonz

The connection is open.

Now we need to decide what comes through it.

The Moonz public Geyser gives you two especially useful live surfaces:

```text
Transactions

Accounts
```

They tell you different parts of the same story.

A transaction tells you:

**Something happened.**

An account update tells you:

**Something changed.**

Put them together and you have the foundation of a Moonz indexer.

{% hint style="info" %}
**A strong Moonz data pipeline does not treat transactions and state as the same thing.**

Transactions describe activity.

Accounts describe current program state.
{% endhint %}

## ⚡ Transaction Stream

The transaction subscription is scoped to transactions involving the Moonz Program ID.

```text
DBc9SEQghiJUj52YPqTKk8R4CMRgagBxi2LU1yBbeMpk
```

The public example requests it with:

```js
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
}
```

The label:

```text
moonz_transactions
```

is your subscription filter name.

When a matching transaction update arrives, that label may appear in:

```js
update.filters
```

That makes it easier to understand which subscription filter produced the update.

## 🌙 What Makes a Transaction Moonz?

For the public Moonz stream, the transaction must involve:

```text
MOONZ_PROGRAM_ID
```

through:

```js
account_include: [
  PROGRAM_ID
]
```

Moonz also enforces this scope on the server.

So the public endpoint cannot be widened into:

```text
All Solana transactions

Another program feed

A general validator stream
```

It is deliberately the Moonz pipe.

## 🚫 Transaction Filters You Cannot Broaden

The public Moonz transaction interface currently does not support using features that broaden or alter the permitted scope such as:

```text
Vote transactions

Failed transactions

Signature filtering

Required account filtering

Account exclusions
```

The intended subscription is simple.

```text
Confirmed successful transactions
involving the Moonz program
```

If a request exceeds that boundary, the server can reject it.

## 🧬 Why Transactions Matter

Transactions are where protocol activity happens.

They are the raw material for understanding actions such as:

```text
Token creation

Bonding activity

AMM trading

Migration

Fee activity

PCLS activity
```

But the Geyser transaction itself is not the final developer friendly Moonz event.

Your integration still needs to decode what happened.

That is where the Moonz IDL and canonical event definitions come in.

We will do that shortly.

## 🌙 Account Stream

The second major surface is the account subscription.

The working public request is:

```js
accounts: {
  moonz_accounts: {
    owner: [
      PROGRAM_ID
    ],

    account: [],
    filters: []
  }
}
```

This asks for accounts directly owned by the Moonz program.

Again:

```text
moonz_accounts
```

is the subscription label.

## 🛰️ Account Ownership Is the Boundary

This distinction matters.

The account stream is based on runtime account ownership.

For example:

```text
Launch State
    ↓
Owned by Moonz Program
    ↓
Eligible for Moonz account stream
```

But:

```text
SPL Token Vault
    ↓
Owned by SPL Token Program
    ↓
Not part of Moonz owner stream
```

The vault may be an important part of Moonz.

That does not make the account itself owned by the Moonz program.

## 🔍 What Account Updates Give You

A Moonz account update can provide information including:

```text
Slot

Subscription filter

Account public key

Runtime owner

Raw account data

write_version
```

The raw account data is where the interesting part lives.

Using the Moonz IDL, your indexer can decode that byte data into an actual Moonz account structure.

Instead of storing:

```text
A mysterious byte array changed
```

you can understand:

```text
This is Launch State

This mint is bonding

This amount has been sold

This is the active quote asset

This token entered switching
```

That is where an account stream becomes protocol data.

## ⚠️ write_version Is Not Ordering

The Moonz bridge currently exposes:

```text
write_version = 0
```

Do not use it as a validator native ordering field.

The current upstream account source does not provide native validator `write_version` semantics.

So:

```text
write_version
≠
reliable native write order
```

Your ingestion design must not depend on it.

## 🧠 Transactions Versus Accounts

This is the distinction to remember.

```text
TRANSACTION

What happened?
Who called the program?
What instruction executed?
What events were emitted?

        versus

ACCOUNT

What does the Moonz state
look like after activity?
```

Neither replaces the other.

They complement each other.

## 🌊 Use Both Together

A practical Moonz indexer can subscribe to both in one request.

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

Now one stream can carry both activity and state.

## 🧭 Route Updates by Type

Your consumer can split the incoming stream immediately.

```js
stream.on(
  "data",
  update => {
    if (update?.transaction) {
      handleTransaction(
        update.transaction
      );

      return;
    }

    if (update?.account) {
      handleAccount(
        update.account
      );

      return;
    }

    if (update?.ping) {
      handleHeartbeat(
        update.ping
      );

      return;
    }

    if (update?.pong) {
      handlePong(
        update.pong
      );
    }
  }
);
```

This creates a clean ingestion boundary.

The gRPC listener receives data.

Dedicated processors decide what to do with it.

## ⚡ Transaction Processor

A simple transaction processor might begin like this:

```js
function handleTransaction(
  update
) {
  console.log({
    slot:
      update.slot,

    signature:
      update.transaction
        ?.signature
  });
}
```

Later we can expand this into:

```text
Transaction bytes
        ↓
Moonz instructions
        ↓
Program logs
        ↓
Canonical events
        ↓
Structured records
```

That is how raw chain activity becomes an integration friendly event history.

## 🌙 Account Processor

The account processor starts with the raw account payload.

```js
function handleAccount(
  update
) {
  console.log({
    slot:
      update.slot,

    pubkey:
      update.account
        ?.pubkey,

    owner:
      update.account
        ?.owner,

    data:
      update.account
        ?.data
  });
}
```

Then the decoder determines which Moonz account structure those bytes represent.

The result can become current state in your database.

## 🏗️ The Indexer Pattern

The architecture now starts taking shape.

```text
             MOONZ GEYSER
                  |
          confirmed stream
                  |
        ┌─────────┴─────────┐
        ↓                   ↓
 TRANSACTIONS            ACCOUNTS
        |                   |
        ↓                   ↓
Decode actions          Decode state
        |                   |
        └─────────┬─────────┘
                  ↓
              DATABASE
                  ↓
          YOUR OWN API
                  ↓
             YOUR PRODUCT
```

The transaction side can build history.

The account side can maintain current state.

## 🧪 Example: A Buy Happens

Imagine a Moonz bonding buy.

The transaction stream can tell your processor:

```text
A Moonz transaction happened

A buy instruction executed

A trade event was emitted
```

The account stream can then tell your state processor that Launch State changed.

For example, the new state may reflect updated:

```text
tokens_sold

sol_collected

last_trade_ts
```

The transaction explains the action.

The account explains the resulting state.

That is exactly why using both is powerful.

## 🚀 Example: Bonding Completes

Now imagine the token reaches the end of bonding.

Your transaction processor may observe protocol activity associated with that transition.

Your account processor may then observe the Launch State moving into its new lifecycle phase.

Your database can update both:

```text
Historical event record

and

Current token state
```

Now your frontend does not have to reconstruct the entire chain every time somebody opens the page.

## 🔄 Example: PCLS

The same model becomes especially useful during PCLS.

Transactions can represent the sequence of actions.

Accounts can represent the current switch state.

Conceptually:

```text
Switch transaction
      ↓
Store event

Launch State update
      ↓
Store current state

Next transaction
      ↓
Store event

Next account update
      ↓
Update current state
```

Now an integration can preserve both the history and the latest protocol state.

## 🎯 What About Filtering One Moonz Token?

The public account stream does not currently support individual account address filters.

The public transaction stream also does not provide arbitrary signature or required account filtering.

That means the intended model is:

```text
Consume the permitted Moonz stream
        ↓
Decode it
        ↓
Apply your own application filters
```

For example, once decoded:

```js
if (
  decoded.mint ===
  WATCHED_MINT
) {
  processToken(
    decoded
  );
}
```

The server protects the protocol scope.

Your application handles product specific filtering.

## 📚 Current Account Filter Limits

The public account interface currently does not support:

```text
Individual account addresses

Account data filters

Account data slices
```

Do not design an integration that assumes those Yellowstone features are available simply because they exist in the protobuf definitions.

The Moonz integration specification is the authority.

## ✅ One Commitment Model

Both streams operate at:

```text
CONFIRMED
```

That gives the transaction and account surfaces a consistent commitment model.

Your database should record the slot supplied with updates and treat the stream according to the Moonz confirmed interface.

## 🧾 Keep the Filter Name

Each update can contain:

```js
update.filters
```

Do not throw that information away without considering it.

If your system eventually has multiple processing paths, filter labels make incoming updates easier to route and debug.

For example:

```text
moonz_transactions

moonz_accounts
```

are much easier to reason about in logs than an anonymous stream.

## 🧯 Do Not Perform Heavy Work Inside the Listener

Production consumers should keep the gRPC data handler lightweight.

Avoid turning this:

```js
stream.on(
  "data",
  update => {
    // receive
  }
);
```

into a giant synchronous processing function.

A better pattern is:

```text
Receive update
      ↓
Validate shape
      ↓
Queue or dispatch
      ↓
Decode
      ↓
Store
      ↓
Publish downstream
```

That keeps your stream consumer responsive even when Moonz activity increases.

## 🌌 What We Have Now

At this point you can:

```text
Connect to Moonz Geyser

Subscribe to Moonz transactions

Subscribe to Moonz program accounts

Receive heartbeats

Route incoming updates

Separate activity from state
```

But most of the payload is still raw blockchain data.

That is our next problem.

{% hint style="success" %}
**The pipe is no longer just open.**

You now know which parts of Moonz should flow through it and why each stream exists.
{% endhint %}

## 🧬 Next Stop

We have the messages.

Now we need to understand what is inside them.

Next stop:

**Understand the Messages**
