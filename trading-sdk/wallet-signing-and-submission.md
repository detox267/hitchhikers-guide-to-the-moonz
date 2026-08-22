# 👛 Wallet Signing and Submission

A Moonz transaction can be perfectly constructed and still do absolutely nothing.

Until the wallet signs it.

The Trading SDK keeps transaction construction and wallet authority separate.

Moonz builds the transaction.

The wallet authorizes it.

Solana executes it.

{% hint style="success" %}
Moonz never needs the user's private key.

The wallet signs locally.
{% endhint %}

## 🔐 The Wallet Contract

The high level trading methods use a deliberately small wallet interface.

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

That gives Moonz exactly what it needs:

```text
Wallet public key

Transaction signing function
```

Nothing more.

## 🚫 No Private Key Input

The Trading SDK does not accept:

```text
Private key

Secret key

Seed phrase

Recovery phrase
```

The wallet keeps custody.

The SDK receives only the signed transaction returned by the wallet.

## 👛 Wallet Connection

Before a high level buy or sell can continue, the SDK checks:

```ts
wallet.publicKey
```

If it is missing or null, the SDK throws:

```text
Wallet is not connected
```

Your interface should therefore establish wallet connection before initiating a trade.

## ✍️ Signing Support

The SDK also checks:

```ts
wallet.signTransaction
```

If it is unavailable, the trade cannot continue through the high level method.

The SDK throws:

```text
Wallet does not support signTransaction
```

## 🟢 High Level Buy

The shortest buy flow is:

```ts
const result =
  await moonz.buy({
    mint,
    wallet,
    amount: "0.1",
    slippageBps: 100
  });
```

Inside that call, Moonz performs:

```text
Build
  ↓
Wallet sign
  ↓
Serialize
  ↓
Submit
  ↓
Confirm
```

## 🔴 High Level Sell

The sell flow is equivalent:

```ts
const result =
  await moonz.sell({
    mint,
    wallet,
    amount: "100000",
    slippageBps: 100
  });
```

Again, the wallet signs the transaction locally.

## 🧱 Construction Happens First

Before asking the wallet to sign, the high level methods call the corresponding builder.

For buys:

```text
buy()
   ↓
buildBuyTransaction()
```

For sells:

```text
sell()
   ↓
buildSellTransaction()
```

That means the wallet receives the fully constructed Moonz transaction.

## 💬 The Builder Uses Current State

The builder rereads current Moonz state before constructing the transaction.

So the signing flow is conceptually:

```text
User saw quote
      ↓
User presses trade
      ↓
Builder reads current state
      ↓
Fresh quote
      ↓
Current route
      ↓
Protected transaction
      ↓
Wallet signs
```

The wallet is authorizing the current transaction, not an old interface assumption.

## ✍️ The Signing Call

The SDK invokes:

```ts
const signed =
  await wallet.signTransaction(
    built.transaction
  );
```

The wallet is expected to return a signed Solana `Transaction`.

## 🛡️ Signed Transaction Validation

After the wallet returns, the SDK checks that the returned value exists and supports:

```ts
signed.serialize()
```

If not, it throws:

```text
Wallet returned an invalid signed transaction
```

This prevents the SDK from attempting to submit an invalid wallet response.

## 📡 Submission

Once the transaction is signed, the high level trade methods call:

```ts
this.connection.sendRawTransaction(
  signed.serialize()
)
```

The result is a Solana transaction signature.

Conceptually:

```text
Signed Transaction
       ↓
serialize()
       ↓
sendRawTransaction()
       ↓
Signature
```

## ⚠️ A Signature Is Not Final Success

A transaction signature means the transaction has been submitted.

It does not prove that the Moonz trade executed successfully.

The SDK therefore continues to confirmation.

## ⛓️ Confirmation Context

The transaction builder previously returned:

```ts
built.blockhash

built.lastValidBlockHeight
```

The high level methods use those values when confirming.

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
    commitment
  );
```

This ties confirmation to the blockhash context used when the transaction was built.

## ✅ Confirmation Success

If Solana confirms the transaction without an execution error, the SDK returns the Moonz trade result.

For buys:

```ts
MoonzBuyResult
```

For sells:

```ts
MoonzSellResult
```

## ❌ Confirmation Failure

The SDK explicitly checks:

```ts
confirmation.value.err
```

If an execution error exists, the high level method throws.

For a buy the error begins with:

```text
Moonz buy transaction failed:
```

For a sell:

```text
Moonz sell transaction failed:
```

The failed transaction is not returned as a successful trade result.

## 🛰️ Confirmation Slot

A successful trade result includes:

```ts
confirmationSlot
```

This comes from:

```ts
confirmation.context.slot
```

That gives your application the Solana slot associated with confirmation.

## 🟢 Buy Result

A successful buy returns:

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

So you retain:

```text
Transaction signature

Confirmation slot

Selected Moonz instruction

Quote used during construction
```

## 🔴 Sell Result

A successful sell returns:

```ts
interface MoonzSellResult {
  signature: string;

  confirmationSlot: number;

  instruction:
    | "sell"
    | "amm_sell_usdc";

  quote: MoonzSellQuote;
}
```

The result follows the same pattern.

## 🛣️ Route Information Survives Submission

The SDK preserves the selected instruction in the final result.

For a buy:

```ts
result.instruction
```

can be:

```text
buy

amm_buy_usdc
```

For a sell:

```text
sell

amm_sell_usdc
```

This is useful for transaction history and diagnostics.

## 💬 The Quote Survives Too

The result also contains:

```ts
result.quote
```

This is useful because it preserves the trade context associated with the transaction construction.

For example:

```ts
console.log(
  result.quote.quoteAsset
);

console.log(
  result.quote.slippageBps
);
```

For a buy:

```ts
console.log(
  result.quote.expectedTokensOut
);

console.log(
  result.quote.minTokensOut
);
```

For a sell:

```ts
console.log(
  result.quote.expectedQuoteOut
);

console.log(
  result.quote.minQuoteOut
);
```

## 🧰 Custom Submission

You do not have to use the high level trade methods.

If your application already has its own transaction infrastructure, build first:

```ts
const built =
  await moonz.buildBuyTransaction({
    mint,
    buyer: wallet.publicKey,
    amount,
    slippageBps: 100
  });
```

Then sign:

```ts
const signed =
  await wallet.signTransaction(
    built.transaction
  );
```

Then submit however your application requires.

## 📡 Standard Custom Submission

A basic custom submission path can be:

```ts
const signature =
  await connection.sendRawTransaction(
    signed.serialize()
  );
```

Then:

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

## 🛑 Check the Result

If you manage confirmation yourself, check:

```ts
confirmation.value.err
```

before recording the trade as successful.

For example:

```ts
if (
  confirmation.value.err
) {
  throw new Error(
    "Transaction failed"
  );
}
```

## 🧠 Submission Policy Is Yours

The lower level builders are useful when your application has its own:

```text
RPC provider strategy

Retry policy

Transaction telemetry

Simulation layer

Confirmation strategy

Queue

Monitoring
```

The SDK constructs the Moonz transaction.

Your infrastructure can own everything after that.

## ⏳ Blockhash Expiry

A Solana transaction does not remain valid forever.

The builder returns:

```ts
built.lastValidBlockHeight
```

If the transaction waits too long before submission, its blockhash can expire.

The correct response is usually to rebuild.

```text
Transaction expired
      ↓
Build again
      ↓
Fresh state
      ↓
Fresh quote
      ↓
Fresh blockhash
      ↓
Sign again
```

## 🔄 Why Rebuild Matters

Rebuilding does more than refresh the blockhash.

It also refreshes:

```text
Moonz state

Market route

Reserves

Quote

Minimum output
```

That is important in an active market.

## 👛 Wallet Rejection Is Normal

A user can reject the wallet signing request.

Your application should treat that as a normal possible outcome.

For example:

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

A rejected signature does not mean the Moonz program failed.

The transaction may never have been submitted.

## 🧯 Separate Failure Stages

A trade can fail at different stages.

```text
INPUT
  ↓
Could fail validation

STATE READ
  ↓
Could fail RPC read

QUOTE
  ↓
Could fail market validation

BUILD
  ↓
Could fail account or state checks

SIGN
  ↓
User may reject

SUBMIT
  ↓
RPC may reject submission

CONFIRM
  ↓
Transaction may fail execution
```

A useful interface should avoid treating all of these as the same error.

## 🔍 Useful Error Categories

Applications can generally distinguish between:

```text
Wallet not connected

Wallet cannot sign

Wallet rejected

Invalid amount

Token not tradeable

Insufficient liquidity

RPC failure

Transaction failure
```

The exact user messaging is an application decision.

## 🔐 Custody Boundary

The most important security boundary is simple.

```text
MOONZ
  ↓
Build transaction

WALLET
  ↓
Authorize transaction

SOLANA
  ↓
Execute transaction
```

Moonz does not need access to the private key at any point.

## 🧱 Builder Versus High Level Trade

Use:

```ts
buy()

sell()
```

when you want:

```text
Build

Sign

Submit

Confirm
```

in one public SDK operation.

Use:

```ts
buildBuyTransaction()

buildSellTransaction()
```

when you want:

```text
Moonz transaction construction

plus

your own wallet and submission pipeline
```

## 🖥️ Recommended User Flow

A clean wallet interface can follow:

```text
Connect wallet
      ↓
Enter amount
      ↓
Quote trade
      ↓
Display expected output
      ↓
Display minimum output
      ↓
Display SOL or USDC
      ↓
User confirms
      ↓
Build current transaction
      ↓
Wallet approval
      ↓
Submit
      ↓
Confirm
      ↓
Show signature
```

## 🌌 Complete Trading Journey

The complete public Trading SDK flow is now:

```text
TOKEN MINT
    ↓
Read current state
    ↓
Quote
    ↓
Determine market
    ↓
Determine SOL or USDC
    ↓
Apply fee
    ↓
Apply slippage
    ↓
Minimum output
    ↓
Build transaction
    ↓
Resolve accounts
    ↓
Recent blockhash
    ↓
Wallet signs locally
    ↓
Submit to Solana
    ↓
Confirm
    ↓
Check execution result
    ↓
Return signature
and trade context
```

{% hint style="success" %}
Moonz owns protocol construction.

The wallet owns authorization.

Solana owns execution.
{% endhint %}

## 🚀 Where Next?

We can now quote, route, build, sign, submit, buy and sell Moonz.

But trading is only half of the public SDK.

The same package can also launch a token.

Next section:

**Token Creation**
