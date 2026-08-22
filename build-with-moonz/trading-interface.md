# 🚀 Trading Interface

We have reached the controls.

A Moonz trading interface does not need separate user flows for bonding, SOL AMM and USDC AMM.

The public Trading SDK reads current Moonz state and selects the correct route.

Your interface can focus on:

```text
Wallet

Mint

Buy or sell

Amount

Quote

Slippage

Minimum output

User confirmation

Wallet signature

Transaction result
```

{% hint style="success" %}
Moonz routing follows current protocol state.

Your interface does not need to hardcode whether a token is bonding or trading against SOL or USDC.
{% endhint %}

## 📦 Install the Trading SDK

```bash
npm install @moonz-fun/trading-sdk
```

Then import:

```ts
import {
  MoonzTradingSDK
} from "@moonz-fun/trading-sdk";
```

## 🌙 Create the Client

```ts
const trading =
  new MoonzTradingSDK({
    rpcUrl:
      process.env.SOLANA_RPC_URL
  });
```

The Trading SDK uses current Solana state for trade quotes and transaction construction.

## 👛 Wallet Interface

The high level trading methods expect a wallet shaped like:

```ts
{
  publicKey,
  signTransaction
}
```

The public type is:

```text
MoonzWalletSigner
```

with:

```text
publicKey

signTransaction
```

This is compatible with the normal Solana wallet adapter style.

{% hint style="warning" %}
Moonz never needs the user's private key.

The wallet signs the transaction locally.
{% endhint %}

## 🧭 One Interface, Multiple Markets

Moonz automatically routes according to current state.

Conceptually:

```text
BONDING
      ↓
Bonding buy or sell


AMM_LIVE + SOL
      ↓
SOL AMM buy or sell


AMM_LIVE + USDC
      ↓
USDC AMM buy or sell
```

A large SOL buy can also cross:

```text
BONDING
      ↓
BONDING_TO_AMM
      ↓
AMM
```

inside one trade path.

## 🔎 Quote Before Asking for a Signature

For a buy:

```ts
const quote =
  await trading.quoteBuy({
    mint,
    amount:
      "0.5",
    slippageBps:
      100
  });
```

No wallet signature is required to obtain this quote.

## 💰 Buy Amount Meaning

For a buy:

```text
SOL market

amount
=
SOL to spend


USDC market

amount
=
USDC to spend
```

The SDK determines the active quote asset.

## 📊 Buy Quote

Useful fields include:

```ts
quote.market

quote.quoteAsset

quote.amountIn

quote.expectedTokensOut

quote.minTokensOut

quote.tradeFeeRaw

quote.creatorFeeRaw

quote.platformFeeRaw

quote.lpFeeRaw

quote.slippageBps

quote.crossesCurve

quote.migratesToAmm
```

## 🛣️ Market Field

A buy quote can report:

```text
BONDING

AMM

BONDING_TO_AMM
```

This is useful for telling the user how the current trade will execute.

## 🌉 Crossing the Curve

If:

```ts
quote.crossesCurve
```

is true, one SOL buy uses both the final bonding inventory and the new AMM.

If:

```ts
quote.migratesToAmm
```

is true, the buy completes the bonding allocation and causes migration.

A UI could display:

```text
This trade completes bonding
and enters the Moonz AMM.
```

## 🛡️ Minimum Buy Output

The SDK calculates:

```ts
quote.minTokensOut
```

from the quote and selected slippage tolerance.

That is the protected minimum token output for the trade.

## 🔴 Quote a Sell

```ts
const quote =
  await trading.quoteSell({
    mint,
    amount:
      "100000",
    slippageBps:
      100
  });
```

For a sell:

```text
amount
=
Moonz tokens to sell
```

Moonz launch tokens use 6 decimals.

## 📊 Sell Quote

Useful fields include:

```ts
quote.market

quote.quoteAsset

quote.tokensIn

quote.grossQuoteOut

quote.expectedQuoteOut

quote.minQuoteOut

quote.tradeFeeRaw

quote.creatorFeeRaw

quote.platformFeeRaw

quote.lpFeeRaw

quote.slippageBps
```

## 💱 Sell Output

If:

```text
quoteAsset = SOL
```

the expected quote output is SOL.

If:

```text
quoteAsset = USDC
```

the expected quote output is USDC.

The interface should label the output from:

```ts
quote.quoteAsset
```

rather than assuming SOL.

## 🎚️ Slippage

If omitted, the SDK default is:

```text
100 basis points

=

1 percent
```

For example:

```ts
slippageBps:
  100
```

Slippage is user protection.

It is not the Moonz protocol trading fee.

## 💸 Show the Fee Breakdown

The quote exposes:

```text
tradeFeeRaw

creatorFeeRaw

platformFeeRaw

lpFeeRaw
```

This lets an interface show the actual quoted fee allocation without recreating Moonz fee mathematics.

## 🧠 Raw Fee Values

Fee values ending in:

```text
Raw
```

are base units.

The decimals depend on the quote asset.

```text
SOL
9 decimals

USDC
6 decimals
```

Use exact integer handling when converting raw values.

## ✅ Confirm Screen

Before opening the wallet, a useful confirmation panel could show:

```text
BUY

Spend
0.5 SOL

Expected
1,234,567 MOONZ

Minimum received
1,222,221 MOONZ

Market
BONDING

Slippage
1%

Trading fee
Quoted by Moonz
```

The numbers above are only a display example.

Use the current SDK quote for a real trade.

## ⚠️ A Preview Quote Is Not Locked Forever

Blockchain state can change after you display a quote.

Another trade may happen.

Bonding may complete.

PCLS may change the active quote asset.

The high level `buy()` and `sell()` paths build against current state again before signing.

So do not promise that an old preview quote is permanently locked.

## 🟢 Execute a Buy

Once the user confirms:

```ts
const result =
  await trading.buy({
    mint,
    wallet,
    amount:
      "0.5",
    slippageBps:
      100
  });
```

The high level method:

```text
Reads current state

Builds the transaction

Calculates a fresh quote

Routes the trade

Requests wallet signature

Submits the signed transaction

Confirms it
```

## 📜 Buy Result

The result contains:

```ts
result.signature

result.confirmationSlot

result.instruction

result.quote
```

The instruction is:

```text
buy
```

for the SOL paths or:

```text
amm_buy_usdc
```

for a USDC AMM buy.

## 🔴 Execute a Sell

```ts
const result =
  await trading.sell({
    mint,
    wallet,
    amount:
      "100000",
    slippageBps:
      100
  });
```

The same wallet boundary applies.

## 📜 Sell Result

```ts
result.signature

result.confirmationSlot

result.instruction

result.quote
```

The instruction is:

```text
sell
```

for the SOL paths or:

```text
amm_sell_usdc
```

for a USDC AMM sell.

## 🔐 What the Wallet Signs

The SDK requires:

```ts
wallet.publicKey
```

and:

```ts
wallet.signTransaction
```

The flow is:

```text
Moonz builds transaction
        ↓
Wallet receives transaction
        ↓
User approves
        ↓
Wallet signs locally
        ↓
Signed bytes returned
        ↓
Transaction submitted
```

Moonz does not ask your application to send a private key to the SDK.

## 🚫 Missing Wallet

If the wallet has no public key, the high level trade fails with:

```text
Wallet is not connected
```

If the wallet does not support transaction signing, it fails with:

```text
Wallet does not support signTransaction
```

Your interface should handle both before presenting a final submit state.

## 🧩 Basic Buy Function

```ts
async function executeBuy({
  mint,
  wallet,
  amount,
  slippageBps
}) {
  if (
    !wallet.publicKey
  ) {
    throw new Error(
      "Connect wallet first"
    );
  }

  if (
    !wallet.signTransaction
  ) {
    throw new Error(
      "Wallet cannot sign transactions"
    );
  }

  return trading.buy({
    mint,
    wallet,
    amount,
    slippageBps
  });
}
```

## 🧩 Basic Sell Function

```ts
async function executeSell({
  mint,
  wallet,
  amount,
  slippageBps
}) {
  if (
    !wallet.publicKey
  ) {
    throw new Error(
      "Connect wallet first"
    );
  }

  if (
    !wallet.signTransaction
  ) {
    throw new Error(
      "Wallet cannot sign transactions"
    );
  }

  return trading.sell({
    mint,
    wallet,
    amount,
    slippageBps
  });
}
```

## 🛠️ Build Without Sending

Some applications need more control over submission.

Use:

```ts
const built =
  await trading
    .buildBuyTransaction({
      mint,
      buyer:
        wallet.publicKey,
      amount:
        "0.5",
      slippageBps:
        100
    });
```

This returns an unsigned transaction.

Nothing is signed or submitted.

## 📦 Built Buy Result

The returned object includes:

```ts
built.transaction

built.quote

built.instruction

built.blockhash

built.lastValidBlockHeight

built.accounts
```

This gives advanced applications access to the exact transaction Moonz constructed.

## 🔴 Build a Sell

```ts
const built =
  await trading
    .buildSellTransaction({
      mint,
      seller:
        wallet.publicKey,
      amount:
        "100000",
      slippageBps:
        100
    });
```

Again:

```text
Nothing is signed

Nothing is submitted
```

until your application chooses to continue.

## ⚠️ Build Methods Also Use Current State

The transaction builders generate a fresh quote while constructing the transaction.

So this sequence:

```text
Preview quote

Wait

Build transaction
```

can legitimately produce a different quote if the market changed in between.

Always show the user appropriate transaction state and avoid presenting stale values as guaranteed execution.

## ☀️ SOL Buy Handling

For SOL trading, the builder handles the WSOL mechanics required by the SPL Token program.

Conceptually:

```text
Native SOL
    ↓
WSOL trading account
    ↓
Moonz trade
    ↓
Temporary WSOL handling
cleaned up
```

The user interface can continue describing the quote asset as SOL.

## 💵 USDC Buy Handling

For a USDC AMM buy, the Trading SDK uses the buyer's USDC associated token account.

The route is selected automatically when current Moonz state says:

```text
AMM_LIVE
+
USDC
```

## 🔴 SOL Sell Handling

A SOL market sell produces WSOL through the protocol transaction and the builder handles the native SOL return path for the seller.

Your interface can display the expected output as SOL.

## 💵 USDC Sell Handling

For a USDC market:

```text
Moonz tokens
      ↓
USDC AMM
      ↓
USDC output
```

No SOL output should be shown simply because the token originally launched against SOL.

## 🔀 PCLS and Trading

While the lifecycle is:

```text
SWITCHING
```

there is no normal live trading route.

Do not leave a stale quote active while the token is switching.

A good interface should refresh token state and disable submission until a valid trading route is available again.

## 🌕 After PCLS

When PCLS completes, the same code still works:

```ts
await trading.quoteBuy({
  mint,
  amount
});
```

If the new quote asset is USDC, the quote returns:

```text
quoteAsset
USDC
```

and the transaction builder selects the USDC instruction.

## 🌉 Bonding to AMM

A buy near the end of bonding can report:

```ts
quote.market ===
  "BONDING_TO_AMM"
```

and:

```ts
quote.crossesCurve ===
  true
```

Your interface does not need to ask the user to submit one bonding transaction and another AMM transaction.

Moonz handles the supported crossing path.

## 🛡️ Keep Minimum Output Visible

For buys show:

```ts
quote.minTokensOut
```

For sells show:

```ts
quote.minQuoteOut
```

These communicate what the current slippage protection means in actual output terms.

## 🚦 Disable Double Submission

While a trade is being signed or submitted, disable the submit control.

For example:

```text
IDLE

↓

QUOTING

↓

AWAITING WALLET

↓

SUBMITTING

↓

CONFIRMING

↓

SUCCESS
or
ERROR
```

These are interface states, not Moonz protocol phases.

## 🧠 Do Not Mark Success at Signature Request

The wallet accepting a signing request does not mean the trade landed.

Treat the trade as successful only after the execution method resolves with its confirmed result.

## ✅ Successful Trade

A useful success state can show:

```text
Trade confirmed

Transaction
SIGNATURE

Confirmation slot
SLOT

Route
AMM

Quote asset
USDC
```

Use the returned result values.

## ❌ Failed Trade

Handle errors and restore the interface to a usable state.

Do not silently turn a failed transaction into a successful balance update.

After an error, reread current token and wallet state before the next trade attempt.

## 🔄 Refresh After Confirmation

Once a trade confirms, refresh:

```text
Token balance

Current market

Current price

Current reserves

Bonding progress

Recent trades
```

because the user's transaction changed protocol state.

## 📡 Combine With the Information SDK

A full trading page can use both public packages.

```bash
npm install @moonz-fun/sdk @moonz-fun/trading-sdk
```

Use:

```text
@moonz-fun/sdk

for

Token information
Metadata
Market data
Reserves
Integrity
Events


@moonz-fun/trading-sdk

for

Quotes
Transaction building
Buy execution
Sell execution
```

## 🧭 Recommended Page Flow

A simple architecture is:

```text
Load token
      ↓
Verify Moonz state
      ↓
Display market
      ↓
Connect wallet
      ↓
Choose buy or sell
      ↓
Enter amount
      ↓
Request quote
      ↓
Display quote
      ↓
User confirms
      ↓
Build current transaction
      ↓
Wallet signs
      ↓
Submit
      ↓
Confirm
      ↓
Refresh state
```

## 🚀 Complete Trading Example

```ts
import {
  MoonzTradingSDK
} from "@moonz-fun/trading-sdk";

const trading =
  new MoonzTradingSDK({
    rpcUrl:
      process.env.SOLANA_RPC_URL
  });

const mint =
  "TOKEN_MINT";

async function previewBuy(
  amount: string
) {
  return trading.quoteBuy({
    mint,
    amount,
    slippageBps:
      100
  });
}

async function submitBuy(
  wallet,
  amount: string
) {
  const result =
    await trading.buy({
      mint,
      wallet,
      amount,
      slippageBps:
        100
    });

  console.log({
    signature:
      result.signature,

    confirmationSlot:
      result.confirmationSlot,

    instruction:
      result.instruction,

    market:
      result.quote.market,

    quoteAsset:
      result.quote.quoteAsset,

    expectedTokensOut:
      result.quote
        .expectedTokensOut,

    minTokensOut:
      result.quote
        .minTokensOut
  });

  return result;
}
```

## 🌌 What You Have Built

With the public Moonz SDKs you can now build:

```text
Token explorer

Live trade feed

Bonding tracker

AMM dashboard

PCLS monitor

Trading interface
```

using public developer surfaces.

{% hint style="success" %}
Read current state.

Quote the current market.

Let Moonz choose the protocol route.

Let the wallet sign.

Then refresh from Solana after confirmation.
{% endhint %}

# 🌕 You Have Arrived

You now have the pieces required to build directly on Moonz.

The rest is yours.

Try things.

Break your interface.

Read the state again.

And whatever you build next, remember the most useful rule in the guide:

**Don't Panic.**
