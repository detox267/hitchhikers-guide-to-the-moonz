# 💰 Dev Buy

A Moonz launch can begin with a creator purchase.

The public creation input for that purchase is:

```ts
devBuySol
```

The creator chooses the SOL amount.

Moonz quotes the expected tokens, applies slippage protection, prepares an escrow funding transaction and uses the funded escrow later in the launch sequence.

{% hint style="info" %}
The creator signs the escrow funding transaction.

The later Dev Buy execution uses the funded launch escrow as part of the coordinated Moonz creation flow.
{% endhint %}

## 🌙 Dev Buy Input

`CreateTokenInput` requires:

```ts
devBuySol:
  string | number
```

For example:

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
```

Here:

```text
0.5
```

means:

```text
0.5 SOL
```

## 🔢 Strings Are a Good Fit

The SDK accepts either:

```text
string

number
```

For precise user entered decimal amounts, a decimal string is a straightforward choice.

For example:

```ts
devBuySol:
  "0.5"
```

The SDK converts the value into lamports before contacting the Moonz launch API.

## 🪙 SOL to Lamports

Moonz uses:

```text
1 SOL
=
1000000000 lamports
```

So conceptually:

```text
0.5 SOL
=
500000000 lamports
```

The creation flow converts `devBuySol` into an integer lamport amount.

## 🧮 Conversion Behaviour

The SDK converts the input by:

```text
Convert value to text

Trim whitespace

Validate decimal format

Split whole and fractional SOL

Pad fraction to 9 decimal places

Use at most 9 decimal places

Convert to lamports
```

The accepted textual form is an ordinary non negative decimal value such as:

```text
0

0.5

1

1.25
```

## 🚫 Invalid Dev Buy Format

Inputs that do not match the SDK decimal format fail with:

```text
Invalid devBuySol
```

For example, integrations should supply a normal decimal representation rather than relying on alternate numeric notation.

## 🛡️ Dev Buy Slippage

The Dev Buy uses the same public creation field:

```ts
slippageBps
```

If omitted, the SDK defaults to:

```text
100 basis points
=
1 percent
```

The value must be an integer from:

```text
0
through
9999
```

## 💬 Quote Before Funding

After the image and metadata stages, the SDK emits:

```text
QUOTING_DEV_BUY
```

It then asks Moonz to quote the creator purchase.

The request is made through:

```text
POST /launch/{mint}/devbuy-quote
```

## 📦 Dev Buy Quote Request

The SDK sends:

```text
devBuyLamports

slippageBps
```

Conceptually:

```text
devBuySol
    ↓
Convert to lamports
    ↓
devBuyLamports
    ↓
Dev Buy quote
```

## 🎯 Expected Tokens

The quote calculates:

```text
expectedTokensOut
```

This is the expected creator token output from the Dev Buy calculation.

The final successful `CreateTokenResult` preserves it as:

```ts
result.expectedTokensOut
```

## 🛡️ Minimum Tokens

The quote also produces:

```text
minTokensOut
```

This is the slippage protected minimum token output.

The successful creation result returns:

```ts
result.minTokensOut
```

## 🌌 Expected Versus Minimum

The distinction is the same principle used by normal Moonz trading.

```text
expectedTokensOut
        ↓
Expected creator token amount

minTokensOut
        ↓
Protected minimum accepted
for the Dev Buy
```

The minimum output is carried forward into the launch sequence.

## 🧱 Build the Escrow Funding Transaction

After the Dev Buy quote, the SDK emits:

```text
BUILDING_ESCROW_TX
```

It then requests:

```text
POST /launch/{mint}/tx0
```

## 📦 tx0 Inputs

The SDK supplies:

```text
creator

devBuyLamports

devBuyMinTokensOut
```

`devBuyMinTokensOut` comes directly from:

```text
quote.minTokensOut
```

So the Dev Buy protection is established before the creator funds the launch escrow.

## 🛰️ tx0 Response

Moonz returns the escrow funding transaction together with Solana blockhash context.

The SDK expects:

```text
mint

creator

blockhash

lastValidBlockHeight

tx
```

The transaction itself is returned as encoded transaction data.

## 🧬 Reconstruct the Transaction

The SDK reconstructs the Solana transaction from the returned data.

Conceptually:

```text
Moonz API
    ↓
Encoded tx0
    ↓
Decode
    ↓
Solana Transaction
    ↓
Creator wallet
```

At this point it has not yet been signed by the creator.

## 👛 Creator Signature

Immediately before requesting creator authorization, the SDK emits:

```text
AWAITING_CREATOR_SIGNATURE
```

Then it calls:

```ts
input.signTransaction(
  transaction
)
```

This is the creator authorization point.

## 🔐 What the Creator Authorizes

The creator signs the escrow funding transaction.

Conceptually:

```text
Creator SOL
    ↓
Signed tx0
    ↓
Launch escrow funded
```

The SDK never needs the creator's private key.

The connected wallet performs the signature.

{% hint style="success" %}
The creator approves the funding transaction through their wallet.

Moonz receives the signed transaction, not the wallet secret.
{% endhint %}

## 📡 Send Escrow Funding

After signing, the SDK emits:

```text
SENDING_ESCROW_TX
```

The signed transaction is submitted with:

```ts
sendRawTransaction()
```

using preflight.

The SDK currently supplies:

```text
skipPreflight
false

preflightCommitment
SDK commitment
```

## ⛓️ Confirm Escrow

Once the transaction has been submitted, the SDK emits:

```text
CONFIRMING_ESCROW
```

This progress event can include the transaction signature.

The SDK waits on Solana confirmation using:

```text
fund signature

tx0 blockhash

tx0 lastValidBlockHeight
```

## 📣 Report tx0 to Moonz

After the confirmation call completes, the SDK reports the funding signature back to Moonz through:

```text
POST /launch/{mint}/tx0/confirm
```

The request includes:

```text
txSig

creator
```

This associates the submitted escrow funding transaction with the continuing launch flow.

## 🧭 Why Escrow Exists

The creation sequence separates creator authorization from later Dev Buy execution.

Conceptually:

```text
CREATOR
   ↓
Signs tx0
   ↓
Funds launch escrow
   ↓
Launch initialization
   ↓
Metadata creation
   ↓
Mint finalization
   ↓
Dev Buy executed
from funded escrow
```

The creator authorizes the SOL funding before Moonz moves through the remaining launch stages.

## 🚀 Launch Continues Before Dev Buy Execution

After tx0 confirmation, the creation flow continues through:

```text
INITIALIZING_LAUNCH

CREATING_METADATA

FINALIZING_MINT
```

Only after those stages does the SDK reach:

```text
EXECUTING_DEV_BUY
```

## 💸 Execute the Dev Buy

The later execution request is:

```text
POST /launch/{mint}/devbuy-escrow
```

The SDK supplies:

```text
minTokensOut
```

from the earlier Dev Buy quote.

## 🛡️ The Same Minimum Is Preserved

This is an important part of the flow.

The minimum calculated during Dev Buy quoting is used when preparing escrow funding and again when executing the Dev Buy.

Conceptually:

```text
Dev Buy quote
     ↓
minTokensOut
     ↓
tx0 preparation
     ↓
Creator funding
     ↓
Launch stages
     ↓
devbuy-escrow
     ↓
minTokensOut
```

The creation flow therefore carries the protected output through the launch sequence.

## ✍️ No Second Creator Signing Call Here

The public `createToken()` implementation does not call the supplied `signTransaction` again when it reaches:

```text
/devbuy-escrow
```

The creator signing callback was used for the earlier escrow funding transaction.

The later Dev Buy execution is requested as part of the Moonz launch orchestration.

## 📡 Dev Buy Signature

The Dev Buy execution returns a transaction signature.

That signature becomes:

```ts
result.signatures.devBuy
```

in a successful creation result.

## 🌕 Settlement Follows

After Dev Buy execution, the next progress step is:

```text
SETTLING
```

The SDK requests:

```text
POST /launch/{mint}/settle
```

The resulting settlement signature becomes:

```ts
result.signatures.settle
```

After successful settlement, the public progress state reaches:

```text
LIVE
```

## ✅ Dev Buy Values in the Final Result

A successful `CreateTokenResult` exposes:

```ts
result.devBuyLamports

result.expectedTokensOut

result.minTokensOut
```

These values allow the application to retain the important Dev Buy context after launch.

## 🧾 Funding Signature

The creator signed escrow transaction is returned as:

```ts
result.signatures.fund
```

This is the actual signature produced when the signed `tx0` transaction is submitted through the SDK's Solana connection.

## 🧾 Dev Buy Signature

Later Dev Buy execution is returned separately as:

```ts
result.signatures.devBuy
```

These signatures refer to different stages.

```text
fund
   ↓
Creator funded launch escrow

devBuy
   ↓
Dev Buy executed from
the launch flow
```

## 🔎 Example Result Inspection

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
      wallet.signTransaction
  });

console.log({
  devBuyLamports:
    result.devBuyLamports,

  expectedTokensOut:
    result.expectedTokensOut,

  minTokensOut:
    result.minTokensOut,

  fundSignature:
    result.signatures.fund,

  devBuySignature:
    result.signatures.devBuy
});
```

## 📡 Progress During Dev Buy Preparation

The relevant progress sequence is:

```text
QUOTING_DEV_BUY
      ↓
BUILDING_ESCROW_TX
      ↓
AWAITING_CREATOR_SIGNATURE
      ↓
SENDING_ESCROW_TX
      ↓
CONFIRMING_ESCROW
```

Later:

```text
EXECUTING_DEV_BUY
      ↓
SETTLING
      ↓
LIVE
```

This gives a creation interface enough information to clearly separate wallet authorization from later launch execution.

## 🖥️ Useful Creator Messages

An application can map these public states into user facing messages.

For example:

```text
QUOTING_DEV_BUY
Calculating your Dev Buy

BUILDING_ESCROW_TX
Preparing launch funding

AWAITING_CREATOR_SIGNATURE
Approve launch funding in your wallet

SENDING_ESCROW_TX
Submitting launch funding

CONFIRMING_ESCROW
Confirming launch funding

EXECUTING_DEV_BUY
Executing Dev Buy

SETTLING
Finalizing launch

LIVE
Token is live
```

The exact wording is up to the application.

## 🧠 Do Not Build a Separate Dev Buy Guess

The public creation flow already obtains the Moonz Dev Buy quote and carries its minimum output into the launch sequence.

An integration using `createToken()` should not independently calculate a different Dev Buy output and assume it is authoritative.

Use:

```ts
result.expectedTokensOut

result.minTokensOut
```

for the values returned by the completed creation flow.

## 🔁 Failure After Mint Reservation

By the Dev Buy stage, the mint is already known.

If the creation sequence fails, the SDK wraps the error with the reserved mint:

```text
Moonz createToken failed for {mint}:
```

That gives the application a concrete launch identifier for diagnostics or recovery handling.

## 🌌 Dev Buy Journey

The complete Dev Buy path inside creation is:

```text
devBuySol
    ↓
Convert to lamports
    ↓
slippageBps
    ↓
Dev Buy quote
    ↓
expectedTokensOut
    ↓
minTokensOut
    ↓
Build tx0
    ↓
Creator signs
    ↓
Submit escrow funding
    ↓
Confirm
    ↓
Report funding signature
to Moonz
    ↓
Initialize launch
    ↓
Create metadata
    ↓
Finalize mint
    ↓
Execute Dev Buy
from escrow
    ↓
Settle
    ↓
LIVE
```

{% hint style="success" %}
The creator authorizes the SOL funding once through the wallet.

The Moonz creation flow carries the quoted minimum output through escrow and into Dev Buy execution.
{% endhint %}

## 🧾 Next Stop

We now understand the creator funding path.

Next we will map every launch stage from `tx0` through settlement and show what each returned signature represents.

Next stop:

**Launch Sequence**
