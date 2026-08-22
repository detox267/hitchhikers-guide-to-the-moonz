# 🟢 Buy Moonz

We have the quote.

Now we can build the trade.

The Moonz Trading SDK gives you two public ways to buy:

```text
buy()

buildBuyTransaction()
```

Use `buy()` when you want the SDK to handle the complete wallet flow.

Use `buildBuyTransaction()` when your application wants control over signing or submission.

{% hint style="info" %}
Both methods automatically choose the correct Moonz trading route from current Solana state.
{% endhint %}

## 🚀 The Fastest Buy

The simplest complete buy is:

```ts
const result =
  await moonz.buy({
    mint: "TOKEN_MINT",
    wallet,
    amount: "0.1",
    slippageBps: 100
  });

console.log(
  result.signature
);
```

That single call performs the full trade flow.

```text
Read Moonz state
      ↓
Quote
      ↓
Choose route
      ↓
Build transaction
      ↓
Wallet signs
      ↓
Submit to Solana
      ↓
Confirm
      ↓
Return result
```

## 👛 Wallet Requirements

The high level buy method expects a wallet compatible with:

```ts
interface MoonzWalletSigner {
  publicKey:
    | PublicKey
    | null;

  signTransaction?: (
    transaction: Transaction
  ) => Promise<Transaction>;
}
```

The wallet must provide:

```text
publicKey

signTransaction()
```

If the wallet is not connected, the SDK rejects the trade.

If the wallet cannot sign a transaction, the SDK rejects the trade.

## 🔐 No Private Key

The SDK never asks for a private key.

The signing sequence is:

```text
Moonz builds transaction
        ↓
Your wallet receives transaction
        ↓
Wallet signs locally
        ↓
Signed transaction returns
        ↓
SDK submits serialized transaction
```

Moonz does not take custody of the wallet.

{% hint style="success" %}
The application provides a signing interface, not a private key.
{% endhint %}

## 🛣️ Automatic Buy Routing

Your application calls the same public method regardless of the token's current trade route.

The SDK selects:

```text
BONDING
    ↓
buy

AMM_LIVE + SOL
    ↓
buy

AMM_LIVE + USDC
    ↓
amm_buy_usdc
```

You do not need to manually select `buy` or `amm_buy_usdc`.

## 🧭 How Routing Works

Before constructing the transaction, the SDK reads the token's Launch State.

Conceptually:

```text
Token mint
    ↓
Read Launch State
    ↓
AMM_LIVE + USDC?
    |
    ├── YES
    |     ↓
    | amm_buy_usdc
    |
    └── NO
          ↓
       SOL path
          ↓
   Validate BONDING
   or AMM_LIVE + SOL
```

The selected builder then performs its own state validation before constructing the transaction.

## 🌱 Bonding Buy

While a Moonz token is in:

```text
BONDING
```

the buy route is:

```text
buy
```

and the quote asset is SOL.

For example:

```ts
const result =
  await moonz.buy({
    mint,
    wallet,
    amount: "0.25",
    slippageBps: 100
  });
```

Here:

```text
0.25
```

means:

```text
0.25 SOL
```

## 🚀 SOL AMM Buy

If the token has reached:

```text
AMM_LIVE
```

and its active quote asset is:

```text
SOL
```

the SDK still uses the canonical:

```text
buy
```

instruction.

The calling interface does not change.

```ts
await moonz.buy({
  mint,
  wallet,
  amount: "0.25",
  slippageBps: 100
});
```

## 💵 USDC AMM Buy

If the current state is:

```text
AMM_LIVE + USDC
```

the Trading SDK automatically uses:

```text
amm_buy_usdc
```

For example:

```ts
await moonz.buy({
  mint,
  wallet,
  amount: "25.50",
  slippageBps: 100
});
```

In this market:

```text
25.50
```

means:

```text
25.50 USDC
```

The public call remains `buy()`.

## 💬 The Buy Is Quoted Again

The transaction builder does not rely on an old quote your interface may have displayed earlier.

During construction it calls the current buy quote logic again.

Conceptually:

```text
User saw quote
      ↓
Time passes
      ↓
buildBuyTransaction()
      ↓
Read current state again
      ↓
Create fresh quote
      ↓
Build using fresh minimum output
```

The quote returned inside the built transaction is the quote actually associated with that construction.

## 🛡️ Minimum Output Goes On Chain

The builder takes:

```ts
quote.minTokensOutRaw
```

and encodes it into the Moonz buy instruction.

For the SOL route, instruction data contains:

```text
buy discriminator

amountIn

minTokensOut
```

For the USDC route:

```text
amm_buy_usdc discriminator

usdcIn

minTokensOut
```

This is how the quoted slippage protection becomes part of the actual transaction.

## 🧱 Build Without Signing

If your application wants the unsigned transaction, use:

```ts
const built =
  await moonz.buildBuyTransaction({
    mint: "TOKEN_MINT",
    buyer: wallet.publicKey,
    amount: "0.1",
    slippageBps: 100
  });
```

Nothing is signed.

Nothing is sent.

The result gives you the transaction and its construction context.

## 📦 Built Buy Result

`MoonzBuiltBuyTransaction` contains:

```ts
{
  transaction,
  quote,
  instruction,
  blockhash,
  lastValidBlockHeight,
  accounts
}
```

The `transaction` is a Solana `Transaction`.

The `quote` is the fresh `MoonzBuyQuote` used during construction.

## 🛰️ Selected Instruction

Inspect:

```ts
built.instruction
```

The value will be:

```text
buy

or

amm_buy_usdc
```

That tells you which canonical Moonz route was selected.

## ⛓️ Blockhash Information

The builder fetches a recent Solana blockhash and returns:

```ts
built.blockhash

built.lastValidBlockHeight
```

The transaction itself is also populated with:

```text
fee payer

recent blockhash
```

The buyer is the fee payer.

## 🌙 Resolved Accounts

The built result exposes the account addresses used during transaction construction.

Common fields include:

```text
buyer

launchState

saleVault

lpVault

buyerTokenAta

buyerWsolAta

treasuryWsolVault

creatorFeeAuthority
```

Route specific account information is also returned.

This is particularly useful for:

```text
Debugging

Transaction inspection

Testing

Integration diagnostics
```

## 🟣 The SOL Transaction

For a SOL buy, the SDK constructs a transaction that performs the required WSOL handling for the user.

The current sequence is:

```text
1
Ensure buyer token ATA exists

2
Ensure buyer WSOL ATA exists

3
Ensure platform WSOL ATA exists

4
Transfer requested native SOL
into buyer WSOL ATA

5
Sync native WSOL balance

6
Execute Moonz buy

7
Close buyer WSOL ATA
back to buyer
```

The associated token account creation instructions are idempotent.

That means they can safely ensure the required account exists without requiring your application to determine whether it already exists first.

## 🌊 Why WSOL Appears

The user enters a human SOL amount.

The Moonz program trade operates through WSOL token accounts.

So the transaction handles the bridge:

```text
Native SOL
    ↓
Buyer WSOL ATA
    ↓
syncNative
    ↓
Moonz buy
    ↓
Close WSOL ATA
    ↓
Remaining native SOL
returns to buyer
```

Your application does not need to manually wrap SOL before calling the public buy method.

## 🧾 Buyer Token Account

The SOL transaction ensures the buyer has an associated token account for the Moonz mint.

That is:

```text
buyerTokenAta
```

The purchased Moonz tokens can therefore be delivered through the same transaction flow.

## 💵 The USDC Transaction

The USDC path is different.

There is no native SOL wrapping step for the trade input.

The current transaction prepares:

```text
Buyer Moonz token ATA

Buyer USDC ATA

Creator fee USDC vault

Platform USDC ATA
```

then executes:

```text
amm_buy_usdc
```

## 🧱 USDC Account Preparation

The current USDC builder uses idempotent associated token account creation for:

```text
buyerTokenAta

buyerUsdcAta

creatorFeeUsdcVault

platformUsdcAta
```

Then it adds the Moonz USDC AMM buy instruction.

## 💳 Buyer Must Have USDC

Creating the buyer USDC associated token account does not fund it.

For a USDC market, the buyer still needs sufficient USDC to execute the requested trade.

Conceptually:

```text
Buyer USDC balance
      ↓
amm_buy_usdc
      ↓
Moonz tokens
```

If the required funds are not available, the Solana transaction cannot complete successfully.

## 🎯 State Is Validated

The builders do not blindly trust the mint supplied by the application.

They verify current Moonz state.

For example, the SOL path checks that the state is:

```text
BONDING

or

AMM_LIVE + SOL
```

The USDC path requires:

```text
AMM_LIVE + USDC
```

The Launch State mint must also match the requested mint.

## 🚫 Non Tradeable State

If the token is not in a supported trade phase, the builder rejects the operation.

For the SOL path, valid phases are:

```text
BONDING

AMM_LIVE
```

and an `AMM_LIVE` token must currently use SOL.

The USDC path specifically requires `AMM_LIVE` with USDC.

## 🛑 Zero Output Protection

The builder refuses to construct a transaction when the quote produces:

```text
zero input

or

zero minimum token output
```

That protects the transaction construction layer from producing a meaningless Moonz instruction.

## 🌉 A Buy Can Cross Into the AMM

The quote returned from the builder can indicate:

```ts
built.quote.crossesCurve
```

and:

```ts
built.quote.migratesToAmm
```

So after building you can still determine whether the SOL buy:

```text
remains entirely in bonding

or

finishes bonding and crosses
into the AMM
```

The transaction construction remains automatic.

## 👛 Sign a Built Transaction Yourself

If you use the lower level builder, your application controls signing.

For example:

```ts
const built =
  await moonz.buildBuyTransaction({
    mint,
    buyer: wallet.publicKey,
    amount: "0.1",
    slippageBps: 100
  });

const signed =
  await wallet.signTransaction(
    built.transaction
  );
```

At this point the SDK has not submitted anything.

Your application decides what happens next.

## 📡 Submit It Yourself

A custom submission flow can use your Solana connection:

```ts
const signature =
  await connection.sendRawTransaction(
    signed.serialize()
  );
```

Then confirmation can use the builder's blockhash context:

```ts
const confirmation =
  await connection.confirmTransaction(
    {
      signature,
      blockhash:
        built.blockhash,
      lastValidBlockHeight:
        built.lastValidBlockHeight
    },
    "confirmed"
  );
```

This is useful when your application has its own:

```text
Transaction queue

RPC policy

Telemetry

Retry handling

Submission service
```

## 🚀 What buy() Does For You

The high level `buy()` method performs that flow internally.

It:

```text
Checks wallet connection

Checks signTransaction exists

Builds fresh transaction

Requests wallet signature

Serializes signed transaction

Calls sendRawTransaction

Confirms using blockhash context

Checks confirmation error

Returns MoonzBuyResult
```

So for most wallet applications, `buy()` is the shortest integration path.

## ✅ Successful Buy Result

A successful high level buy returns:

```ts
interface MoonzBuyResult {
  signature: string;
  confirmationSlot: number;

  instruction:
    | "buy"
    | "amm_buy_usdc";

  quote: MoonzBuyQuote;
}
```

That gives your application both transaction identity and trade context.

## 🔎 Example Result Handling

```ts
const result =
  await moonz.buy({
    mint,
    wallet,
    amount: "0.1",
    slippageBps: 100
  });

console.log(
  "Signature:",
  result.signature
);

console.log(
  "Confirmed slot:",
  result.confirmationSlot
);

console.log(
  "Instruction:",
  result.instruction
);

console.log(
  "Quote asset:",
  result.quote.quoteAsset
);

console.log(
  "Expected tokens:",
  result.quote.expectedTokensOut
);
```

## ❌ Failed Confirmation

Submitting a transaction is not the same as successfully executing it.

After submission, `buy()` checks the Solana confirmation result.

If:

```ts
confirmation.value.err
```

contains an error, the SDK throws instead of returning a successful `MoonzBuyResult`.

That distinction is important for application state.

Do not mark the trade complete merely because a transaction signature was created.

## 🖥️ A Practical Buy Button

A user interface can follow this pattern:

```text
User enters amount
      ↓
quoteBuy()
      ↓
Display expected output
      ↓
Display minimum output
      ↓
Display quote asset
      ↓
User presses Buy
      ↓
buy()
      ↓
Wallet approval
      ↓
Solana confirmation
      ↓
Show success
```

If the user changes the amount, refresh the quote.

## 🧯 Handle Wallet Rejection

A wallet can decline the signature request.

Your application should treat that as a normal user outcome.

Conceptually:

```ts
try {
  const result =
    await moonz.buy({
      mint,
      wallet,
      amount,
      slippageBps: 100
    });

  showSuccess(
    result.signature
  );
} catch (error) {
  showTradeError(
    error
  );
}
```

Do not assume every thrown error means the Moonz program failed.

Possible failure points include:

```text
Invalid input

Token state changed

Missing liquidity

Wallet unavailable

Wallet rejected signature

RPC submission failed

Solana transaction failed
```

## 🔄 State Can Change Before Signing

Remember that Moonz is live.

Between displaying a quote and signing the transaction:

```text
Another trade may execute

Bonding may finish

Reserves may move

The token may migrate
```

The builder rereads current state and produces a fresh quote when constructing the transaction.

Minimum output protection then guards execution.

## 🌌 The Complete Buy Path

The full public Moonz buy journey now looks like:

```text
TOKEN MINT
    ↓
quoteBuy()
    ↓
Preview trade
    ↓
buy()
    ↓
Read current state again
    ↓
Automatic route
    |
    ├───────────────┐
    ↓               ↓
SOL route        USDC route
    |               |
    ↓               ↓
buy          amm_buy_usdc
    |               |
    └───────┬───────┘
            ↓
      Build transaction
            ↓
       Wallet signs
            ↓
      Submit to Solana
            ↓
         Confirm
            ↓
     MoonzBuyResult
```

{% hint style="success" %}
Your application asks to buy Moonz.

The SDK handles which Moonz market it actually needs to use.
{% endhint %}

## 🔴 Next Stop

Buying gets you aboard.

Eventually somebody wants to leave with the quote asset.

Next stop:

**Sell Moonz**
