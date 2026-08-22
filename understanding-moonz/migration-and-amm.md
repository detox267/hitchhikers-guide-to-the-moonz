# 🌊 Migration and AMM

The bonding curve is only the first Moonz market.

When the entire sale allocation has been distributed, Moonz does not send the token somewhere else to find liquidity.

The protocol changes market mode.

The same Launch State moves from:

```text
Curve
```

to:

```text
AmmLive
```

and the Moonz controlled liquidity vaults become the live AMM.

{% hint style="success" %}
Moonz migration is a protocol state transition.

The token does not leave Moonz to create an unrelated external pool.
{% endhint %}

## 🌗 What Exists Before Migration

Long before bonding finishes, Moonz has already created the accounts needed for the later AMM.

During launch initialization the protocol creates:

```text
Sale vault

LP vault

WSOL treasury

USDC treasury
```

The Launch State records their addresses.

## 🪙 The LP Tokens Already Exist

At launch initialization the fixed token supply is minted.

The allocation is:

```text
650,000,000 tokens
      ↓
Sale vault

350,000,000 tokens
      ↓
LP vault
```

The LP allocation therefore does not need to be minted when bonding finishes.

It has already been reserved inside the Moonz controlled LP vault.

## 🔒 No New Supply at Migration

Migration does not mint another:

```text
350,000,000 tokens
```

The total supply remains:

```text
1,000,000,000 tokens
```

with:

```text
6 decimals
```

The later AMM uses the LP allocation that was locked into the launch structure from the beginning.

## 💰 The Quote Side Builds During Bonding

While the token is in `Curve`, effective bonding WSOL is accumulated in the Moonz WSOL treasury.

Conceptually:

```text
Bonding buys
      ↓
Trading fee separated
      ↓
Effective WSOL
      ↓
WSOL treasury
```

So by the time the sale allocation is exhausted, Moonz already has both sides required for the initial AMM:

```text
Quote side
WSOL treasury

Token side
LP vault
```

## 🌕 The Migration Trigger

Migration occurs when:

```text
sale_vault.amount = 0
```

The program also verifies:

```text
tokens_sold
=
sale_supply
```

and requires both:

```text
WSOL treasury amount > 0

LP vault amount > 0
```

Only then does the protocol enter the AMM state.

## 🔄 State Transition

At migration Moonz sets:

```text
quote_asset
=
WSOL

pending_quote_asset
=
WSOL

state
=
AmmLive
```

It also emits:

```text
MigratedEvent
```

with the token mint.

## 🧠 What Migration Does Not Do

The migration path does not need to:

```text
Mint new LP inventory

Move the token to an external DEX

Create an unrelated external pool

Ask the creator to provide liquidity

Ask holders to migrate tokens
```

The required Moonz liquidity accounts already exist.

Migration activates the next market mode.

## 🌊 Initial AMM Reserves

Immediately after migration, AMM pricing reads the actual vault balances.

For the WSOL market:

```text
Quote reserve
=
treasury_wsol_vault.amount

Token reserve
=
lp_vault.amount
```

These are real SPL token account balances.

## 🔍 Actual Reserves Matter

During bonding, developers may care about:

```text
tokens_sold

sol_collected
```

After migration, AMM pricing is based on:

```text
Actual quote treasury balance

Actual LP vault balance
```

The bonding counters are not substitutes for the live AMM reserve accounts.

{% hint style="info" %}
For an AMM market, read the current vault reserves.

Do not reconstruct the AMM from the old bonding counters.
{% endhint %}

## ✖️ Constant Product AMM

Moonz uses constant product reserve math.

Conceptually:

```text
k
=
quote reserve
×
token reserve
```

If we call the quote reserve:

```text
x
```

and the Moonz reserve:

```text
y
```

then:

```text
k
=
x × y
```

Trades move the reserve balances along that curve.

## 🟢 AMM Buy

For an AMM buy, the trader provides the active quote asset.

That can be:

```text
WSOL

or

USDC
```

The high level shape is:

```text
Gross quote input
      ↓
1.25 percent trading fee
      ↓
Trade amount
      ↓
AMM reserve calculation
      ↓
Moonz tokens out
```

## 🧮 Buy Math

For the pricing calculation:

```text
x
=
current quote reserve

y
=
current token reserve

trade
=
gross input
minus total fee
```

The invariant is:

```text
k
=
x × y
```

Then:

```text
x_new
=
x + trade
```

and:

```text
y_new
=
ceil(
  k / x_new
)
```

Finally:

```text
tokens_out
=
y - y_new
```

## 🛡️ Conservative Integer Rounding

Moonz uses ceiling division for AMM output calculations.

This ensures integer rounding does not cause the pool to overpay token output.

Blockchain reserves are integer base units, so the rounding direction is part of the protocol economics.

## 💸 The AMM Fee

Every Moonz trade currently charges:

```text
125 basis points
=
1.25 percent
```

For an AMM trade the total fee is split between:

```text
Platform

Creator

LP reserve
```

We will map the exact percentages on the Fees page.

## 🌱 The LP Fee Stays With Liquidity

An important AMM distinction is that the LP portion remains with the pool.

For a buy:

```text
Gross quote input
      ↓
Trade amount
+
LP fee
      ↓
Quote treasury
```

while the creator and platform portions are directed to their respective fee accounts.

## 🟢 Buy Reserve Movement

An AMM buy therefore moves the reserves like:

```text
Quote reserve
      ↑
increases

Token reserve
      ↓
decreases
```

The user receives Moonz tokens from the LP vault.

## 🔴 AMM Sell

An AMM sell moves in the opposite direction.

The trader supplies Moonz tokens.

```text
Moonz tokens in
      ↓
Gross quote output
      ↓
Trading fee
      ↓
Net quote output
      ↓
Seller
```

The sold tokens move into the LP vault.

## 🧮 Sell Math

For a sell:

```text
x
=
quote reserve

y
=
token reserve
```

After adding the sold Moonz tokens:

```text
y_new
=
y + tokens_in
```

Moonz calculates:

```text
x_new
=
ceil(
  k / y_new
)
```

Then:

```text
gross quote out
=
x - x_new
```

The trading fee is calculated from that gross quote output.

## 💰 Seller Output

The seller receives:

```text
net quote output
=
gross quote output
minus
total trading fee
```

The minimum output supplied with the transaction must also be satisfied.

## 🌱 LP Fee on a Sell

On AMM sells, the LP fee is not withdrawn from the quote treasury.

Conceptually:

```text
Gross quote output
      ↓
Seller net

Creator fee
      ↓
Creator fee vault

Platform fee
      ↓
Platform account

LP fee
      ↓
Remains in pool
```

That retained LP fee grows protocol controlled liquidity over trading activity.

## 🔁 Sell Reserve Movement

An AMM sell therefore causes:

```text
Token reserve
      ↑
increases

Quote reserve
      ↓
decreases
```

while the LP portion of the fee remains inside the quote reserve.

## 🌊 Protocol Controlled Liquidity

There is no user LP token that needs to be deposited to make this Moonz market function.

The pool inventory is held in the protocol controlled vault structure.

At the core are:

```text
Launch State PDA

LP token vault

Active quote treasury
```

The Launch State PDA is the authority over the protocol token vaults used by the AMM.

## 👤 Protocol Controlled Vault Authority

The LP vault is not owned by the creator wallet.

At the SPL token account level, Moonz requires:

```text
lp_vault.owner
=
launch_state.key()
```

The same authority model applies to the protocol quote treasuries.

For WSOL:

```text
treasury_wsol_vault.owner
=
launch_state.key()
```

For USDC:

```text
treasury_usdc_vault.owner
=
launch_state.key()
```

When Moonz transfers tokens out of the LP vault during an AMM buy, the Launch State PDA acts as the token authority through program signer seeds.

Conceptually:

```text
Launch State PDA
       ↓
Controls protocol vault authority
       ↓
LP token vault

WSOL treasury

USDC treasury
```

The creator address is recorded in Launch State, but the creator wallet is not the SPL token authority for these protocol liquidity vaults.

{% hint style="info" %}
Protocol controlled liquidity means movement from these vaults must satisfy Moonz program execution rather than a creator wallet signature alone.
{% endhint %}

## ☀️ WSOL AMM

When:

```text
state = AmmLive

quote_asset = WSOL
```

the reserves are:

```text
treasury_wsol_vault

lp_vault
```

The standard Moonz:

```text
buy

sell
```

instructions route through this WSOL AMM path.

## 💵 USDC AMM

Moonz can also operate with:

```text
quote_asset = USDC
```

In that mode the quote reserve becomes:

```text
treasury_usdc_vault
```

while the token side remains:

```text
lp_vault
```

The program exposes:

```text
amm_buy_usdc

amm_sell_usdc
```

for this market.

## 🧬 Same AMM Principle

The quote asset changes, but the core reserve model does not.

```text
WSOL mode

WSOL treasury
×
LP vault


USDC mode

USDC treasury
×
LP vault
```

Both use the same constant product style reserve calculations.

## 🛣️ Trading SDK Hides the Instruction Choice

Applications do not normally need to decide:

```text
buy

sell

amm_buy_usdc

amm_sell_usdc
```

themselves.

The Trading SDK reads the current Launch State and selects the current route.

For example:

```ts
const quote =
  await trading.quoteBuy({
    mint,
    amount:
      "0.5"
  });

console.log(
  quote.quoteAsset
);

console.log(
  quote.market
);
```

## 🌉 Crossing From Bonding Into AMM

The most interesting migration case occurs when one buy is large enough to finish bonding and still has quote input remaining.

Moonz handles this inside the buy path.

```text
Gross WSOL buy
      ↓
Use required portion
on bonding curve
      ↓
Sale vault reaches zero
      ↓
Enter AmmLive
      ↓
Calculate leftover gross WSOL
      ↓
Use leftover against
new WSOL AMM
```

## 🎯 Exact Curve Completion

If a buy would ask the curve for more tokens than remain, Moonz calculates the effective WSOL required to purchase exactly the remaining bonding inventory.

The protocol does not pretend additional sale vault tokens exist.

It caps the bonding portion at:

```text
sale_remaining
```

and calculates the quote required to finish that amount.

## 💸 Separate Fee Treatment Across the Boundary

A crossing buy can contain:

```text
Bonding portion
      +
AMM portion
```

The bonding portion uses the bonding fee split.

The leftover AMM portion uses the AMM fee split.

The total trade can therefore cross two fee allocation models inside the same user buy.

## 🛡️ One Minimum Output

At the program instruction level, the protected minimum for a buy is:

```text
min_tokens_out
```


Even when the buy crosses both markets, Moonz adds the token output together.

```text
Bonding tokens out
      +
AMM tokens out
      ↓
Total tokens out
```

The program then checks:

```text
total tokens out
>=
minimum tokens out
```

So the caller retains one slippage protection boundary for the complete buy.

## 📡 Migration Event

When the lifecycle changes to `AmmLive`, Moonz emits:

```text
MigratedEvent
```

This is useful to:

```text
Indexers

Trading interfaces

Analytics systems

Market trackers
```

that need to know a token has finished bonding.

## 📡 AMM Events

Normal AMM trades emit:

```text
AmmBuyEvent

AmmSellEvent
```

These events contain information including:

```text
Mint

User

Quote asset

Input amount

Output amount

Trading fee

Creator fee

Platform fee

LP fee

Timestamp
```

The Moonz SDK and Geyser tooling can be used to consume these public protocol events.

## 📊 Do Not Use tokens_sold as AMM Reserve

During bonding:

```text
tokens_sold
```

describes progress through the sale allocation.

Once the market becomes an AMM, live token reserve information comes from:

```text
lp_vault.amount
```

AMM buys and sells move tokens in and out of that vault.

They do not turn the bonding `tokens_sold` counter into an AMM reserve counter.

## 💰 Do Not Use sol_collected as the Live AMM Reserve

Likewise:

```text
sol_collected
```

belongs to the bonding state model.

For a live WSOL AMM, pricing reads:

```text
treasury_wsol_vault.amount
```

For a live USDC AMM:

```text
treasury_usdc_vault.amount
```

Those actual token account balances are the reserve source.

## 🔎 Read Market Data Through the SDK

For most applications, the easiest route is the Moonz SDK.

```ts
const market =
  await moonz.getMarketData(
    mint
  );

console.log(
  market.market
);

console.log(
  market.priceSource
);
```

Once the token is using AMM reserves, the public SDK can derive market information from those current protocol balances.

## 🌌 Migration in One Picture

```text
           BONDING

650M sale vault
      ↓
Users buy
      ↓
sale vault reaches zero

WSOL treasury
already contains
bonding liquidity

350M LP vault
already exists

        ↓

     MIGRATION

state = AmmLive
quote = WSOL

        ↓

          AMM

WSOL treasury
       ×
LP token vault

        ↓

Constant product trading

        ↓

LP fee remains
with liquidity

        ↓

Optional future PCLS

        ↓

WSOL or USDC AMM
```

{% hint style="success" %}
Moonz does not finish bonding and then search for a market.

The liquidity needed for the next stage is already inside the protocol structure, ready for the lifecycle to move into `AmmLive`.
{% endhint %}

## 💸 Next Stop

Every bonding and AMM trade charges the same headline trading fee.

Where that fee goes depends on the market stage.

Next stop:

**Fees**
