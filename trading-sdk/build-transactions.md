# 🧱 Build Transactions

Sometimes you want Moonz to handle the whole trade.

Sometimes you want the transaction first.

The Trading SDK supports both.

For applications that need control over signing, submission or inspection, use:

```text
buildBuyTransaction()

buildSellTransaction()
```

These methods construct unsigned Solana transactions from current Moonz state.

Nothing is signed.

Nothing is submitted.

{% hint style="success" %}
The builder prepares the transaction.

Your application decides what happens next.
{% endhint %}

## 🟢 Build a Buy

A buy transaction can be constructed with:

```ts
const built =
  await moonz.buildBuyTransaction({
    mint: "TOKEN_MINT",
    buyer: wallet.publicKey,
    amount: "0.1",
    slippageBps: 100
  });
```

The result contains:

```ts
built.transaction
```

which is a standard Solana `Transaction`.

## 🔴 Build a Sell

A sell is similar:

```ts
const built =
  await moonz.buildSellTransaction({
    mint: "TOKEN_MINT",
    seller: wallet.publicKey,
    amount: "100000",
    slippageBps: 100
  });
```

Again:

```ts
built.transaction
```

is unsigned.

## 🌌 What the Builder Does

Building is more than assembling one instruction.

The SDK performs the protocol preparation required for the current market.

Conceptually:

```text
Input
  ↓
Read current Moonz state
  ↓
Quote current market
  ↓
Validate phase
  ↓
Validate quote asset
  ↓
Resolve protocol accounts
  ↓
Resolve wallet token accounts
  ↓
Choose canonical instruction
  ↓
Encode protected minimum output
  ↓
Prepare supporting instructions
  ↓
Fetch recent blockhash
  ↓
Return unsigned transaction
```

## 🛣️ Routing Is Still Automatic

Using the builder does not mean you need to choose the Moonz route yourself.

For buys:

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

For sells:

```text
BONDING
    ↓
sell

AMM_LIVE + SOL
    ↓
sell

AMM_LIVE + USDC
    ↓
amm_sell_usdc
```

The same route logic used by the high level trade methods is used by the transaction builders.

## 🔎 State Is Read Before Construction

The builder reads the current Moonz Launch State.

That matters because your interface may have displayed a quote several seconds earlier.

The chain may have changed since then.

```text
Old interface quote
      ↓
Time passes
      ↓
Builder starts
      ↓
Read current state
      ↓
Fresh quote
      ↓
Current route
      ↓
Transaction
```

## 💬 The Builder Creates Its Own Quote

Every built transaction contains:

```ts
built.quote
```

That quote is generated during construction from current Solana state.

For a buy:

```ts
console.log(
  built.quote.expectedTokensOut
);

console.log(
  built.quote.minTokensOut
);
```

For a sell:

```ts
console.log(
  built.quote.expectedQuoteOut
);

console.log(
  built.quote.minQuoteOut
);
```

## 🛡️ Minimum Output Is Encoded

The builder does not simply return a quote beside the transaction.

It uses the quote's raw minimum output to construct the Moonz instruction.

For buys:

```ts
built.quote.minTokensOutRaw
```

becomes the protected minimum token output.

For sells:

```ts
built.quote.minQuoteOutRaw
```

becomes the protected minimum quote output.

## 🔢 Raw Protocol Values

The Moonz program works with integer base units.

So transaction construction uses raw amounts such as:

```text
amountInRaw

tokensInRaw

minTokensOutRaw

minQuoteOutRaw
```

The human readable fields remain useful for user interfaces.

The raw fields are what matter for exact protocol encoding.

## 📦 Built Buy Structure

The public buy builder returns:

```ts
interface MoonzBuiltBuyTransaction {
  transaction: Transaction;

  quote: MoonzBuyQuote;

  instruction:
    | "buy"
    | "amm_buy_usdc";

  blockhash: string;

  lastValidBlockHeight: number;

  accounts: {
    ...
  };
}
```

The important pieces are:

```text
transaction

quote

instruction

blockhash

lastValidBlockHeight

accounts
```

## 📦 Built Sell Structure

The sell builder returns the same type of construction context:

```ts
interface MoonzBuiltSellTransaction {
  transaction: Transaction;

  quote: MoonzSellQuote;

  instruction:
    | "sell"
    | "amm_sell_usdc";

  blockhash: string;

  lastValidBlockHeight: number;

  accounts: {
    ...
  };
}
```

This makes custom trade pipelines straightforward.

## 🛰️ Inspect the Selected Instruction

After building:

```ts
console.log(
  built.instruction
);
```

A buy returns:

```text
buy

or

amm_buy_usdc
```

A sell returns:

```text
sell

or

amm_sell_usdc
```

You can log this for diagnostics without choosing it manually.

## ⛓️ Recent Blockhash

Before returning the transaction, the builder asks Solana for a recent blockhash.

The transaction receives:

```text
recentBlockhash
```

and the result separately exposes:

```ts
built.blockhash
```

plus:

```ts
built.lastValidBlockHeight
```

These values are important when confirming a submitted transaction.

## 👛 Fee Payer

The transaction fee payer is the trading wallet.

For a buy:

```text
fee payer = buyer
```

For a sell:

```text
fee payer = seller
```

The SDK assigns this during construction.

## 🟣 SOL Buy Construction

A SOL buy currently prepares:

```text
Buyer Moonz ATA

Buyer WSOL ATA

Platform WSOL ATA

Native SOL transfer

WSOL sync

Moonz buy instruction

WSOL account close
```

The sequence allows the caller to provide native SOL while the protocol instruction operates through token accounts.

## 🌊 SOL Wrapping

The builder handles the native SOL conversion inside the transaction.

```text
Native SOL
    ↓
buyerWsolAta
    ↓
syncNative
    ↓
Moonz buy
```

The caller does not need to pre wrap SOL.

## 🔁 SOL Buy Cleanup

After the Moonz buy instruction, the transaction closes:

```text
buyerWsolAta
```

back to the buyer.

That returns remaining lamports from the wrapped SOL account.

## 🔴 SOL Sell Construction

The SOL sell builder prepares:

```text
Seller Moonz ATA

Seller WSOL ATA

Platform WSOL ATA

Moonz sell instruction

WSOL account close
```

The protocol returns WSOL.

The final account close converts that output back into native SOL for the seller.

## 💵 USDC Buy Construction

For `AMM_LIVE + USDC`, the buy builder prepares:

```text
buyerTokenAta

buyerUsdcAta

creatorFeeUsdcVault

platformUsdcAta
```

then executes:

```text
amm_buy_usdc
```

No WSOL wrapping is needed for the trade input.

## 💵 USDC Sell Construction

For the USDC sell route, the builder prepares:

```text
sellerTokenAta

sellerUsdcAta

creatorFeeUsdcVault

platformUsdcAta
```

then executes:

```text
amm_sell_usdc
```

The resulting user output is sent through the seller's USDC token account.

## 🧾 Associated Token Accounts

Moonz builders use idempotent associated token account creation instructions where required.

That means the transaction can safely request the account whether or not it already exists.

This is used for wallet and fee token accounts required by the selected route.

{% hint style="info" %}
Idempotent account preparation keeps integrations from needing a separate account existence check before every trade.
{% endhint %}

## 🧬 Account Resolution

The builder returns the addresses it resolved.

For a buy, common fields include:

```text
buyer

launchState

saleVault

lpVault

buyerTokenAta

buyerWsolAta

treasuryWsolVault

creatorFeeAuthority

creatorFeeWsolVault

platformWsolAta
```

USDC route information can additionally include:

```text
buyerUsdcAta

treasuryUsdcVault

creatorFeeUsdcVault

platformUsdcAta
```

## 🔴 Sell Account Resolution

A built sell exposes fields such as:

```text
seller

launchState

saleVault

lpVault

sellerTokenAta

sellerWsolAta

treasuryWsolVault

creatorFeeAuthority

creatorFeeWsolVault

platformWsolAta
```

USDC routes can additionally expose:

```text
sellerUsdcAta

treasuryUsdcVault

creatorFeeUsdcVault

platformUsdcAta
```

## 🔍 Why Return Accounts?

The account map is useful when building:

```text
Developer tools

Transaction inspectors

Debugging interfaces

Integration tests

Execution logs

Support tooling
```

For example:

```ts
console.log(
  built.accounts.launchState
);

console.log(
  built.accounts.lpVault
);
```

## 🛡️ Builders Validate State

The builder does not merely trust an address list.

It validates Moonz state before producing the transaction.

Examples include:

```text
Launch State exists

Mint matches Launch State

Trade phase is supported

Quote asset matches route

Required reserves exist

Input is positive

Protected minimum is positive
```

If those conditions are not satisfied, construction fails.

## 🚫 Nothing Is Signed

This is the key boundary.

After:

```ts
const built =
  await moonz.buildBuyTransaction(...)
```

the transaction has not been authorized by the wallet.

The buyer or seller must still sign it.

## 🚫 Nothing Is Submitted

The builder also does not call:

```ts
sendRawTransaction()
```

It returns the unsigned transaction to you.

That makes it suitable for applications with their own submission architecture.

## 👛 Custom Wallet Flow

A common custom flow is:

```ts
const built =
  await moonz.buildBuyTransaction({
    mint,
    buyer: wallet.publicKey,
    amount,
    slippageBps: 100
  });

const signed =
  await wallet.signTransaction(
    built.transaction
  );
```

At that point the wallet has authorized the transaction.

Submission is still under application control.

## 📡 Custom Submission

You can then submit through your own Solana connection:

```ts
const signature =
  await connection.sendRawTransaction(
    signed.serialize()
  );
```

And confirm with:

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

## 🔎 Always Check Confirmation

A returned transaction signature means the transaction was submitted.

It does not by itself prove successful execution.

Inspect:

```ts
confirmation.value.err
```

before treating the trade as successful.

## 🧰 Why Use the Builder?

Use the builder when your application needs things such as:

```text
Custom RPC submission

Custom confirmation

Transaction inspection

Additional telemetry

Custom wallet orchestration

Transaction simulation

Testing

Special transaction pipelines
```

If you do not need that level of control, the high level `buy()` and `sell()` methods are simpler.

## ⚡ High Level Versus Builder

The difference is:

```text
buy()
sell()

Quote
Build
Sign
Submit
Confirm

        versus

buildBuyTransaction()
buildSellTransaction()

Quote
Build
Return unsigned transaction
```

Choose the level that fits your integration.

## 🧪 Inspect Before Signing

Since the builder returns the transaction first, you can inspect it before presenting it to the wallet.

For example:

```ts
const built =
  await moonz.buildBuyTransaction({
    mint,
    buyer: wallet.publicKey,
    amount,
    slippageBps: 100
  });

console.log(
  built.instruction
);

console.log(
  built.quote.quoteAsset
);

console.log(
  built.transaction.instructions.length
);
```

This is useful during integration development.

## 🧠 Do Not Modify Blindly

You are free to inspect a built transaction.

But if you modify its instruction ordering, account metas, encoded amounts or protocol instructions, you are no longer executing the exact transaction the SDK constructed.

If your goal is ordinary Moonz trading, preserve the builder output.

{% hint style="warning" %}
Inspecting a transaction is safe.

Changing its protocol construction without understanding the Moonz instruction contract can invalidate the trade.
{% endhint %}

## 🔁 Blockhash Expiry

A built transaction does not remain valid forever.

Its blockhash has a validity window represented by:

```ts
built.lastValidBlockHeight
```

If a user leaves a signing request open for too long, the transaction may need to be rebuilt with a fresh blockhash and current Moonz quote.

## 🔄 Rebuild Rather Than Reuse Stale Transactions

If the transaction becomes stale:

```text
Do not reuse the expired transaction

Build again
      ↓
Read current Moonz state
      ↓
Fresh quote
      ↓
Fresh blockhash
      ↓
Fresh protected output
```

This also ensures the market route has not changed while the old transaction was waiting.

## 🌉 Curve Crossing Builds Normally

A buy that crosses the bonding boundary can still be constructed through:

```ts
buildBuyTransaction()
```

The returned quote can identify:

```ts
built.quote.crossesCurve

built.quote.migratesToAmm
```

The application does not need to manually construct separate bonding and AMM transactions.

## 🌌 Transaction Construction Boundary

The builder is the point where several Moonz layers meet:

```text
Current protocol state
        ↓
Market quote
        ↓
Route selection
        ↓
Slippage protection
        ↓
Account resolution
        ↓
Solana instructions
        ↓
Recent blockhash
        ↓
Unsigned Transaction
```

Everything after that belongs to wallet authorization and submission.

{% hint style="success" %}
The Trading SDK builds the protocol transaction.

The wallet decides whether to authorize it.
{% endhint %}

## 👛 Next Stop

The transaction exists.

It still cannot move anything until the wallet approves it.

Next stop:

**Wallet Signing and Submission**
