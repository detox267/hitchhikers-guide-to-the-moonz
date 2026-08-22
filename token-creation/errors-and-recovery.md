# 🛡️ Errors and Recovery

Token creation is a multi stage operation.

A failure can happen before the mint exists, during image publication, while waiting for the creator wallet, or after one or more launch transactions have already completed.

The important rule is simple:

{% hint style="warning" %}
Do not treat every failed `createToken()` call as though nothing happened.

A launch may already have a reserved mint or completed transaction stages.
{% endhint %}

## 🌌 Where Creation Can Fail

The public flow crosses several boundaries:

```text
Input validation
      ↓
Mint reservation
      ↓
Image publication
      ↓
Metadata publication
      ↓
Dev Buy quote
      ↓
Escrow transaction preparation
      ↓
Creator wallet signing
      ↓
Solana submission
      ↓
Escrow confirmation
      ↓
Launch initialization
      ↓
Metadata transaction
      ↓
Mint finalization
      ↓
Dev Buy execution
      ↓
Settlement
```

Any network, wallet, API or Solana error along that journey can cause `createToken()` to reject.

## 🚦 Fail Fast Before Launch Work

Some errors are detected before the mint reservation request begins.

Examples include:

```text
Missing name

Missing symbol

Invalid slippage

Invalid Dev Buy amount

Invalid creator address
```

These failures happen before a launch mint has been reserved.

## 🏷️ Missing Name

The SDK trims:

```ts
input.name
```

and rejects an empty value with:

```text
name is required
```

For example, this fails:

```ts
name:
  "   "
```

because the trimmed name is empty.

## 🔠 Missing Symbol

The SDK trims and uppercases:

```ts
input.symbol
```

An empty symbol fails with:

```text
symbol is required
```

## 🛡️ Invalid Slippage

Creation slippage must be an integer from:

```text
0
through
9999
```

Otherwise the SDK throws:

```text
slippageBps must be from 0 to 9999
```

## 💰 Invalid Dev Buy Amount

`devBuySol` must use the decimal form accepted by the SDK.

Invalid input throws:

```text
Invalid devBuySol
```

Examples of ordinary accepted forms include:

```text
0

0.5

1

1.25
```

## 👤 Invalid Creator

The SDK normalizes the creator using Solana `PublicKey`.

If a supplied creator string is not a valid Solana public key, public key construction can throw before mint reservation begins.

## 🌙 The Mint Changes the Error Boundary

Inside `createToken()`, the SDK starts with:

```ts
let mint = "";
```

Once the vanity request returns a mint, it is normalized as a Solana public key and stored.

From that point onward, failures are wrapped with the mint address.

## ❌ Failure Before Mint Reservation

If an error occurs while:

```text
mint = ""
```

the original error is rethrown.

Conceptually:

```text
Validation fails
      ↓
No reserved mint
      ↓
Original error
```

## 🛰️ Failure After Mint Reservation

Once the mint is known, the catch block throws:

```text
Moonz createToken failed for {mint}: {message}
```

For example:

```text
Moonz createToken failed for ABC...XYZ:
Moonz API did not return metadata URI
```

This gives the application a stable identifier for the launch that encountered the problem.

## 📣 Capture the Mint Early

If recovery or diagnostics matter to your application, use:

```ts
onMintReserved
```

to store the mint as soon as it becomes available.

Example:

```ts
let reservedMint:
  string | undefined;

await moonz.createToken({
  creator:
    wallet.publicKey,

  name,
  symbol,
  image,
  devBuySol,

  signTransaction:
    wallet.signTransaction,

  onMintReserved(mint) {
    reservedMint = mint;
  }
});
```

If a later stage throws, your application still has the mint independently of parsing an error string.

## 📡 Capture Progress Too

`onProgress()` can tell you the latest stage reached.

For example:

```ts
let latestStep:
  MoonzProgress | undefined;

await moonz.createToken({
  creator,
  name,
  symbol,
  image,
  devBuySol,
  signTransaction,

  onProgress(progress) {
    latestStep =
      progress.step;
  }
});
```

This can help distinguish:

```text
Image failure

Wallet stage failure

Escrow stage failure

Launch transaction failure

Settlement failure
```

## 🧾 Preserve Signatures as They Appear

Later progress events can also contain transaction signatures.

An integration that needs a diagnostic trail can retain them as they arrive.

```ts
const observedSignatures:
  string[] = [];

await moonz.createToken({
  creator,
  name,
  symbol,
  image,
  devBuySol,
  signTransaction,

  onProgress(progress) {
    if (
      progress.signature
    ) {
      observedSignatures.push(
        progress.signature
      );
    }
  }
});
```

This can be useful if creation fails after an earlier transaction has already completed.

## 🧠 A Rejected Promise Does Not Mean Zero On Chain Work

Consider a failure at:

```text
SETTLING
```

By then the flow may already have obtained signatures for:

```text
fund

initialize

metadata

finalize

devBuy
```

The fact that `createToken()` ultimately rejects does not erase those completed operations.

## 👛 Wallet Rejection

The creator can reject the `tx0` signing request.

That happens after:

```text
AWAITING_CREATOR_SIGNATURE
```

and before the signed escrow funding transaction is submitted.

A wallet rejection should be treated differently from a later on chain launch failure.

## ✍️ Signer Errors

The public signer type is:

```ts
type MoonzTransactionSigner = (
  transaction: Transaction
) => Promise<Transaction>;
```

If the supplied signer throws or rejects, that error propagates through the creation flow.

After a mint has been reserved, it will be wrapped with the mint context.

## 🖼️ Image Errors

The SDK can fail before or during image upload.

An unsupported local image representation produces:

```text
Unsupported image input
```

If the API response succeeds but contains no image URI, the SDK throws:

```text
Moonz API did not return image URI
```

## 🧾 Metadata Errors

If metadata publication does not return a URI, creation throws:

```text
Moonz API did not return metadata URI
```

Because the mint has already been reserved by this point, the final thrown message includes the mint prefix.

## 🌐 Moonz API Errors

Moonz API requests are checked for successful HTTP responses.

When an API response is not successful, the SDK prefers an API supplied:

```text
error

or

message
```

when one is available.

Otherwise it reports:

```text
Moonz API returned HTTP {status}
```

## 🧬 Invalid API JSON

If the HTTP response is successful but cannot be parsed as JSON, the SDK throws:

```text
Moonz API returned invalid JSON
```

If an unsuccessful response also cannot be parsed, the SDK falls back to the HTTP status error.

## 📡 Solana Submission Errors

The creator signed funding transaction is submitted using:

```ts
sendRawTransaction()
```

RPC submission errors can therefore reject the creation call before the later launch stages begin.

## ⛓️ Confirmation Errors

The SDK also awaits confirmation of the escrow funding transaction using:

```text
signature

blockhash

lastValidBlockHeight
```

An RPC or confirmation failure can interrupt creation at this boundary.

## 📣 Callback Errors Matter

Both:

```ts
onMintReserved

onProgress
```

are awaited by the SDK.

That means application code inside those callbacks participates directly in the creation flow.

A thrown callback error or rejected callback promise can cause `createToken()` to reject.

## ⚠️ Optional Telemetry Should Not Break Launches

Suppose your progress callback sends analytics:

```ts
onProgress:
  async (progress) => {
    await analytics.track(
      progress
    );
  }
```

If that analytics request rejects, the callback rejects too.

Because the SDK awaits it, token creation can be interrupted by something unrelated to the protocol.

## 🛡️ Isolate Nonessential Callback Work

For optional work, handle failure locally.

```ts
onProgress:
  async (progress) => {
    try {
      await analytics.track(
        progress
      );
    } catch (error) {
      console.error(
        error
      );
    }
  }
```

Your application can decide which callback operations are important enough to stop creation.

## 🌕 LIVE Is the Success Boundary

Intermediate progress does not equal completed creation.

The successful method return is:

```ts
CreateTokenResult
```

with:

```text
status = LIVE
```

If `createToken()` resolves normally, the public result has reached that terminal state.

## 🚫 There Is No Public Resume Method

The current public package exports:

```ts
createToken()
```

for creation.

It does not expose a separate public method such as:

```text
resumeCreateToken

recoverCreateToken

continueLaunch
```

for continuing an interrupted creation from a known stage.

{% hint style="warning" %}
Do not invent a recovery endpoint or resume method in your integration.

Use only the public surfaces exposed by the SDK.
{% endhint %}

## 🔁 Do Not Blindly Call createToken Again

A fresh call to:

```ts
createToken()
```

begins its flow by reserving a mint again.

Conceptually:

```text
New createToken() call
      ↓
RESERVING_MINT
      ↓
/vanity
      ↓
Mint reservation
```

Therefore a second call should not be assumed to resume the first mint.

## 🧭 Preserve Context Before Deciding What to Do

For an integration that needs operational recovery, retain:

```text
Reserved mint

Latest progress step

Observed signatures

Original error
```

before deciding how to handle an interrupted launch.

## 🗃️ Example Recovery Context

```ts
let reservedMint:
  string | undefined;

let latestProgress:
  MoonzCreateProgress | undefined;

const signatures:
  string[] = [];

try {
  const result =
    await moonz.createToken({
      creator,
      name,
      symbol,
      image,
      devBuySol,
      signTransaction,

      onMintReserved(mint) {
        reservedMint =
          mint;
      },

      onProgress(progress) {
        latestProgress =
          progress;

        if (
          progress.signature
        ) {
          signatures.push(
            progress.signature
          );
        }
      }
    });

  console.log(
    result.status
  );
} catch (error) {
  console.error({
    reservedMint,
    latestProgress,
    signatures,
    error
  });
}
```

This does not automatically recover the launch.

It gives your application the evidence needed to understand what happened.

## 🔍 Distinguish Failure Stages

A useful application model is:

```text
PRE MINT FAILURE
No reserved mint

POST MINT PRE FUNDING FAILURE
Mint exists
No creator funding submitted yet

FUNDING FAILURE
Creator signing or Solana funding stage failed

POST FUNDING FAILURE
Some launch work may already exist

LATE LAUNCH FAILURE
Several signatures may already exist

LIVE
Creation completed successfully
```

## 🧯 Do Not Hide Partial State

A poor error interface might say:

```text
Launch failed
Try again
```

for every failure.

That can be misleading after the mint or transactions already exist.

A better interface can retain the mint and tell the user which stage stopped.

## 🖥️ Example User Facing State

For example:

```text
Launch interrupted

Token CA
ABC...XYZ

Last stage
FINALIZING_MINT

Keep this token address for support
or diagnostics
```

The exact wording belongs to your application.

## 🛰️ Explorer Links

When a progress callback has already supplied a signature, an application can use it to offer an appropriate Solana explorer link.

Keep in mind that the meaning of the signature depends on the progress mapping documented in the previous page.

## 🧠 Do Not Infer Final State From One Signature

A valid:

```text
fund signature
```

does not mean:

```text
Token is LIVE
```

Likewise, a finalization signature still does not prove Dev Buy execution and settlement have completed.

The complete success boundary remains the resolved `CreateTokenResult`.

## 🔬 Error Message Versus Stored Context

The mint is included in wrapped errors after reservation.

Even so, storing:

```ts
onMintReserved(mint)
```

is cleaner than relying on parsing:

```text
Moonz createToken failed for ...
```

from human readable error text.

## 🔁 Retry Policy Belongs to the Application

The current public SDK does not define an automatic retry policy for the complete creation workflow.

That is important because some stages have side effects.

An application should not blindly rerun the whole creation operation simply because a network request failed.

## 🛡️ Safe Integration Principle

Treat creation as an orchestrated stateful operation.

```text
Before mint reservation
      ↓
Retry may begin a new launch attempt

After mint reservation
      ↓
Preserve mint context

After transaction signatures
      ↓
Preserve transaction context

Before retrying anything
      ↓
Understand the actual launch state
```

## 🌌 Error Journey

A robust integration should think about errors like this:

```text
createToken()
      ↓
Validate input
      ↓
Reserve mint
      ↓
CAPTURE MINT
      ↓
Track progress
      ↓
CAPTURE SIGNATURES
      ↓
Launch continues
     / \
    /   \
SUCCESS FAILURE
  ↓       ↓
LIVE    Preserve
result   context
          ↓
       Diagnose
          ↓
       Do not
       blindly
       restart
```

{% hint style="success" %}
The most useful recovery data is collected before a failure happens.

Capture the mint, progress and signatures while the launch is running.
{% endhint %}

## 🌕 Token Creation Complete

You now have the complete public creation surface:

```text
Create token

Publish image

Publish metadata

Quote Dev Buy

Fund escrow

Initialize launch

Create metadata

Finalize mint

Execute Dev Buy

Settle

Track progress

Handle failures
```

From creator input to a `LIVE` Moonz token, the full public flow is now mapped.
