# 🔴 Sell Moonz

Buying Moonz puts tokens in the wallet.

Selling turns them back into the market's active quote asset.

The Moonz Trading SDK exposes two public sell paths:

```text
sell()

buildSellTransaction()
```

Use `sell()` for the complete wallet flow.

Use `buildSellTransaction()` when your application wants control over signing and submission.

{% hint style="info" %}
The SDK automatically decides whether the sell should use the standard Moonz `sell` instruction or the USDC AMM `amm_sell_usdc` instruction.
{% endhint %}

## 🚀 The Fastest Sell

A complete sell can be:

```ts
const result =
  await moonz.sell({
    mint: "TOKEN_MINT",
    wallet,
    amount: "100000",
    slippageBps: 100
  });

console.log(
  result.signature
);
```

The `amount` is the human readable Moonz token quantity being sold.

Moonz launch tokens use:

```text
6 decimals
```

## 🧭 The Full Sell Flow

The high level method performs:

```text
Read current Moonz state
        ↓
Quote sell
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

The application does not need to select the protocol instruction itself.

## 🛣️ Automatic Sell Routing

The current routes are:

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

That means the public interface remains:

```ts
await moonz.sell(...)
```

even when the underlying Moonz market changes.

## 🌱 Bonding Sell

During:

```text
BONDING
```

the active quote asset is SOL.

For example:

```ts
const result =
  await moonz.sell({
    mint,
    wallet,
    amount: "50000",
    slippageBps: 100
  });
```

The amount represents:

```text
50000 Moonz tokens
```

The expected output is quoted in SOL.

## 🚀 SOL AMM Sell

When the token reaches:

```text
AMM_LIVE
```

and the current quote asset is:

```text
SOL
```

the canonical instruction remains:

```text
sell
```

The same public method works:

```ts
await moonz.sell({
  mint,
  wallet,
  amount: "50000",
  slippageBps: 100
});
```

## 💵 USDC AMM Sell

If Moonz state says:

```text
AMM_LIVE + USDC
```

the SDK automatically switches to:

```text
amm_sell_usdc
```

The seller still supplies the Moonz token quantity:

```ts
await moonz.sell({
  mint,
  wallet,
  amount: "50000",
  slippageBps: 100
});
```

But the expected output is now USDC.

## 💬 The Sell Is Quoted From Current State

Before transaction construction, the SDK calls the current sell quote logic.

Conceptually:

```text
Mint
  ↓
Read Launch State
  ↓
Read reserves
  ↓
Determine market
  ↓
Calculate sell output
  ↓
Apply fee
  ↓
Apply slippage
  ↓
Build transaction
```

The transaction therefore uses a fresh quote rather than relying on an old interface preview.

## 🛡️ Minimum Output

For sells, the important protected raw value is:

```ts
quote.minQuoteOutRaw
```

The builder converts it into the amount expected by the on chain instruction.

For the SOL route:

```text
tokensIn

minWsolOut
```

For the USDC route:

```text
tokensIn

minUsdcOut
```

This is the execution protection derived from the requested slippage.

## 🧱 Build Without Signing

To construct an unsigned sell transaction:

```ts
const built =
  await moonz.buildSellTransaction({
    mint: "TOKEN_MINT",
    seller: wallet.publicKey,
    amount: "100000",
    slippageBps: 100
  });
```

Nothing is signed.

Nothing is submitted.

## 📦 Built Sell Result

The public `MoonzBuiltSellTransaction` contains:

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

The `transaction` is the unsigned Solana transaction.

The `quote` is the `MoonzSellQuote` used during construction.

## 🛰️ Selected Instruction

Inspect:

```ts
built.instruction
```

The result will be:

```text
sell

or

amm_sell_usdc
```

This is useful for logging and transaction diagnostics.

## 🔗 Resolved Accounts

The builder exposes account information including:

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

For USDC trades, the returned structure can also include:

```text
sellerUsdcAta

treasuryUsdcVault

creatorFeeUsdcVault

platformUsdcAta
```

These addresses make it easier to inspect exactly what the SDK constructed.

## 🟣 The SOL Sell Transaction

The SOL sell route currently builds this sequence:

```text
1
Ensure seller Moonz token ATA exists

2
Ensure seller WSOL ATA exists

3
Ensure platform WSOL ATA exists

4
Execute Moonz sell

5
Close seller WSOL ATA
back to seller
```

That final step matters.

The Moonz trade returns WSOL.

The SDK then closes the seller WSOL account so the resulting lamports become native SOL again.

## 🌊 WSOL Back to Native SOL

The transaction handles:

```text
Moonz tokens
      ↓
Moonz sell
      ↓
WSOL received
      ↓
sellerWsolAta
      ↓
close account
      ↓
native SOL returned
to seller
```

Your application does not need to manually unwrap the proceeds after the trade.

## 🔐 The Close Happens in the Same Transaction

The SDK adds:

```ts
createCloseAccountInstruction(...)
```

after the Moonz sell instruction.

The destination is the seller.

The received WSOL is unwrapped and native SOL returns to seller in the same transaction.

So the sell and unwrap are part of the same transaction construction.

## 💵 The USDC Sell Transaction

The USDC path does not need WSOL conversion.

The SDK prepares:

```text
sellerTokenAta

sellerUsdcAta

creatorFeeUsdcVault

platformUsdcAta
```

and executes:

```text
amm_sell_usdc
```

The resulting user output is USDC.

## 💳 USDC Goes to the Seller ATA

For the USDC path:

```text
Moonz tokens
      ↓
amm_sell_usdc
      ↓
USDC
      ↓
sellerUsdcAta
```

There is no WSOL close operation in this route.

## 🧱 Account Creation Is Idempotent

Both sell builders use idempotent associated token account creation instructions.

This allows the transaction to ensure required token accounts exist without your application having to perform separate existence checks first.

For the SOL route this includes the seller Moonz token ATA and seller WSOL ATA.

For USDC it includes the seller Moonz token ATA and seller USDC ATA.

## 🎯 Current State Is Checked Again

The builder validates the current Moonz Launch State before constructing the instruction.

The SOL route requires:

```text
BONDING

or

AMM_LIVE + SOL
```

The USDC route requires:

```text
AMM_LIVE + USDC
```

The requested mint must also match the mint recorded in Launch State.

## 🚫 Non Tradeable Tokens

If the token is not currently in a tradeable state, the builder fails rather than constructing an invalid transaction.

The supported phases for trading are:

```text
BONDING

AMM_LIVE
```

Other lifecycle states are not treated as active markets.

## 🔴 Bonding Sell Limits

During bonding, the sell quote validates that the token amount being sold does not exceed:

```text
tokensSold
```

on the bonding curve.

Conceptually:

```text
tokensIn
   ≤
tokensSold
```

If not, the quote fails before transaction construction.

## 💧 Bonding Liquidity

The bonding sell quote also verifies the current WSOL treasury has enough liquidity for the calculated gross output.

If the treasury cannot satisfy the calculated sell, the SDK rejects the quote.

That means transaction construction does not proceed with an impossible output assumption.

## 🌊 AMM Liquidity

For an AMM sell, the SDK reads the current:

```text
LP token reserve

active quote reserve
```

The active quote reserve is either:

```text
WSOL

or

USDC
```

depending on current Moonz state.

## 💰 Gross and Expected Output

A sell quote exposes:

```ts
quote.grossQuoteOut
```

before the trading fee.

Then:

```ts
quote.expectedQuoteOut
```

after the trading fee.

Finally:

```ts
quote.minQuoteOut
```

is the slippage protected minimum.

Conceptually:

```text
Gross output
      ↓
Trading fee
      ↓
Expected output
      ↓
Slippage protection
      ↓
Minimum output
```

## 🛑 Zero Input Protection

The SOL builder refuses to construct a sell with:

```text
zero tokensIn

or

zero minWsolOut
```

The USDC builder similarly rejects:

```text
zero tokensIn

or

zero minUsdcOut
```

The SDK does not intentionally construct zero value Moonz sell instructions.

## ⛓️ Blockhash Context

The builder obtains a recent blockhash and sets:

```text
feePayer = seller

recentBlockhash = latest blockhash
```

It also returns:

```ts
built.blockhash

built.lastValidBlockHeight
```

for use during confirmation.

## 👛 Sign the Transaction Yourself

When using the lower level builder:

```ts
const built =
  await moonz.buildSellTransaction({
    mint,
    seller: wallet.publicKey,
    amount: "100000",
    slippageBps: 100
  });

const signed =
  await wallet.signTransaction(
    built.transaction
  );
```

Your application now controls submission.

## 📡 Submit a Built Sell

For example:

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

This is useful when your infrastructure already manages transaction submission.

## 🚀 What sell() Does For You

The higher level method handles the complete flow.

It:

```text
Checks wallet connection

Checks signTransaction exists

Builds fresh sell transaction

Requests wallet signature

Serializes signed transaction

Calls sendRawTransaction

Confirms transaction

Checks confirmation error

Returns MoonzSellResult
```

## ✅ Successful Sell Result

A successful high level sell returns:

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

The result preserves both chain confirmation and the quote used during construction.

## 🔎 Inspect the Result

```ts
const result =
  await moonz.sell({
    mint,
    wallet,
    amount: "100000",
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
  "Expected output:",
  result.quote.expectedQuoteOut
);
```

## ❌ Confirmation Errors

The SDK checks:

```ts
confirmation.value.err
```

after submitting the signed transaction.

If Solana reports an execution error, `sell()` throws rather than returning a successful result.

A transaction signature alone is therefore not treated as proof of a successful Moonz sell.

## 👛 Wallet Rejection

The wallet can reject the signing request.

That is not a protocol failure.

Applications should handle the full error boundary around the trade.

```ts
try {
  const result =
    await moonz.sell({
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

Possible failures include:

```text
Invalid amount

Unsupported state

Insufficient liquidity

Wallet disconnected

Wallet rejected signature

RPC failure

Transaction execution failure
```

## 🔁 Market State Can Move

Between the user's first quote and the eventual transaction:

```text
Another trade may execute

Reserves may change

The active market may move

The token lifecycle may change
```

The builder rereads current state.

The slippage protected minimum output then protects execution against unacceptable movement.

## 🖥️ A Practical Sell Button

A trading interface can follow:

```text
User enters Moonz amount
        ↓
quoteSell()
        ↓
Display expected output
        ↓
Display minimum output
        ↓
Display SOL or USDC
        ↓
User presses Sell
        ↓
sell()
        ↓
Wallet approval
        ↓
Solana confirmation
        ↓
Show result
```

## 🌌 Complete Sell Journey

The entire sell path looks like:

```text
MOONZ TOKENS
      ↓
quoteSell()
      ↓
Preview
      ↓
sell()
      ↓
Read current state
      ↓
Automatic route
      |
      ├────────────────────┐
      ↓                    ↓
BONDING or             AMM_LIVE
AMM_LIVE + SOL           + USDC
      |                    |
      ↓                    ↓
    sell             amm_sell_usdc
      |                    |
      ↓                    ↓
Receive WSOL          Receive USDC
      |                    |
      ↓                    ↓
Close seller          sellerUsdcAta
WSOL ATA
      |
      ↓
Native SOL
      |
      └─────────┬──────────┘
                ↓
          Confirm trade
                ↓
        MoonzSellResult
```

{% hint style="success" %}
The application chooses how many Moonz tokens to sell.

The Trading SDK determines which market returns the quote asset.
{% endhint %}

## 🛣️ Next Stop

We can now buy and sell through every currently supported Moonz trading market.

Next we will look at the routing itself and why your integration should let the SDK handle it.

Next stop:

**Automatic Market Routing**
