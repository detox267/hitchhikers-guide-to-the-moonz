# 🌙 Token Lifecycle

A Moonz token does not use one market forever.

It moves through a protocol controlled lifecycle.

Understanding that lifecycle makes everything else in Moonz easier to understand.

{% hint style="info" %}
The Launch State is the source of truth for where a Moonz token currently is.
{% endhint %}

## 🧭 The Four On Chain Phases

The Moonz program defines four lifecycle phases:

```text
PendingDevBuy = 0

Curve = 1

AmmLive = 2

Switching = 3
```

At a high level:

```text
PENDING DEV BUY
       ↓
     CURVE
       ↓
    AMM LIVE
       ↕
   SWITCHING
```

The first two transitions happen as part of launch and bonding.

`Switching` exists after AMM migration when PCLS is changing the active quote asset.

## 🧬 Launch State

Each Moonz token has a canonical Launch State PDA.

It stores the protocol state needed to understand the token, including:

```text
Mint

Creator

Lifecycle phase

Sale supply

Tokens sold

SOL collected

Sale vault

LP vault

WSOL treasury

USDC treasury

Active quote asset

Pending quote asset

Pool switch state

Last trade time
```

For integrations, this account is the core protocol record for a Moonz launch.

## 🚀 Before Trading

A newly initialized launch begins in:

```text
PendingDevBuy
```

On chain that is:

```text
state = 0
```

The program also begins with:

```text
quote_asset = WSOL
```

and:

```text
pending_quote_asset = WSOL
```

At this stage the launch is still completing its creation sequence.

## 🔐 Trading Is Gated

Moonz does not allow normal trading simply because the Launch State exists.

Before trading is reachable, the program requires:

```text
Metadata initialized

Mint finalized
```

Mint finalization permanently removes:

```text
MintTokens authority

FreezeAccount authority
```

So the tradeable lifecycle begins only after the token has passed the required launch finalization checks.

{% hint style="success" %}
Moonz trading is gated behind metadata creation and permanent mint finalization.
{% endhint %}

## 💰 Dev Buy Comes First

The creator Dev Buy executes while the token is still in:

```text
PendingDevBuy
```

The program requires:

```text
Launch escrow initialized

Launch escrow not already executed

Launch escrow not refunded

state = PendingDevBuy

dev_buy_done = false
```

The Dev Buy uses the same bonding curve mechanics that begin the token's price discovery.

## 🌒 Entering the Curve

After the Dev Buy succeeds, the program records:

```text
dev_buy_done = true

state = Curve
```

This is the point where the token enters its normal bonding phase.

The public Moonz SDK presents this phase as:

```text
BONDING
```

So developers may encounter two names for the same lifecycle stage:

```text
On chain program
Curve

Public SDK
BONDING
```

They refer to the same phase.

## 📈 Bonding

During `Curve`, users buy and sell against Moonz bonding curve state.

The Launch State tracks:

```text
sale_supply

tokens_sold

sol_collected
```

A useful relationship is:

```text
Sale remaining
=
sale_supply
minus
tokens_sold
```

As users buy, `tokens_sold` rises.

As users sell back into the curve, `tokens_sold` can fall.

## 🪙 Fixed Token Supply

Every Moonz launch uses:

```text
1,000,000,000 total tokens
```

with:

```text
6 decimals
```

The supply is divided into:

```text
650,000,000
Sale allocation

350,000,000
LP allocation
```

Together:

```text
650,000,000
+
350,000,000
=
1,000,000,000
```

## 🌗 The Sale Allocation

The:

```text
650,000,000
```

sale allocation is the inventory used during bonding.

It lives in the Moonz sale vault.

The Launch State records its base unit amount as:

```text
sale_supply
```

and tracks how much has moved through bonding with:

```text
tokens_sold
```

## 🌊 The LP Allocation

The remaining:

```text
350,000,000
```

tokens form the LP allocation.

These tokens are held separately from the bonding sale allocation.

They become the token side of the Moonz AMM after bonding completes.

## 🌕 Bonding Completion

Migration occurs when the sale vault has been exhausted.

The program verifies:

```text
sale_vault amount = 0

tokens_sold = sale_supply

WSOL treasury has liquidity

LP vault has tokens
```

It then moves the lifecycle into:

```text
AmmLive
```

and emits the Moonz migration event.

## 🚀 Migration Is Built Into the Buy Path

A user does not need to send a separate public migration transaction simply because their buy reaches the end of bonding.

A buy can consume the final bonding inventory and move the token into the AMM as part of the trade path.

Conceptually:

```text
User buy
   ↓
Bonding inventory available
   ↓
Consume remaining sale tokens
   ↓
Sale vault reaches zero
   ↓
Moonz enters AMM
   ↓
Any applicable remaining buy
continues against AMM
```

This is why the Trading SDK can expose:

```text
BONDING_TO_AMM
```

for a buy that crosses the boundary.

## 🌉 One Buy Can Cross the Boundary

Suppose a user submits a buy large enough to consume the remaining sale allocation.

Moonz can split the economic path between:

```text
Final bonding portion
       +
AMM portion
```

while still protecting the minimum output for the overall trade.

This allows the market transition to happen without asking the user to manually calculate two independent trades.

## 🌕 AMM LIVE

After migration:

```text
state = AmmLive
```

The initial active quote asset is:

```text
WSOL
```

The public interfaces normally present that to users as:

```text
SOL
```

The protocol itself uses the WSOL SPL token account representation for trading.

## 🔄 AMM Trading

Once `AmmLive`, buys and sells use real pool reserves.

The important reserves are conceptually:

```text
Quote reserve

Moonz token reserve
```

For the initial SOL market those are represented through:

```text
WSOL treasury

LP token vault
```

The AMM uses constant product style reserve math.

We will cover that separately.

## 💵 The AMM Can Use USDC

`AmmLive` does not permanently mean SOL.

Moonz supports two quote assets:

```text
WSOL = 0

USDC = 1
```

The public SDK displays these as:

```text
SOL

USDC
```

The active quote asset is stored in:

```text
quote_asset
```

inside Launch State.

## 🛣️ Why Integrations Must Read State

A Moonz token that was using SOL yesterday may use USDC later.

A token that is bonding now may be an AMM token later.

So applications should not permanently cache assumptions such as:

```text
This token is bonding

This token always trades in SOL

This token uses this route forever
```

Instead:

```text
Read current Launch State
        ↓
Determine phase
        ↓
Determine quote asset
        ↓
Choose current behaviour
```

The Moonz SDK and Trading SDK already perform this state aware routing for their respective operations.

## 🔀 SWITCHING

After a token is `AmmLive`, its creator can initiate a supported quote asset switch.

The program moves the state into:

```text
Switching
```

On chain:

```text
state = 3
```

The requested destination asset is recorded in:

```text
pending_quote_asset
```

while:

```text
quote_asset
```

still identifies the currently active asset until completion.

## ⏸️ Switching Is Not a Trading Market

`Switching` is a protocol control state.

It is not another AMM.

Normal trading routes require `Curve` or `AmmLive`.

So integrations should treat:

```text
Switching
```

as a temporary non trading lifecycle state rather than trying to price it as a new market type.

## 🔁 Completing a Switch

When a pool switch completes successfully:

```text
quote_asset
=
pending_quote_asset
```

The switch tracking fields are cleared and the token returns to:

```text
AmmLive
```

The AMM is live again using its new quote asset.

## ↩️ Cancelled or Aborted Switches

A switch can also return to the existing quote asset if it cannot safely complete.

The program restores:

```text
pending_quote_asset
=
quote_asset
```

clears switch state and returns the lifecycle to:

```text
AmmLive
```

The detailed PCLS rules are covered later in this section.

## 🧭 Lifecycle Versus Market

It helps to separate these concepts.

```text
Lifecycle phase
      ↓
Where the token is
in the protocol

Market
      ↓
How a trade is
currently priced
```

For example:

```text
Curve
      ↓
Bonding market

AmmLive + WSOL
      ↓
SOL AMM

AmmLive + USDC
      ↓
USDC AMM

Switching
      ↓
No normal trade route
```

## 🛰️ SDK Naming

If you are reading Moonz through `@moonz-fun/sdk`, the developer friendly phase names include:

```text
PENDING_DEV_BUY

BONDING

AMM_LIVE

SWITCHING
```

The important naming difference is:

```text
Program:
Curve

SDK:
BONDING
```

Do not treat those as different protocol phases.

## 🔎 Read the Current Token

A developer can use the Moonz SDK:

```ts
const token =
  await moonz.getToken(
    mint
  );

console.log(
  token.phase
);

console.log(
  token.quoteAsset
);
```

That gives an application a developer friendly representation of the current lifecycle.

## ⚡ Trading SDK Handles the Route

For actual trading, the Trading SDK reads current state before constructing the transaction.

So integrations normally do not need logic such as:

```ts
if (bonding) {
  buildBondingBuy();
} else {
  buildAmmBuy();
}
```

Instead:

```ts
const quote =
  await trading.quoteBuy({
    mint,
    amount
  });
```

and:

```ts
await trading.buy({
  mint,
  wallet,
  amount
});
```

can follow the current Moonz route.

## 🌌 The Whole Journey

A normal Moonz token lifecycle looks like:

```text
TOKEN CREATED
      ↓
PENDING DEV BUY
      ↓
Creator Dev Buy
      ↓
CURVE
      ↓
Bonding buys and sells
      ↓
650M sale allocation
fully distributed
      ↓
MIGRATION
      ↓
AMM LIVE
      ↓
SOL AMM
      ↓
Optional PCLS
      ↓
SWITCHING
      ↓
AMM LIVE
      ↓
SOL or USDC
```

{% hint style="success" %}
Moonz is not a bonding curve with an unrelated pool bolted onto the end.

The bonding market, migration, AMM and quote asset switching are stages of one protocol controlled lifecycle.
{% endhint %}

## 📈 Next Stop

Now that we know where a token can be, we can look at how the first tradeable market actually determines price.

Next stop:

**Bonding Curve**
