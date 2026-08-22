# 🧬 Understand the Messages

The stream is running.

Data is arriving.

Now comes the useful part.

Turning raw Solana messages into Moonz.

The Moonz Geyser uses Yellowstone compatible protobuf messages, so each update arrives as a `SubscribeUpdate`.

Inside that update is one active message type.

For the Moonz public stream, the messages you will care about most are:

```text
transaction

account

ping

pong
```

{% hint style="info" %}
**The Geyser transports blockchain data.**

Your decoder turns that data into Moonz concepts.
{% endhint %}

## 📦 The SubscribeUpdate Envelope

At the protocol level, an incoming update contains:

```proto
message SubscribeUpdate {
  repeated string filters = 1;

  oneof update_oneof {
    SubscribeUpdateAccount account = 2;
    SubscribeUpdateSlot slot = 3;
    SubscribeUpdateTransaction transaction = 4;
    SubscribeUpdateTransactionStatus transaction_status = 10;
    SubscribeUpdateBlock block = 5;
    SubscribeUpdatePing ping = 6;
    SubscribeUpdatePong pong = 9;
    SubscribeUpdateBlockMeta block_meta = 7;
    SubscribeUpdateEntry entry = 8;
  }

  google.protobuf.Timestamp created_at = 11;
}
```

The `oneof` is important.

An individual update represents one of those message variants.

Your consumer should inspect the update type before attempting to decode its contents.

## 🧭 Route Before You Decode

A simple router might look like:

```js
function handleUpdate(update) {
  if (update?.transaction) {
    handleTransaction(
      update.transaction,
      update.filters
    );

    return;
  }

  if (update?.account) {
    handleAccount(
      update.account,
      update.filters
    );

    return;
  }

  if (update?.ping) {
    handleHeartbeat();
    return;
  }

  if (update?.pong) {
    handlePong(
      update.pong
    );
  }
}
```

This keeps transport handling separate from Moonz decoding.

That separation becomes useful once your indexer grows.

# ⚡ Transaction Messages

A transaction update has this structure:

```proto
message SubscribeUpdateTransaction {
  SubscribeUpdateTransactionInfo transaction = 1;
  uint64 slot = 2;
}
```

The transaction information contains:

```proto
message SubscribeUpdateTransactionInfo {
  bytes signature = 1;
  bool is_vote = 2;
  solana.storage.ConfirmedBlock.Transaction transaction = 3;
  solana.storage.ConfirmedBlock.TransactionStatusMeta meta = 4;
  uint64 index = 5;
}
```

That gives your processor several useful pieces of information.

```text
slot

signature

transaction

meta

transaction index
```

## 🧾 The Transaction Itself

The underlying Solana transaction contains:

```proto
message Transaction {
  repeated bytes signatures = 1;
  Message message = 2;
}
```

And the message contains:

```proto
message Message {
  MessageHeader header = 1;
  repeated bytes account_keys = 2;
  bytes recent_blockhash = 3;
  repeated CompiledInstruction instructions = 4;
  bool versioned = 5;
  repeated MessageAddressTableLookup address_table_lookups = 6;
}
```

This is the raw transaction structure your instruction decoder works from.

## 🛰️ Compiled Instructions

A compiled instruction contains:

```proto
message CompiledInstruction {
  uint32 program_id_index = 1;
  bytes accounts = 2;
  bytes data = 3;
}
```

The `program_id_index` points into the transaction account key list.

The instruction `data` contains the encoded instruction payload.

For Moonz instruction indexing, the first job is to determine whether the instruction belongs to:

```text
DBc9SEQghiJUj52YPqTKk8R4CMRgagBxi2LU1yBbeMpk
```

Then its discriminator can be matched against the verified Moonz IDL.

## 🧬 Canonical Instruction Names

Moonz publishes the canonical names used by the verified program.

They are:

```text
abort_pool_switch_route_invalid

amm_buy_usdc

amm_sell_usdc

begin_pool_switch

buy

cancel_initialized_launch

cancel_pool_switch

claim_fees

complete_pool_switch

dev_buy_start_curve_from_escrow

execute_pool_switch_swap

finalize_mint_authorities

fund_launch_escrow

initialize_launch

initialize_metadata

refund_launch_escrow

sell

settle_escrow_to_platform
```

If your integration claims to expose canonical Moonz instruction names, preserve those names exactly.

Do not silently rename them into your own protocol vocabulary.

Your application can create friendly display labels separately.

## 🔎 Example

Suppose your decoder sees instruction data whose discriminator matches the canonical Moonz `buy` instruction.

The verified discriminator is:

```text
102, 6, 61, 18, 1, 218, 235, 234
```

That instruction takes:

```text
wsol_in

min_tokens_out
```

Your indexer can therefore identify the instruction as:

```text
buy
```

before decoding its arguments and accounts.

# 📜 Transaction Metadata

The transaction metadata is just as important as the instruction message.

Moonz receives the Yellowstone compatible `TransactionStatusMeta` structure.

Important fields include:

```text
err

fee

pre_balances

post_balances

inner_instructions

log_messages

pre_token_balances

post_token_balances

loaded_writable_addresses

loaded_readonly_addresses

return_data

compute_units_consumed
```

For Moonz event indexing, one field is particularly important:

```text
log_messages
```

## 🌙 Moonz Events Live in the Logs

Anchor events emitted by the Moonz program are represented through the transaction logs.

That means a common event indexing path is:

```text
Geyser transaction
        ↓
transaction.meta
        ↓
log_messages
        ↓
Moonz event decoder
        ↓
Canonical event
```

The public developer toolkit includes the verified Moonz IDL and the canonical event discriminators required to understand those events.

## 🌟 Canonical Moonz Events

For complete Moonz lifecycle coverage, the integration guide recommends indexing all canonical events emitted by the program.

The canonical names are:

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

These are protocol names.

Preserve them exactly in any schema that claims to expose canonical Moonz events.

## 🧬 Event Discriminators

Each canonical event has its own discriminator.

For example:

```text
BuyEvent

103, 244, 82, 31, 44, 245, 119, 119
```

And:

```text
AmmBuyEvent

49, 113, 75, 208, 243, 103, 62, 31
```

And:

```text
PoolSwitchStartedEvent

12, 17, 8, 74, 0, 167, 14, 157
```

Your decoder should match against the canonical IDL rather than guessing an event from the transaction shape.

## ⚡ Trade Events

Moonz exposes separate events for bonding and AMM trading.

Bonding:

```text
BuyEvent

SellEvent
```

AMM:

```text
AmmBuyEvent

AmmSellEvent
```

These trade events expose fields including:

```text
mint

user

quote_asset

input_amount

input_mint

output_amount

output_mint

quote_amount

token_amount

trade_fee

creator_fee

platform_fee

lp_fee

tokens_sold_total

quote_collected_total

timestamp
```

This means an indexer does not need to infer every trade value by comparing balances.

The canonical event already exposes the protocol level trade information.

## 🔄 PCLS Events

PCLS has its own event sequence.

```text
PoolSwitchStartedEvent

PoolSwitchSwapExecutedEvent

PoolSwitchCompletedEvent

PoolSwitchCancelledEvent
```

These allow an integration to represent the switch lifecycle rather than treating it as an unexplained series of transactions.

For example, `PoolSwitchStartedEvent` exposes:

```text
mint

creator

from_asset

to_asset

amount_in

min_amount_out

switch_fee_lamports
```

`PoolSwitchSwapExecutedEvent` then exposes:

```text
mint

executor

from_asset

to_asset

amount_in

amount_out

source_remaining

destination_balance
```

This is valuable for explorers, analytics platforms and protocol monitoring.

## 🚀 Migration

Moonz exposes:

```text
MigratedEvent
```

with the launch mint.

This gives indexers a canonical protocol event for the migration point rather than forcing them to infer the transition solely from later state.

## 👛 Fee Claims

Moonz exposes:

```text
ClaimFeesEvent
```

with:

```text
creator

fee_mint

amount
```

Notice that this event does not contain the launch mint.

Your database design should not invent a mint field where the canonical event does not provide one.

Context can be stored separately when your ingestion system can establish it safely.

## 🏗️ Creation Events

The creation flow includes events such as:

```text
CreatedTxn

LaunchEscrowFundedEvent

LaunchEscrowRefundedEvent
```

These provide useful lifecycle information before ordinary trading begins.

That is why the Moonz indexing guide recommends indexing the complete canonical event set rather than only buy and sell activity.

# 🌙 Account Messages

Transactions explain activity.

Account updates represent state.

The account update structure is:

```proto
message SubscribeUpdateAccount {
  SubscribeUpdateAccountInfo account = 1;
  uint64 slot = 2;
  bool is_startup = 3;
}
```

The account information contains:

```proto
message SubscribeUpdateAccountInfo {
  bytes pubkey = 1;
  uint64 lamports = 2;
  bytes owner = 3;
  bool executable = 4;
  uint64 rent_epoch = 5;
  bytes data = 6;
  uint64 write_version = 7;
  optional bytes txn_signature = 8;
}
```

## 🧠 Fields Worth Keeping

A useful account record should consider preserving:

```text
slot

pubkey

owner

lamports

executable

rent_epoch

raw account data

transaction signature when present

is_startup
```

Even if your application only needs a subset immediately, retaining enough provenance makes debugging and replay logic easier.

## 🧬 Decode the Account Data

The account `data` field is raw program account data.

For Moonz program owned accounts, use the verified Moonz Anchor IDL to determine the account type and decode its fields.

Conceptually:

```text
account.data
      ↓
Anchor account discriminator
      ↓
Moonz IDL
      ↓
Decoded account
      ↓
Application state
```

Do not identify a Moonz account solely by its byte length if a discriminator based decoder is available.

## 🛰️ Verify the Owner

The stream is already restricted to Moonz program owned accounts, but a robust decoder should still understand the ownership rule.

For this stream, the expected program owner is:

```text
DBc9SEQghiJUj52YPqTKk8R4CMRgagBxi2LU1yBbeMpk
```

This is separate from the deterministic PDA relationships described in the Moonz SDK section.

Ownership tells you which program owns the account.

The PDA relationship tells you whether the address is the expected deterministic Moonz address.

## 🧾 is_startup

Account messages include:

```text
is_startup
```

Your storage layer should preserve it if your application needs to distinguish initial stream population behaviour from later live updates.

Do not confuse it with a Moonz launch phase.

It is part of the Geyser account update message.

## 🔗 txn_signature

An account update can optionally contain:

```text
txn_signature
```

When present, that can give your ingestion system useful transaction context for the state update.

Because the field is optional, your database logic must not require it to exist for every account message.

## ⚠️ write_version Again

The protocol structure includes:

```text
write_version
```

but the Moonz bridge currently reports:

```text
0
```

because native validator write version information is not available from the current upstream source.

Therefore:

```text
Do not sort Moonz account writes
using write_version
```

We will build safer ingestion rules in the production reliability page.

# 🔢 Preserve Raw Integer Precision

Several fields carried through these messages are `u64` or `i64`.

That includes values such as:

```text
amounts

fees

balances

slots

timestamps

token quantities
```

The public Node example configures protobuf loading with:

```js
longs: String
```

Keep that precision.

Do not automatically convert every blockchain integer to a JavaScript `number`.

For database storage, use a representation capable of preserving the complete integer value.

## 💰 Token Balances

Transaction metadata can also contain:

```text
pre_token_balances

post_token_balances
```

Each token balance can expose:

```text
account_index

mint

ui_token_amount

owner

program_id
```

The raw token amount is available as a string inside `ui_token_amount.amount`.

For protocol accounting, raw integer amounts are generally safer than floating point UI values.

## 🧠 Instructions and Events Are Not the Same

This distinction matters.

An instruction answers:

```text
What operation was invoked?
```

An event answers:

```text
What canonical protocol information
did the program emit?
```

For example:

```text
Instruction
buy

Event
BuyEvent
```

Do not collapse the two into a single concept in your database unless your application deliberately creates a separate normalized layer.

Keeping the raw canonical identity available makes integrations easier to audit.

## 🗃️ A Useful Storage Model

A serious indexer may eventually maintain separate records such as:

```text
transactions

instructions

events

program_accounts

token_state
```

Then link them using information such as:

```text
signature

slot

transaction index

account address

mint

event position
```

The exact schema is yours.

The important principle is to preserve enough chain identity to trace an application record back to its source.

## 🌌 From Wire Message to Product Data

The whole decoding pipeline now looks like:

```text
MOONZ GEYSER
      |
      ↓
SubscribeUpdate
      |
      ├───────────────┐
      ↓               ↓
Transaction         Account
      |               |
      ↓               ↓
Instructions       Raw data
      |               |
      ↓               ↓
Logs              Moonz IDL
      |               |
      ↓               ↓
Events            State
      |               |
      └───────┬───────┘
              ↓
         Your database
              ↓
           Your API
              ↓
        Your application
```

This is the point where Geyser stops looking like a stream of bytes and starts looking like Moonz.

{% hint style="success" %}
**Do not throw away the raw identity of what you decoded.**

Canonical instruction names, canonical event names, signatures, slots and account addresses make your integration traceable.
{% endhint %}

## 🔁 Next Stop

We can connect.

We can subscribe.

We can understand what arrives.

Now we need to make sure the system survives the real world.

Next stop:

**Production Reliability**
