# 💸 Fees

Every Moonz trade currently uses the same headline trading fee:

```text
125 basis points
=
1.25 percent
```

What changes is where that fee goes.

Bonding and AMM trading use different fee allocation models because the markets have different liquidity needs.

{% hint style="info" %}
The trading fee rate stays at 1.25 percent.

The allocation of that fee changes when the token moves from bonding into the AMM.
{% endhint %}

## 🧮 Basis Points

The program defines:

```text
TRADE_FEE_TOTAL_BPS
=
125
```

Moonz uses:

```text
10000 basis points
=
100 percent
```

So:

```text
125 basis points
=
1.25 percent
```

## 🌙 The Two Fee Models

Moonz has two trading fee allocation models.

```text
BONDING

30 percent platform
70 percent creator
0 percent LP


AMM

15 percent platform
47.5 percent creator
37.5 percent LP
```

These percentages describe shares of the total trading fee.

They are not additional fees stacked on top of the 1.25 percent trade fee.

## ⚠️ Do Not Add the Shares to 1.25 Percent

For example:

```text
AMM fee
=
1.25 percent total

That 1.25 percent is then divided:

15 percent of fee
to platform

47.5 percent of fee
to creator

37.5 percent of fee
retained for LP
```

The user is not charged:

```text
1.25 percent
+
15 percent
+
47.5 percent
+
37.5 percent
```

The latter percentages allocate the existing fee.

## 📈 Bonding Fee Split

During the bonding phase the program defines:

```text
BONDING_PLATFORM_SHARE_BPS
=
3000

BONDING_CREATOR_SHARE_BPS
=
7000
```

That represents:

```text
30 percent
Platform

70 percent
Creator
```

There is no bonding LP fee.

## 🧮 Bonding Split Logic

At the program level:

```text
platform fee
=
total fee
×
3000
÷
10000
```

The creator receives the remainder.

Conceptually:

```text
Total bonding fee
        ↓
   ┌────┴────┐
   ↓         ↓
Platform   Creator
  30%        70%
```

Integer base unit rounding is handled by the program.

## 🟢 Bonding Buy Fees

For a normal bonding buy:

```text
Gross WSOL input
      ↓
Calculate 1.25 percent fee
      ↓
Split fee
      ↓
Creator fee
+
Platform fee
      ↓
Remaining effective WSOL
      ↓
Bonding treasury
```

The effective WSOL is the amount that participates in bonding curve pricing.

## 🏦 Bonding Buy Destinations

The quote flow is:

```text
Buyer WSOL
   |
   ├── Creator fee
   |      ↓
   |   Creator WSOL
   |   fee vault
   |
   ├── Platform fee
   |      ↓
   |   Platform WSOL ATA
   |
   └── Effective WSOL
          ↓
       Moonz WSOL
       treasury
```

The buyer receives tokens from the sale vault.

## 🔴 Bonding Sell Fees

For a bonding sell, the curve first determines the gross WSOL output.

Then:

```text
Gross WSOL output
      ↓
Calculate 1.25 percent fee
      ↓
Creator fee
+
Platform fee
      ↓
Net WSOL
      ↓
Seller
```

The sold Moonz tokens return to the sale vault.

## 📡 Bonding Fee Events

Bonding trade events expose:

```text
trade_fee

creator_fee

platform_fee

lp_fee
```

For a pure bonding sell:

```text
lp_fee = 0
```

because bonding does not use the AMM liquidity growth allocation.

## 🌊 AMM Fee Split

After migration the fee allocation changes.

The program defines:

```text
AMM_PLATFORM_SHARE_BPS
=
1500

AMM_CREATOR_SHARE_BPS
=
4750

AMM_LP_SHARE_BPS
=
3750
```

Those represent:

```text
15 percent
Platform

47.5 percent
Creator

37.5 percent
LP reserve
```

Together they account for the complete AMM trading fee.

## 🌱 Why the LP Share Exists

The AMM has real protocol controlled reserves.

Moonz therefore directs part of each AMM trading fee back into liquidity.

Conceptually:

```text
AMM trading
      ↓
Trading fees
      ↓
LP share
      ↓
Stays with
protocol liquidity
```

This allows trading activity itself to contribute to pool reserves.

## 🟢 AMM Buy Fee Flow

For an AMM buy:

```text
Gross quote input
      ↓
1.25 percent total fee
      ↓
Split into
Platform
Creator
LP
      ↓
Trade amount
+
LP fee
      ↓
Quote treasury
```

The creator and platform portions are transferred to their respective fee accounts.

## 🌱 LP Fee on a Buy

If:

```text
trade
=
gross input
minus total fee
```

then Moonz sends:

```text
trade
+
LP fee
```

into the active quote treasury.

For WSOL that is:

```text
treasury_wsol_vault
```

For USDC:

```text
treasury_usdc_vault
```

## 🔴 AMM Sell Fee Flow

For an AMM sell:

```text
Moonz tokens in
      ↓
Gross quote output
      ↓
1.25 percent fee
      ↓
Net seller output
+
Creator fee
+
Platform fee
```

Those amounts leave the quote treasury.

The LP fee does not.

## 🌱 LP Fee on a Sell

The program explicitly calculates the LP share but excludes it from the quote treasury outflow.

Conceptually:

```text
Gross quote output
        ↓
  ┌─────┼─────────┐
  ↓     ↓         ↓
Seller Creator Platform

LP share
   ↓
Remains in pool
```

This means the pool retains that portion rather than paying the complete gross output away.

## ☀️ WSOL and USDC Use the Same AMM Split

The AMM fee allocation is not limited to WSOL.

Both:

```text
AmmLive + WSOL

AmmLive + USDC
```

use the same:

```text
15 percent platform

47.5 percent creator

37.5 percent LP
```

share model.

The difference is which quote asset accounts hold the values.

## 💵 Quote Asset Determines the Fee Asset

When the AMM quote asset is WSOL:

```text
Creator fee
=
WSOL

Platform fee
=
WSOL

LP fee
=
WSOL retained
in WSOL treasury
```

When the AMM quote asset is USDC:

```text
Creator fee
=
USDC

Platform fee
=
USDC

LP fee
=
USDC retained
in USDC treasury
```

## 👤 Creator Fee Authority

Moonz does not send creator trading fees directly into an arbitrary wallet account on every trade.

Instead it derives a creator fee authority PDA using:

```text
creator_fees

+

creator public key
```

The program constant is:

```text
CREATOR_FEES_SEED
=
"creator_fees"
```

There is one creator fee authority PDA per creator wallet.

## 🏦 Creator Fee Vaults

That authority can own canonical quote token ATAs such as:

```text
WSOL creator fee vault

USDC creator fee vault
```

Trading fees accumulate in the appropriate quote asset vault.

Conceptually:

```text
Creator
   ↓
Creator fee authority PDA
   |
   ├── WSOL fee ATA
   |
   └── USDC fee ATA
```

## 🌌 Fees Can Aggregate Across Launches

The creator fee authority is derived from the creator wallet rather than from an individual token mint.

That means the canonical WSOL or USDC fee vault belongs to the creator fee authority, not to one particular Launch State.

A creator operating multiple Moonz launches can therefore have creator trading fees accumulate by quote asset under the same creator fee authority structure.

## 💰 Claim Fees

The Moonz program exposes:

```text
claim_fees
```

This instruction allows the creator to withdraw an accumulated creator fee vault balance.

## ✍️ Creator Must Sign

The `ClaimFees` account contract requires:

```text
creator
=
Signer
```

So creator fees cannot be claimed merely because someone knows the fee vault address.

The creator wallet authorizes the claim.

## 🛡️ Canonical Vault Checks

Before transferring creator fees, Moonz verifies:

```text
Creator fee vault owner
=
creator fee authority PDA

Creator receiver owner
=
creator wallet

Fee vault mint
=
receiver mint
```

The fee mint must be:

```text
WSOL

or

USDC
```

The program also requires both token accounts to be their canonical associated token accounts.

## 📦 Claim Amount

`claim_fees()` reads:

```text
creator_fee_vault.amount
```

and requires:

```text
amount > 0
```

It then transfers that entire vault balance to the creator's canonical receiver ATA.

Conceptually:

```text
Creator fee vault
      ↓
claim_fees
      ↓
Creator signature
      ↓
Entire selected
vault balance
      ↓
Creator ATA
```

## 🧬 PDA Signs the Token Transfer

The creator signs the Moonz instruction.

The actual SPL token transfer out of the creator fee vault is authorized by the creator fee authority PDA through program signer seeds.

That keeps the fee vault itself under deterministic program authority.

## 📡 ClaimFeesEvent

A successful fee claim emits:

```text
ClaimFeesEvent
```

with:

```text
creator

amount
```

Indexers can use that event to identify creator fee withdrawal activity.

## 📊 Trade Events Expose the Breakdown

Moonz trade events expose the actual integer amounts applied to the trade.

The relevant fields include:

```text
trade_fee

creator_fee

platform_fee

lp_fee
```

They are available across the Moonz buy and sell event structures.

## 🟢 AmmBuyEvent

An AMM buy event records:

```text
trade_fee

creator_fee

platform_fee

lp_fee
```

along with the quote asset and trade amounts.

## 🔴 AmmSellEvent

AMM sells expose the same fee breakdown.

This gives indexers enough information to distinguish:

```text
Total user trading fee

Creator allocation

Platform allocation

Liquidity allocation
```

without trying to reconstruct the split after the event.

## 🌉 A Crossing Buy Can Use Both Fee Models

A buy that finishes bonding and continues into the AMM can cross two fee allocation models in one user transaction.

The first portion uses:

```text
Bonding split

30 percent platform

70 percent creator
```

The remaining AMM portion uses:

```text
AMM split

15 percent platform

47.5 percent creator

37.5 percent LP
```

## 🧮 Crossing Fee Accounting

Conceptually:

```text
One large buy
      ↓
Bonding portion
      ↓
Bonding fee split
      ↓
Migration
      ↓
Remaining quote input
      ↓
AMM portion
      ↓
AMM fee split
```

The program tracks the resulting creator, platform and LP amounts across the transaction.

## 📡 BuyEvent Can Represent the Combined Buy

For the overall WSOL buy path, `BuyEvent` records totals such as:

```text
trade_fee

creator_fee

platform_fee

lp_fee
```

If the buy crosses into the AMM, its LP field can therefore contain the LP fee generated by the AMM portion.

## 🧠 Slippage Is Not a Fee

Do not confuse:

```text
slippageBps
```

with:

```text
TRADE_FEE_TOTAL_BPS
```

They solve different problems.

```text
Trading fee
      ↓
Protocol economic charge

Slippage
      ↓
Minimum acceptable output
```

Changing a user's slippage tolerance does not change the protocol trading fee.

## 🚀 Launch Fee Is Separate

Token creation has a separate fixed launch funding component.

The program defines:

```text
CREATE_FEE_LAMPORTS
=
40000000
```

which is:

```text
0.04 SOL
```

The creator escrow funding transaction deposits:

```text
Create fee
+
Dev Buy amount
```

These are separate from trading fees.

## 🧱 What the Launch Fee Covers

The program describes the fixed creation fee as covering:

```text
Account setup

Rent funding

Storage and IPFS
operations
```

The launch escrow is used through token initialization and settlement.

## 💰 Dev Buy Is Not the Launch Fee

If a creator selects:

```text
0.5 SOL Dev Buy
```

that does not mean the launch fee is 0.5 SOL.

They are separate values:

```text
Fixed launch funding component
0.04 SOL

Creator selected Dev Buy
Variable
```

The Dev Buy itself also participates in the bonding trading fee model when it executes.

## 🔀 PCLS Fee Is Also Separate

Pool quote asset switching has its own fixed fee.

It is not part of the:

```text
1.25 percent trading fee
```

and it is not the:

```text
0.04 SOL launch fee
```

We will cover that fee and its refund behaviour on the PCLS page.

## 🧭 Three Different Fee Concepts

It helps to keep these separate:

```text
TRADING FEE

1.25 percent
of trades


LAUNCH FEE

Fixed creation
funding component


PCLS SWITCH FEE

Fixed pool switch
control fee
```

Each belongs to a different protocol operation.

## 🔎 Developer View

For trading interfaces, the Trading SDK quote objects already expose:

```text
tradeFeeRaw

creatorFeeRaw

platformFeeRaw

lpFeeRaw
```

So applications do not need to recreate fee splitting logic merely to display a quote.

## 📡 Indexer View

For indexers, Moonz protocol events expose fee components directly.

Useful event fields include:

```text
quote_asset

trade_fee

creator_fee

platform_fee

lp_fee
```

That allows fee analytics to remain tied to canonical protocol events.

## 🌌 Fee Flow in One Picture

```text
             MOONZ TRADE

                 ↓

          1.25% TOTAL FEE

              /       \
             /         \
        BONDING         AMM

           ↓             ↓

       Platform 30%   Platform 15%

       Creator 70%    Creator 47.5%

       LP 0%          LP 37.5%
                          ↓
                     Remains with
                       liquidity


CREATOR FEES

Trade
  ↓
Creator fee vault
  ↓
claim_fees
  ↓
Creator signs
  ↓
Creator WSOL
or USDC ATA
```

{% hint style="success" %}
Moonz uses one headline trading fee with two allocation models.

Bonding rewards the creator and platform.

AMM trading also directs part of the same fee back into protocol controlled liquidity.
{% endhint %}

## 🔀 Next Stop

The AMM can continue operating while its quote asset evolves between SOL and USDC.

That is where PCLS enters the protocol.

Next stop:

**PCLS**
