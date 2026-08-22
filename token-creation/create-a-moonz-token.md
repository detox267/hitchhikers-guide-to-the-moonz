# 🚀 Create a Moonz Token

The Trading SDK does more than trade Moonz.

It can launch one.

The public creation method is:

```ts
createToken()
```

It coordinates the complete Moonz creation flow from token details through image and metadata publication, mint reservation, Dev Buy preparation and launch settlement.

{% hint style="info" %}
Token creation uses both Solana and the public Moonz API.

This is different from normal trading quotes and transaction builders, which read current trading state directly from Solana.
{% endhint %}

## 🌙 The Public Creation Method

A basic launch looks like:

```ts
const result =
  await moonz.createToken({
    creator:
      wallet.publicKey,

    name:
      "Example Token",

    symbol:
      "EXAMPLE",

    description:
      "Created with Moonz",

    image:
      selectedFile,

    devBuySol:
      "0.5",

    slippageBps:
      100,

    signTransaction:
      wallet.signTransaction
  });
```

When the launch completes successfully:

```ts
console.log(
  result.mint
);
```

returns the Moonz token mint.

## 🧬 Creation Input

The public creation input is:

```ts
interface CreateTokenInput {
  creator:
    PublicKey | string;

  name:
    string;

  symbol:
    string;

  description?:
    string;

  image:
    MoonzImageInput;

  imageFilename?:
    string;

  imageContentType?:
    string;

  extensions?:
    Record<string, unknown>;

  devBuySol:
    string | number;

  slippageBps?:
    number;

  signTransaction:
    MoonzTransactionSigner;

  onMintReserved?: (
    mint: string
  ) => void | Promise<void>;

  onProgress?: (
    progress: MoonzCreateProgress
  ) => void | Promise<void>;
}
```

That gives an integration control over:

```text
Creator

Token name

Token symbol

Description

Image

Metadata extensions

Dev Buy amount

Dev Buy slippage

Wallet signing

Mint callback

Progress callback
```

## 👤 Creator

The creator can be supplied as:

```text
PublicKey

or

base58 address string
```

For example:

```ts
creator:
  wallet.publicKey
```

The SDK normalizes the creator to a Solana address before starting the launch flow.

## 🏷️ Name

The token name is required.

```ts
name:
  "Example Token"
```

The SDK trims surrounding whitespace before using it.

An empty name is rejected with:

```text
name is required
```

## 🔠 Symbol

The symbol is also required.

```ts
symbol:
  "example"
```

The SDK:

```text
trims whitespace

converts the symbol to uppercase
```

So:

```text
example
```

becomes:

```text
EXAMPLE
```

An empty symbol is rejected.

## 📝 Description

Description is optional.

```ts
description:
  "Created with Moonz"
```

If omitted, the creation flow sends an empty description when preparing metadata.

## 🖼️ Image

Every creation input requires an image.

The public image type supports:

```ts
Blob

Uint8Array

ArrayBuffer
```

The exported type is:

```ts
type MoonzImageInput =
  | Blob
  | Uint8Array
  | ArrayBuffer;
```

We will cover image handling in detail on the next page.

## 📦 Optional Image Information

You can also provide:

```ts
imageFilename

imageContentType
```

For example:

```ts
imageFilename:
  "moon-token.png",

imageContentType:
  "image/png"
```

If an uploaded browser `File` is used and no explicit filename is supplied, the SDK can use the file name.

Otherwise the fallback filename is:

```text
token-image
```

## 🧩 Metadata Extensions

Optional metadata extensions can be provided through:

```ts
extensions:
  {
    website:
      "https://example.com",

    twitter:
      "https://x.com/example"
  }
```

The public type is:

```ts
Record<string, unknown>
```

If extensions are omitted, the SDK sends an empty object.

## 💰 Dev Buy

Moonz creation includes:

```ts
devBuySol
```

For example:

```ts
devBuySol:
  "0.5"
```

This is expressed as a human readable SOL amount.

The SDK converts it into lamports for the launch flow.

We will cover Dev Buy quoting and escrow separately.

## 🛡️ Dev Buy Slippage

The creation flow also supports:

```ts
slippageBps
```

If omitted, it defaults to:

```text
100 basis points

1 percent
```

The valid SDK range is:

```text
0 through 9999
```

and the value must be an integer.

## ✍️ Transaction Signer

Creation requires:

```ts
signTransaction
```

Its public type is:

```ts
type MoonzTransactionSigner = (
  transaction: Transaction
) => Promise<Transaction>;
```

For a browser wallet this can be supplied from the connected wallet signing interface.

For example:

```ts
signTransaction:
  wallet.signTransaction
```

## 🔐 What the Creator Actually Signs

The creator signing step occurs when Moonz has prepared the launch escrow funding transaction.

The flow is:

```text
Moonz prepares tx0
      ↓
Unsigned escrow funding transaction
      ↓
Creator wallet signs
      ↓
Signed transaction submitted to Solana
      ↓
Escrow funding confirmed
```

The SDK does not ask for the creator's private key.

{% hint style="success" %}
The creator authorizes the escrow funding transaction through their wallet.

The private key remains inside the wallet.
{% endhint %}

## 🛰️ The Rest of the Launch Sequence

After escrow funding is confirmed, the SDK continues through the public Moonz launch API.

The remaining stages request the launch operations and receive their transaction signatures.

The high level sequence is:

```text
Reserve mint
      ↓
Upload image
      ↓
Pin metadata
      ↓
Quote Dev Buy
      ↓
Build escrow funding transaction
      ↓
Creator signs tx0
      ↓
Submit and confirm escrow funding
      ↓
Confirm tx0 with Moonz
      ↓
Initialize launch
      ↓
Create metadata
      ↓
Finalize mint
      ↓
Execute Dev Buy
      ↓
Settle
      ↓
LIVE
```

## 🌙 Mint Reservation Comes Early

One of the first creation operations is mint reservation.

The SDK requests:

```text
/launch/vanity
```

and receives the reserved mint address.

Once the mint has been validated as a Solana public key, the creation flow can expose it immediately.

## 📣 onMintReserved

The optional callback:

```ts
onMintReserved
```

fires as soon as the mint has been reserved.

Example:

```ts
onMintReserved(mint) {
  console.log(
    "Token CA:",
    mint
  );
}
```

This allows an application to learn the future token address before the entire launch has reached `LIVE`.

## 🧭 Why Early Mint Access Matters

A creation interface may want to:

```text
Display the CA

Copy the CA

Update application state

Prepare links

Associate local launch state
with the mint
```

without waiting for all launch stages to finish.

`onMintReserved()` provides that boundary.

## 📡 Creation Progress

The second optional callback is:

```ts
onProgress
```

It receives:

```ts
interface MoonzCreateProgress {
  step:
    MoonzProgress;

  mint?:
    string;

  signature?:
    string;
}
```

That makes it possible to build a detailed launch progress interface.

## 🚦 Public Progress Steps

The exported progress states are:

```text
RESERVING_MINT

MINT_RESERVED

UPLOADING_IMAGE

IMAGE_PINNED

PINNING_METADATA

METADATA_PINNED

QUOTING_DEV_BUY

BUILDING_ESCROW_TX

AWAITING_CREATOR_SIGNATURE

SENDING_ESCROW_TX

CONFIRMING_ESCROW

INITIALIZING_LAUNCH

CREATING_METADATA

FINALIZING_MINT

EXECUTING_DEV_BUY

SETTLING

LIVE
```

These states give the integration a public view of where `createToken()` currently is.

## 🖥️ Example Progress Interface

A simple integration can do:

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
      wallet.signTransaction,

    onProgress(progress) {
      console.log(
        progress.step
      );
    }
  });
```

A richer interface can map those progress values into a launch screen.

## 🌐 Public Moonz API

Unlike ordinary trading quotes, token creation uses the configured Moonz API.

The SDK constructor supports:

```ts
apiUrl
```

If no custom value is supplied, the default is:

```text
https://api.moonz.fun
```

Creation requests are made under:

```text
/launch
```

## 🛰️ Creation API Stages

The current public SDK creation flow uses endpoints for:

```text
vanity mint reservation

image publication

metadata publication

Dev Buy quote

tx0 escrow funding

tx0 confirmation

tx1 launch initialization

tx2 metadata creation

tx3 mint finalization

Dev Buy execution

settlement
```

The SDK orchestrates these for the caller.

## 🖼️ Image Publication

After mint reservation, the SDK uploads the image using:

```text
POST /launch/{mint}/pinata-image
```

The Moonz API returns an image URI.

That becomes:

```ts
result.imageUri
```

after successful creation.

## 🧾 Metadata Publication

The SDK then sends token metadata using:

```text
POST /launch/{mint}/pinata
```

including:

```text
creator

name

symbol

description

image URI

extensions
```

The returned URI becomes:

```ts
result.metadataUri
```

## 💰 Dev Buy Quote

Before funding escrow, the SDK requests a Dev Buy quote from:

```text
POST /launch/{mint}/devbuy-quote
```

The request contains:

```text
devBuyLamports

slippageBps
```

The response includes:

```text
expectedTokensOut

minTokensOut
```

Those values are preserved in the final creation result.

## 🧱 Escrow Funding Transaction

The SDK then requests:

```text
POST /launch/{mint}/tx0
```

using:

```text
creator

devBuyLamports

devBuyMinTokensOut
```

The API returns a serialized transaction plus its blockhash context.

The SDK reconstructs that into a Solana `Transaction`.

## 👛 Creator Approval

At:

```text
AWAITING_CREATOR_SIGNATURE
```

the SDK calls the supplied:

```ts
signTransaction
```

The resulting signed transaction is serialized and sent through the SDK's Solana connection.

## 📡 Escrow Submission

The signed escrow funding transaction is submitted using:

```ts
sendRawTransaction()
```

with preflight enabled.

The SDK then confirms it using the blockhash and last valid block height returned with `tx0`.

## ✅ Moonz Escrow Confirmation

After Solana confirmation, the SDK reports the signature back to Moonz through:

```text
POST /launch/{mint}/tx0/confirm
```

This lets the creation flow move into the remaining launch stages.

## 🚀 Launch Initialization

Next the SDK requests:

```text
POST /launch/{mint}/tx1
```

with creator, name and symbol information.

The API returns the initialization signature.

## 🧾 Metadata Creation

Then:

```text
POST /launch/{mint}/tx2
```

is requested with:

```text
name

symbol
```

The returned signature becomes the creation result's metadata signature.

## 🔒 Mint Finalization

The next stage is:

```text
POST /launch/{mint}/tx3
```

The resulting signature represents the finalization stage.

## 💸 Execute Dev Buy

Once those stages are complete, the SDK requests:

```text
POST /launch/{mint}/devbuy-escrow
```

with:

```text
minTokensOut
```

from the Dev Buy quote.

This executes the prepared Dev Buy path.

## 🌕 Settle

The final launch operation is:

```text
POST /launch/{mint}/settle
```

Once that succeeds, the progress state becomes:

```text
LIVE
```

and `createToken()` returns its final result.

## ✅ Creation Result

The public return structure is:

```ts
interface CreateTokenResult {
  mint: string;

  imageUri: string;

  metadataUri: string;

  devBuyLamports: string;

  expectedTokensOut: string;

  minTokensOut: string;

  signatures: {
    fund: string;
    initialize: string;
    metadata: string;
    finalize: string;
    devBuy: string;
    settle: string;
  };

  status:
    "LIVE";
}
```

## 🧾 Six Signatures

A completed creation result records:

```text
fund

initialize

metadata

finalize

devBuy

settle
```

This provides a useful transaction trail for the complete launch.

## 🔎 Example Result Handling

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

console.log({
  mint:
    result.mint,

  status:
    result.status,

  imageUri:
    result.imageUri,

  metadataUri:
    result.metadataUri,

  expectedTokensOut:
    result.expectedTokensOut,

  minTokensOut:
    result.minTokensOut,

  signatures:
    result.signatures
});
```

## 🌕 Successful Completion

The only successful public status returned by `CreateTokenResult` is:

```text
LIVE
```

If the method returns normally, the creation flow has reached its final live stage.

## ❌ Creation Errors

The creation method wraps errors after a mint has been reserved.

If failure occurs after the mint is known, the thrown message is prefixed with:

```text
Moonz createToken failed for {mint}:
```

That is useful because the application retains the reserved mint context even when a later stage fails.

Before mint reservation, the original error is thrown.

## 🧭 This Is an Orchestrated Flow

`createToken()` is intentionally high level.

The caller provides the token details and creator authorization.

The SDK coordinates the launch sequence.

```text
APPLICATION
    ↓
createToken()
    ↓
MOONZ SDK
    ↓
Moonz API
+
Solana
    ↓
Token LIVE
```

This is different from the trade builders, where the application can request a single unsigned trade transaction directly.

## 🌌 Complete Creation Journey

The whole public path is:

```text
Token details
      ↓
Validate name and symbol
      ↓
Validate slippage
      ↓
Convert Dev Buy SOL
      ↓
RESERVING_MINT
      ↓
Mint reserved
      ↓
onMintReserved()
      ↓
Upload image
      ↓
Pin metadata
      ↓
Quote Dev Buy
      ↓
Build tx0
      ↓
Creator signs escrow funding
      ↓
Submit to Solana
      ↓
Confirm escrow
      ↓
Initialize launch
      ↓
Create metadata
      ↓
Finalize mint
      ↓
Execute Dev Buy
      ↓
Settle
      ↓
LIVE
      ↓
CreateTokenResult
```

{% hint style="success" %}
One public method coordinates the Moonz launch journey while still keeping creator authorization inside the creator's wallet.
{% endhint %}

## 🖼️ Next Stop

The token has a name and symbol.

Now it needs something the internet can actually look at.

Next stop:

**Image and Metadata**
