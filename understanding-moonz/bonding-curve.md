# 📈 Bonding Curve

Before a Moonz token becomes an AMM, it trades against a bonding curve.

There is no external liquidity provider setting the opening market.

The Moonz program calculates each trade from its curve state.

{% hint style="info" %}
The bonding curve is enforced by the Moonz program.

Applications should use the public SDK for quotes rather than recreating protocol math in interface code.
{% endhint %}

## 🌙 The Curve Phase

After the creator Dev Buy completes, the token enters:

```text
Curve
```

The public Moonz SDK exposes this phase as:

```text
BONDING
```

During this phase the market uses:

```text
WSOL quote

Sale vault inventory

Virtual reserves

Constant product math
```

## 🪙 The Sale Inventory

Every Moonz launch allocates:

```text
650,000,000 tokens
```

to the bonding sale.

The Launch State records:

```text
sale_supply

tokens_sold

sol_collected
```

The amount remaining is:

```text
sale_remaining
=
sale_supply
minus
tokens_sold
```

## 🌌 Virtual Reserves

The bonding curve begins with two protocol constants:

```text
Virtual SOL
117 SOL

Virtual tokens
760,000,000 tokens
```

In program units:

```text
V_SOL
=
117 × 1,000,000,000 lamports

V_TOK
=
760,000,000 × 1,000,000 token units
```

Moonz launch tokens use:

```text
6 decimals
```

## 🧠 Virtual Does Not Mean Deposited

The virtual reserves are mathematical values.

They are not an extra:

```text
117 real SOL

or

760 million additional minted tokens
```

sitting in a wallet.

They shape the price function.

Actual bonding inventory and actual collected WSOL remain separately accounted for.

## 🧬 Effective Curve Reserves

The curve combines the virtual values with current real bonding state.

Conceptually:

```text
SOL reserve
=
117 virtual SOL
+
real curve SOL

Token reserve
=
760M virtual tokens
+
remaining sale tokens
```

In the program math:

```text
r_sol
=
V_SOL + sol_real

r_tok
=
V_TOK + tok_real
```

During a buy, the real token component is the remaining sale inventory.

## 🔎 An Important Detail

At the very beginning, before any tokens have been sold:

```text
Real sale inventory
650M

Virtual token reserve
760M
```

So the mathematical token side begins with:

```text
760M
+
650M
=
1.41B
effective curve tokens
```

That does **not** mean 1.41 billion tokens exist.

Only 1 billion Moonz tokens are minted.

The additional 760 million is virtual curve state.

## ✖️ Constant Product

Moonz uses constant product style math.

The invariant is conceptually:

```text
k
=
r_sol × r_tok
```

A trade changes the two reserve values while the calculation preserves the invariant subject to integer rounding.

## 🟢 Bonding Buy

For a normal bonding buy, the user supplies WSOL.

Conceptually:

```text
Gross WSOL input
      ↓
Trading fee
      ↓
Effective WSOL
      ↓
Curve calculation
      ↓
Moonz tokens out
```

The trading fee is separated before the effective buy amount enters the curve.

## 💸 Buy Fee

The current total trading fee is:

```text
125 basis points
=
1.25 percent
```

For a buy:

```text
effective WSOL
=
gross WSOL
minus
trade fee
```

The fee distribution itself is covered on the Fees page.

## 🧮 Buy Mathematics

After the effective input is known:

```text
r_sol_new
=
r_sol
+
effective WSOL
```

The new token reserve is derived from:

```text
r_tok_new
=
k
÷
r_sol_new
```

Then:

```text
tokens_out
=
r_tok
minus
r_tok_new
```

The program performs this calculation with integer arithmetic.

## 📈 Why Price Rises

As users buy:

```text
Real SOL reserve rises

Sale inventory falls
```

Therefore:

```text
r_sol increases

r_tok decreases
```

Each additional token becomes progressively more expensive along the curve.

That is the price discovery mechanism during bonding.

## 🏦 Where the Buy Goes

For the bonding portion of a buy:

```text
Effective WSOL
      ↓
Moonz WSOL treasury

Creator fee
      ↓
Creator fee vault

Platform fee
      ↓
Platform fee account

Moonz tokens
      ↓
Buyer token account
```

The purchased Moonz tokens come from the sale vault.

## 📊 State Updates After a Buy

After the bonding transfer succeeds:

```text
tokens_sold
increases

sol_collected
increases
```

Specifically:

```text
tokens_sold
+=
bonding tokens out

sol_collected
+=
effective bonding WSOL
```

The trade fee is not added to `sol_collected`.

`sol_collected` tracks the effective curve quote amount.

## 🔴 Bonding Sell

Users can also sell Moonz tokens back while the token is in the curve phase.

Conceptually:

```text
Moonz tokens in
      ↓
Curve gross WSOL output
      ↓
Trading fee
      ↓
Net WSOL to seller
```

The tokens return to the sale vault.

## 🧮 Sell Mathematics

For a sell:

```text
r_tok_new
=
r_tok
+
tokens_in
```

The program then calculates:

```text
r_sol_new
=
k
÷
r_tok_new
```

and:

```text
gross WSOL out
=
r_sol
minus
r_sol_new
```

Moonz uses ceiling division for the new SOL reserve on sells.

## 🛡️ Why Ceiling Division Matters

Blockchain arithmetic uses integers.

The sell path deliberately rounds the reserve calculation so integer rounding cannot cause the protocol to overpay quote output.

That keeps the constant product accounting conservative at the smallest units.

## 💸 Sell Fee

For a bonding sell, the curve first determines:

```text
gross WSOL output
```

Then the:

```text
1.25 percent
```

trade fee is calculated.

The seller receives:

```text
net WSOL
=
gross WSOL
minus
trade fee
```

## 🔁 State Updates After a Sell

When a bonding sell succeeds:

```text
tokens_sold
decreases

sol_collected
decreases
```

The returned tokens increase the inventory available for later bonding buys.

So progress through bonding can move in both directions.

## 🧭 Bonding Is Not Just a Percentage Counter

A frontend might display bonding progress as:

```text
tokens sold
÷
sale supply
```

but the protocol itself maintains a real two sided market.

Buyers can move forward along the curve.

Sellers can move back along it.

## 👑 The Dev Buy Uses the Same Curve

The creator Dev Buy happens before the public bonding phase begins.

It uses the same core bonding calculation.

Conceptually:

```text
Initial curve state
      ↓
Creator Dev Buy
      ↓
tokens_sold increases
      ↓
sol_collected increases
      ↓
Phase becomes Curve
```

So the first public bonding trade sees the state left by the creator Dev Buy.

## 🌒 Public Trading Does Not Start at Zero

This distinction matters.

The raw launch state initially has:

```text
tokens_sold = 0

sol_collected = 0
```

But the creator Dev Buy occurs before the token enters normal `Curve` trading.

Therefore public traders generally encounter a curve that has already moved from its initial mathematical position.

## 🛡️ Minimum Output

A buy instruction includes:

```text
min_tokens_out
```

A sell instruction includes:

```text
min_wsol_out
```

If current execution cannot satisfy the minimum, the transaction fails rather than accepting a worse result.

The Trading SDK calculates these protected values for integrations.

## 🧱 Minimum Trade Sizes

The current program also rejects tiny dust trades below its protocol minimums.

For WSOL input:

```text
10,000 lamports
```

which is:

```text
0.00001 SOL
```

For Moonz token input:

```text
1,000 raw units
```

With 6 token decimals, that is:

```text
0.001 Moonz
```

## 🌗 Sale Remaining

The core bonding inventory relationship is:

```text
sale_remaining
=
650M
minus
tokens_sold
```

As that number approaches zero, the bonding phase approaches migration.

## 🌕 Reaching the End

When a buy consumes the final sale inventory, Moonz verifies:

```text
sale vault amount = 0

tokens_sold = sale_supply

WSOL treasury has liquidity

LP vault has liquidity
```

The program then changes the lifecycle to:

```text
AmmLive
```

with WSOL as the initial AMM quote asset.

## 🌉 An Overbuy Does Not Waste the Remaining Input

A buy can be larger than the amount required to finish the bonding inventory.

Moonz detects when the curve calculation would request more tokens than remain.

It then determines the exact effective WSOL required to buy the remaining curve tokens.

Conceptually:

```text
Large buy
     ↓
Enough to finish bonding?
     ↓
YES
     ↓
Calculate exact quote
for remaining sale tokens
     ↓
Finish curve
     ↓
Enter AMM
```

## 🚀 The Leftover Can Continue Into the AMM

After migration, if the original buy still has unused gross WSOL, Moonz can use that remaining portion against the new AMM.

That gives one trade a possible path of:

```text
Bonding
   ↓
Finish sale inventory
   ↓
Migration
   ↓
AMM
   ↓
Additional tokens
```

The public Trading SDK describes this market path as:

```text
BONDING_TO_AMM
```

## 🛡️ One Protected Trade

The caller still supplies one minimum token output.

The protocol checks the total output against that protection.

Applications do not need to manually split the user's order into a curve transaction and an AMM transaction.

## 🧠 Do Not Recreate the Curve in Your Frontend

It can be tempting to copy the equations into an interface.

That creates unnecessary risk because the protocol also contains:

```text
Integer rounding

Trading fees

Exact output boundary math

Sale inventory limits

Treasury checks

Curve crossing

AMM continuation

Minimum trade sizes
```

Use the public SDK instead.

## 💬 Quote a Bonding Buy

With the Trading SDK:

```ts
const quote =
  await trading.quoteBuy({
    mint,
    amount:
      "0.5",
    slippageBps:
      100
  });

console.log(
  quote.market
);

console.log(
  quote.expectedTokensOut
);
```

While the token is still bonding:

```text
quote.market
=
BONDING
```

unless the requested buy crosses the migration boundary.

## 🔴 Quote a Bonding Sell

```ts
const quote =
  await trading.quoteSell({
    mint,
    amount:
      "100000",
    slippageBps:
      100
  });

console.log(
  quote.expectedQuoteOut
);
```

During bonding the quote asset is SOL.

## 🔎 Read Bonding State

The core state can also be inspected through the Moonz SDK.

Useful values include:

```text
sale supply

tokens sold

SOL collected

current phase
```

This is useful for:

```text
Bonding progress

Analytics

Market displays

Indexers

Token explorers
```

## 🌌 Bonding in One Picture

The curve can be thought of as:

```text
            BUY

Gross WSOL
    ↓
1.25% fee
    ↓
Effective WSOL
    ↓
117 virtual SOL
+
real curve SOL
    ↓
CONSTANT PRODUCT
    ↑
760M virtual tokens
+
remaining sale tokens
    ↓
Moonz tokens out
    ↓
tokens_sold rises


           SELL

Moonz tokens in
    ↓
remaining sale inventory rises
    ↓
CONSTANT PRODUCT
    ↓
Gross WSOL out
    ↓
1.25% fee
    ↓
Net WSOL
    ↓
tokens_sold falls
```

{% hint style="success" %}
Virtual reserves shape the market.

Real sale inventory and collected WSOL move through it.

When the 650 million sale allocation is exhausted, the same Moonz lifecycle continues into its AMM.
{% endhint %}

## 🌊 Next Stop

The bonding curve eventually runs out of sale inventory.

That does not end the market.

It changes the market.

Next stop:

**Migration and AMM**
