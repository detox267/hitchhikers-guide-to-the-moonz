# 📡 Progress and Callbacks

`createToken()` coordinates several network and Solana operations.

A creation interface should not have to guess where the launch currently is.

Moonz exposes two public callbacks for that job:

```ts
onMintReserved

onProgress
```

They serve different purposes.

{% hint style="info" %}
Use `onMintReserved()` when you specifically need the token address.

Use `onProgress()` when you want to follow the complete launch journey.
{% endhint %}

## 🌙 onMintReserved

The public callback is:

```ts
onMintReserved?: (
  mint: string
) => void | Promise<void>;
```

It is called after Moonz has successfully reserved and validated the mint address.

Example:

```ts
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

  onMintReserved(mint) {
    console.log(
      "Token CA:",
      mint
    );
  }
});
```

## 🚀 When the Mint Callback Fires

The opening sequence is:

```text
RESERVING_MINT
      ↓
Moonz reserves mint
      ↓
MINT_RESERVED
      ↓
onMintReserved(mint)
      ↓
UPLOADING_IMAGE
```

This means the token address can be exposed before image publication, metadata publication and launch settlement are complete.

## 🧭 What You Can Do With It

Typical uses include:

```text
Display the token CA

Add a copy button

Store the mint in local launch state

Prepare token links

Associate later progress
with the reserved mint
```

The callback means the mint exists as the identity of the launch.

It does not mean the complete launch has reached `LIVE`.

## 📡 onProgress

The broader callback is:

```ts
onProgress?: (
  progress:
    MoonzCreateProgress
) => void | Promise<void>;
```

The progress object is:

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

## 🛰️ MoonzProgress

The complete public progress type is:

```ts
type MoonzProgress =
  | "RESERVING_MINT"
  | "MINT_RESERVED"
  | "UPLOADING_IMAGE"
  | "IMAGE_PINNED"
  | "PINNING_METADATA"
  | "METADATA_PINNED"
  | "QUOTING_DEV_BUY"
  | "BUILDING_ESCROW_TX"
  | "AWAITING_CREATOR_SIGNATURE"
  | "SENDING_ESCROW_TX"
  | "CONFIRMING_ESCROW"
  | "INITIALIZING_LAUNCH"
  | "CREATING_METADATA"
  | "FINALIZING_MINT"
  | "EXECUTING_DEV_BUY"
  | "SETTLING"
  | "LIVE";
```

These values are exported from:

```text
@moonz-fun/trading-sdk
```

## 🌌 Complete Progress Journey

A normal successful creation travels through:

```text
RESERVING_MINT
      ↓
MINT_RESERVED
      ↓
UPLOADING_IMAGE
      ↓
IMAGE_PINNED
      ↓
PINNING_METADATA
      ↓
METADATA_PINNED
      ↓
QUOTING_DEV_BUY
      ↓
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

## 🌑 RESERVING_MINT

The first progress event is:

```text
RESERVING_MINT
```

At this point the SDK is about to request the mint reservation.

A mint is not yet attached to this first progress message.

## 🌙 MINT_RESERVED

After Moonz returns the mint and the SDK validates it as a Solana public key:

```text
MINT_RESERVED
```

is emitted with:

```ts
progress.mint
```

The separate `onMintReserved()` callback follows immediately afterward.

## 🖼️ UPLOADING_IMAGE

Next:

```text
UPLOADING_IMAGE
```

means the SDK is entering the image upload stage.

The reserved mint is included.

## ✅ IMAGE_PINNED

After a valid image URI has been returned:

```text
IMAGE_PINNED
```

is emitted.

At this point the image publication stage has succeeded.

## 🧾 PINNING_METADATA

Then:

```text
PINNING_METADATA
```

signals that the SDK is moving into metadata publication.

## ✅ METADATA_PINNED

When a valid metadata URI has been returned:

```text
METADATA_PINNED
```

is emitted.

The next stage is Dev Buy preparation.

## 💬 QUOTING_DEV_BUY

The SDK emits:

```text
QUOTING_DEV_BUY
```

before requesting the Dev Buy quote.

This quote produces the expected output and protected minimum output used later in the launch.

## 🧱 BUILDING_ESCROW_TX

Next:

```text
BUILDING_ESCROW_TX
```

means Moonz is preparing the `tx0` escrow funding transaction.

## 👛 AWAITING_CREATOR_SIGNATURE

Once `tx0` has been reconstructed into a Solana transaction:

```text
AWAITING_CREATOR_SIGNATURE
```

is emitted.

Immediately afterward the SDK calls:

```ts
input.signTransaction(
  transaction
)
```

This is the point where a frontend should expect the creator wallet approval flow.

## 📡 SENDING_ESCROW_TX

After the signing callback returns:

```text
SENDING_ESCROW_TX
```

is emitted.

The signed escrow funding transaction is then submitted to Solana.

## ⛓️ CONFIRMING_ESCROW

After submission returns a transaction signature:

```text
CONFIRMING_ESCROW
```

is emitted with:

```ts
progress.signature
```

set to the creator funding transaction signature.

The SDK then waits for Solana confirmation.

## 🚀 INITIALIZING_LAUNCH

After the escrow confirmation stage and Moonz `tx0` confirmation request:

```text
INITIALIZING_LAUNCH
```

is emitted.

The SDK then requests:

```text
tx1
```

for launch initialization.

## 🧾 CREATING_METADATA

After `tx1` returns:

```text
CREATING_METADATA
```

is emitted.

This progress message includes:

```ts
progress.signature
```

containing the `tx1` initialization signature.

The SDK then moves into `tx2`.

## 🔒 FINALIZING_MINT

After `tx2` returns:

```text
FINALIZING_MINT
```

is emitted.

Its signature field contains the `tx2` metadata transaction signature.

The SDK then requests `tx3`.

## 💸 EXECUTING_DEV_BUY

After `tx3` returns:

```text
EXECUTING_DEV_BUY
```

is emitted.

Its signature field contains the `tx3` finalization transaction signature.

The SDK then requests the Dev Buy execution.

## 🌕 SETTLING

After the Dev Buy operation returns:

```text
SETTLING
```

is emitted.

Its signature field contains the Dev Buy transaction signature.

The SDK then requests settlement.

## ✅ LIVE

After settlement returns:

```text
LIVE
```

is emitted.

Its signature field contains the settlement transaction signature.

`createToken()` then returns the final successful result.

## 🧠 Understanding step and signature

There is an important pattern once the later launch transactions begin.

`step` describes the stage the SDK is entering.

`signature` can describe the transaction that just completed.

For example:

```text
CREATING_METADATA
+
tx1 signature
```

means:

```text
tx1 initialization finished
      ↓
Its signature is now known
      ↓
SDK is entering
metadata creation
```

Likewise:

```text
FINALIZING_MINT
+
tx2 signature
```

means `tx2` has completed and the SDK is moving into `tx3`.

## 🗺️ Progress Signature Map

The current public flow maps signatures like this:

```text
CONFIRMING_ESCROW
    signature = fund

CREATING_METADATA
    signature = initialize

FINALIZING_MINT
    signature = metadata

EXECUTING_DEV_BUY
    signature = finalize

SETTLING
    signature = devBuy

LIVE
    signature = settle
```

This is useful when building transaction links during creation.

## ⚠️ Do Not Assume step Equals Signature Type

Avoid code that assumes:

```text
step
=
transaction represented
by signature
```

That is not always the current callback contract.

Instead interpret the progress events according to the public sequence.

{% hint style="warning" %}
For later launch stages, the signature often records what just completed while the step names what happens next.
{% endhint %}

## 🖥️ Simple Progress Display

A basic interface can use:

```ts
onProgress(progress) {
  setLaunchStep(
    progress.step
  );
}
```

Then render a message based on the current step.

## 🎛️ Example Progress Labels

For example:

```ts
const labels = {
  RESERVING_MINT:
    "Reserving token address",

  MINT_RESERVED:
    "Token address reserved",

  UPLOADING_IMAGE:
    "Uploading image",

  IMAGE_PINNED:
    "Image published",

  PINNING_METADATA:
    "Publishing metadata",

  METADATA_PINNED:
    "Metadata published",

  QUOTING_DEV_BUY:
    "Calculating Dev Buy",

  BUILDING_ESCROW_TX:
    "Preparing launch funding",

  AWAITING_CREATOR_SIGNATURE:
    "Approve launch funding",

  SENDING_ESCROW_TX:
    "Submitting launch funding",

  CONFIRMING_ESCROW:
    "Confirming launch funding",

  INITIALIZING_LAUNCH:
    "Initializing launch",

  CREATING_METADATA:
    "Creating on chain metadata",

  FINALIZING_MINT:
    "Finalizing mint",

  EXECUTING_DEV_BUY:
    "Executing Dev Buy",

  SETTLING:
    "Settling launch",

  LIVE:
    "Token is live"
};
```

The exact wording belongs to your application.

## 🔗 Capture Signatures as They Arrive

You can also record signatures:

```ts
onProgress(progress) {
  if (
    progress.signature
  ) {
    console.log(
      progress.step,
      progress.signature
    );
  }
}
```

This can be useful for:

```text
Explorer links

Launch diagnostics

Transaction logs

Support tooling
```

## 🌙 Capture the Mint Once

You can use `onMintReserved()` for a dedicated mint update:

```ts
onMintReserved(mint) {
  setMint(mint);
}
```

or read the mint from later progress events.

The dedicated callback is convenient when your application wants a clear one purpose mint notification.

## ⚡ Async Callbacks Are Supported

Both callback types can return:

```text
void

or

Promise<void>
```

That means this is valid:

```ts
onMintReserved:
  async (mint) => {
    await saveMintLocally(
      mint
    );
  }
```

and:

```ts
onProgress:
  async (progress) => {
    await recordProgress(
      progress
    );
  }
```

## ⏳ Callbacks Are Awaited

The SDK awaits the callbacks.

For progress:

```ts
await callback(
  progress
);
```

The mint callback is also awaited.

This means callback work happens inside the `createToken()` orchestration rather than being launched in the background.

## ⚠️ Keep Callbacks Reliable

Because the SDK awaits these callbacks, an exception or rejected promise from application callback code can interrupt `createToken()`.

Avoid putting fragile unrelated work directly inside the callback.

For example, be careful with:

```text
Optional analytics

Third party logging

Nonessential network writes
```

if failure in those operations should not stop the launch flow.

## 🛡️ Isolate Optional Work

If an operation is not essential to creation, your callback can handle its own failure.

For example:

```ts
onProgress:
  async (progress) => {
    try {
      await sendAnalytics(
        progress
      );
    } catch (error) {
      console.error(
        error
      );
    }
  }
```

That way an optional analytics failure does not have to become a creation failure.

## 🧭 Important Versus Optional Callback Work

A useful distinction is:

```text
Essential application state
      ↓
May belong in awaited callback

Optional telemetry
      ↓
Handle failures locally
```

Your application decides which callback work is important enough to affect launch continuation.

## 🔁 Callbacks Describe This Invocation

`onProgress()` reports the stages occurring during the active `createToken()` call.

It is not a historical event query.

If your application needs a permanent launch history, store the progress information you care about as it arrives.

## 🧱 Progress Is Not the Final Result

Progress events help you render an active launch.

The authoritative successful return value is still:

```ts
CreateTokenResult
```

which contains:

```text
mint

imageUri

metadataUri

Dev Buy values

six signatures

LIVE status
```

## ✅ Wait for createToken to Resolve

Do not treat an intermediate progress event as equivalent to full creation success.

For example:

```text
FINALIZING_MINT
```

does not mean settlement is complete.

Even:

```text
EXECUTING_DEV_BUY
```

still has later work remaining.

The completed method returns:

```text
status = LIVE
```

## 🧪 Full Callback Example

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

    slippageBps:
      100,

    signTransaction:
      wallet.signTransaction,

    async onMintReserved(
      mint
    ) {
      console.log(
        "Reserved mint:",
        mint
      );
    },

    async onProgress(
      progress
    ) {
      console.log(
        "Step:",
        progress.step
      );

      if (
        progress.mint
      ) {
        console.log(
          "Mint:",
          progress.mint
        );
      }

      if (
        progress.signature
      ) {
        console.log(
          "Signature:",
          progress.signature
        );
      }
    }
  });

console.log(
  result.status
);
```

## 🌌 Callback Journey

The complete application visibility layer looks like:

```text
createToken()
      ↓
RESERVING_MINT
      ↓
MINT_RESERVED
      ↓
onMintReserved()
      ↓
Image progress
      ↓
Metadata progress
      ↓
Dev Buy quote progress
      ↓
Creator signature progress
      ↓
Escrow confirmation
      ↓
Launch transaction progress
      ↓
Dev Buy execution
      ↓
Settlement
      ↓
LIVE
      ↓
CreateTokenResult
```

{% hint style="success" %}
The creation method performs the launch.

The callbacks let your application travel alongside it.
{% endhint %}

## 🛡️ Next Stop

A multi stage launch can stop before it reaches `LIVE`.

Next we will cover what the SDK throws, what context remains available and how applications should treat partial creation failures.

Next stop:

**Errors and Recovery**
