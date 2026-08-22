# 🧾 Launch Sequence

A Moonz launch is not one transaction.

`createToken()` coordinates a sequence of distinct stages that take the reserved mint from preparation to `LIVE`.

The successful result records six transaction signatures:

```text
fund

initialize

metadata

finalize

devBuy

settle
```

Each belongs to a different part of the launch.

{% hint style="info" %}
Only the escrow funding transaction is passed to the creator through the public `signTransaction` callback.

The later launch operations are coordinated through the Moonz launch API and return their resulting signatures.
{% endhint %}

## 🌌 The Full Sequence

At a high level:

```text
Reserve mint
      ↓
Publish image
      ↓
Publish metadata
      ↓
Quote Dev Buy
      ↓
tx0
Fund launch escrow
      ↓
Confirm tx0
      ↓
tx1
Initialize launch
      ↓
tx2
Create metadata
      ↓
tx3
Finalize mint
      ↓
Execute Dev Buy
      ↓
Settle
      ↓
LIVE
```

## 🧭 The Six Final Signatures

A successful `CreateTokenResult` contains:

```ts
signatures: {
  fund: string;
  initialize: string;
  metadata: string;
  finalize: string;
  devBuy: string;
  settle: string;
}
```

The mapping is:

```text
fund
    ↓
tx0 escrow funding

initialize
    ↓
tx1 launch initialization

metadata
    ↓
tx2 metadata creation

finalize
    ↓
tx3 mint finalization

devBuy
    ↓
Dev Buy execution

settle
    ↓
Launch settlement
```

## 💰 Stage One: tx0

The first transaction stage is:

```text
POST /launch/{mint}/tx0
```

The request includes:

```text
creator

devBuyLamports

devBuyMinTokensOut
```

Moonz returns an encoded Solana transaction plus:

```text
blockhash

lastValidBlockHeight
```

## 👛 tx0 Is the Creator Signed Stage

The SDK reconstructs the returned transaction and emits:

```text
AWAITING_CREATOR_SIGNATURE
```

Then:

```ts
input.signTransaction(
  transaction
)
```

is called.

This is the creator wallet authorization stage.

## 📡 Submit tx0

After the creator signs, the SDK sends the transaction through its Solana connection.

```ts
sendRawTransaction(
  signed.serialize(),
  {
    skipPreflight: false,
    preflightCommitment:
      commitment
  }
)
```

The resulting signature becomes:

```ts
result.signatures.fund
```

## ⛓️ Confirm tx0

The SDK then confirms the escrow funding transaction using:

```text
fund signature

blockhash

lastValidBlockHeight
```

The progress state becomes:

```text
CONFIRMING_ESCROW
```

## ✅ Report Funding Confirmation

After Solana confirmation, the SDK calls:

```text
POST /launch/{mint}/tx0/confirm
```

with:

```text
txSig

creator
```

This lets the Moonz creation flow continue beyond the creator authorized funding stage.

## 🚀 Stage Two: tx1

The next launch operation is:

```text
POST /launch/{mint}/tx1
```

The SDK supplies:

```text
params.creator

params.name

params.symbol
```

Immediately before this request, progress becomes:

```text
INITIALIZING_LAUNCH
```

## 🛰️ Initialization Signature

The `tx1` response contains a signature.

That becomes:

```ts
result.signatures.initialize
```

The public SDK does not pass `tx1` to `input.signTransaction`.

It receives the resulting signature from the Moonz launch flow.

## 🧾 Stage Three: tx2

Next comes:

```text
POST /launch/{mint}/tx2
```

The request includes:

```text
name

symbol
```

The corresponding progress step is:

```text
CREATING_METADATA
```

## 🔗 Metadata Signature

The returned `tx2` signature becomes:

```ts
result.signatures.metadata
```

This signature is separate from:

```ts
result.metadataUri
```

They represent different things.

```text
metadataUri
    ↓
Published metadata resource

signatures.metadata
    ↓
Launch metadata transaction signature
```

## 🔒 Stage Four: tx3

The next operation is:

```text
POST /launch/{mint}/tx3
```

The progress state before this stage is:

```text
FINALIZING_MINT
```

The API returns another transaction signature.

## 🛡️ Finalization Signature

That signature becomes:

```ts
result.signatures.finalize
```

At this point the creation sequence has moved through launch initialization, metadata creation and mint finalization.

## 💸 Stage Five: Dev Buy

After `tx3`, the SDK emits:

```text
EXECUTING_DEV_BUY
```

Then it requests:

```text
POST /launch/{mint}/devbuy-escrow
```

with:

```text
minTokensOut
```

from the earlier Dev Buy quote.

## 🌙 Dev Buy Signature

The returned transaction signature becomes:

```ts
result.signatures.devBuy
```

This is not the same as:

```ts
result.signatures.fund
```

The two signatures represent different stages.

```text
fund
    ↓
Creator funds launch escrow

devBuy
    ↓
Moonz executes Dev Buy
from launch escrow
```

## 🌕 Stage Six: Settlement

After Dev Buy execution, the SDK emits:

```text
SETTLING
```

Then it requests:

```text
POST /launch/{mint}/settle
```

The returned signature becomes:

```ts
result.signatures.settle
```

## ✅ LIVE

After settlement succeeds, the SDK emits:

```text
LIVE
```

and returns:

```ts
status:
  "LIVE"
```

That is the successful terminal state of `CreateTokenResult`.

## 📦 Complete Result

A successful launch returns:

```ts
{
  mint,

  imageUri,

  metadataUri,

  devBuyLamports,

  expectedTokensOut,

  minTokensOut,

  signatures: {
    fund,
    initialize,
    metadata,
    finalize,
    devBuy,
    settle
  },

  status:
    "LIVE"
}
```

## 🔎 Inspect the Launch Trail

An application can retain the complete signature set:

```ts
const result =
  await moonz.createToken({
    creator:
      wallet.publicKey,

    name:
      "Example Token",

    symbol:
      "EXAMPLE",

    image:
      selectedFile,

    devBuySol:
      "0.5",

    signTransaction:
      wallet.signTransaction
  });

console.log(
  result.signatures
);
```

That provides a clear launch trail for later diagnostics or explorer links.

## 🗺️ Signature Map

A useful mental model is:

```text
                 MOONZ CREATE TOKEN

                       mint
                        |
                        ↓
                 image + metadata
                        |
                        ↓
                 Dev Buy quote
                        |
                        ↓
                     tx0
                        |
                 creator signs
                        |
                        ↓
                      fund
                        |
                        ↓
                     tx1
                        |
                        ↓
                   initialize
                        |
                        ↓
                     tx2
                        |
                        ↓
                    metadata
                        |
                        ↓
                     tx3
                        |
                        ↓
                    finalize
                        |
                        ↓
                devbuy-escrow
                        |
                        ↓
                     devBuy
                        |
                        ↓
                    settle
                        |
                        ↓
                     settle
                        |
                        ↓
                      LIVE
```

## 👛 One Public Signing Callback

The creation input contains one signing function:

```ts
signTransaction:
  MoonzTransactionSigner
```

In the current public implementation it is called for the reconstructed `tx0` escrow funding transaction.

Do not design your integration assuming the callback will fire once for every signature in `CreateTokenResult`.

## 📡 Later Signatures Come From Moonz

The later operations:

```text
tx1

tx2

tx3

devbuy-escrow

settle
```

are requested through the public Moonz API.

Their responses contain the signatures that populate the final result.

## 🧠 Do Not Infer Signing From Signature Count

Seeing six signatures in the result does not mean the creator wallet approved six separate signing prompts.

The result records six launch transaction stages.

The wallet callback and the launch signature trail are different concepts.

{% hint style="warning" %}
Do not build a user interface that promises six wallet prompts based on the six final signatures.

The current public `createToken()` flow calls the creator signing function for the escrow funding transaction.
{% endhint %}

## 🚦 Progress Around the Sequence

The relevant public progress states line up with the launch journey:

```text
BUILDING_ESCROW_TX
      ↓
AWAITING_CREATOR_SIGNATURE
      ↓
SENDING_ESCROW_TX
      ↓
CONFIRMING_ESCROW
      ↓
INITIALIZING_LAUNCH
      ↓
CREATING_METADATA
      ↓
FINALIZING_MINT
      ↓
EXECUTING_DEV_BUY
      ↓
SETTLING
      ↓
LIVE
```

A frontend can use those stages rather than trying to infer progress from transaction count.

## 🔁 Why the Order Matters

These operations form one coordinated launch sequence.

For example:

```text
Dev Buy execution
must not be treated as
the first launch step

because

the launch must first
be initialized and finalized
```

Likewise, settlement comes after Dev Buy execution.

`createToken()` owns this ordering.

## 🛡️ Let the SDK Orchestrate

When using the public creation method, do not independently reorder:

```text
tx0

tx1

tx2

tx3

Dev Buy

settle
```

The supported high level integration is:

```ts
await moonz.createToken({
  ...
});
```

and let the SDK coordinate the sequence.

## 🧯 Errors Can Happen Between Stages

A multi stage launch can fail after some earlier operations have already succeeded.

That is why the SDK preserves:

```text
reserved mint

progress state

available transaction signatures
```

through the creation journey.

After mint reservation, thrown creation errors include the mint address in the message.

We will cover that recovery boundary separately.

## 🌌 Launch Sequence Summary

The final sequence is:

```text
tx0
Creator funds escrow
        ↓
fund signature
        ↓
tx0 confirm
        ↓
tx1
Initialize launch
        ↓
initialize signature
        ↓
tx2
Create metadata
        ↓
metadata signature
        ↓
tx3
Finalize mint
        ↓
finalize signature
        ↓
Dev Buy from escrow
        ↓
devBuy signature
        ↓
Settlement
        ↓
settle signature
        ↓
LIVE
```

{% hint style="success" %}
The signature set is a record of the launch journey.

The creator signing callback is specifically the authorization boundary for escrow funding.
{% endhint %}

## 📡 Next Stop

A launch has several stages.

Now we make those stages visible to the application while they happen.

Next stop:

**Progress and Callbacks**
