# 🔀 PCLS

PCLS means:

```text
Program Controlled
Liquidity Switch
```

It allows a live Moonz AMM to change its quote asset while keeping the token liquidity under Moonz program control.

A Moonz pool can operate as:

```text
MOONZ / SOL
```

and later become:

```text
MOONZ / USDC
```

or move back in the opposite direction.

{% hint style="success" %}
PCLS changes the quote side of the Moonz AMM.

It does not move the Moonz token inventory out of the protocol LP vault.
{% endhint %}

## 🌙 The Two Quote Assets

The Moonz program defines:

```text
QUOTE_ASSET_WSOL = 0

QUOTE_ASSET_USDC = 1
```

The public interfaces normally display WSOL mode as:

```text
SOL
```

So the supported switch directions are:

```text
SOL
↓
USDC

or

USDC
↓
SOL
```

## 🧬 Launch State Tracks Both Assets

The Launch State contains:

```text
quote_asset

pending_quote_asset
```

During normal AMM trading:

```text
quote_asset
=
active market asset
```

When a switch begins:

```text
quote_asset
=
current asset

pending_quote_asset
=
requested destination asset
```

The current asset is not replaced until the switch successfully completes.

## 🚦 PCLS Starts From AmmLive

A switch can begin only when:

```text
state
=
AmmLive
```

It cannot begin while a token is:

```text
PendingDevBuy

Curve

Switching
```

PCLS belongs to the AMM stage of the Moonz lifecycle.

## 👤 Creator Controlled Start

The instruction that begins the process is:

```text
begin_pool_switch
```

The creator account is a Solana signer.

The program also verifies:

```text
creator
=
Launch State creator
```

So an unrelated wallet cannot initiate a quote asset switch for someone else's Moonz token.

## 🎯 Target Must Be Different

The requested target must be a valid Moonz quote asset.

It must also differ from:

```text
quote_asset
```

So a SOL pool cannot request a switch to SOL.

Likewise a USDC pool cannot request a switch to USDC.

## ⏱️ 24 Hour Cooldown

The program defines:

```text
POOL_SWITCH_COOLDOWN_SECONDS
=
86400
```

which is:

```text
24 hours
```

The first successful switch establishes:

```text
last_pool_switch_ts
```

Before a later switch can begin, Moonz requires at least 24 hours to have elapsed since that recorded successful switch time.

## 🧠 First Switch Has No Previous Completion Time

A newly migrated token begins with:

```text
last_pool_switch_ts
=
0
```

The cooldown check applies once a previous successful pool switch timestamp exists.

## 💰 Fixed Switch Fee

Starting PCLS requires:

```text
POOL_SWITCH_FEE_LAMPORTS
=
500000000
```

That is:

```text
0.5 SOL
```

This is a fixed PCLS control fee.

It is separate from:

```text
1.25 percent trading fee

0.04 SOL launch fee
```

## 🔐 The Fee Is Escrowed First

When PCLS begins, the creator transfers the 0.5 SOL fee into the Launch State account.

The program records:

```text
switch_fee_escrowed_lamports
=
POOL_SWITCH_FEE_LAMPORTS
```

The fee is therefore not immediately treated as a successful switch payment.

Its destination depends on how the switch ends.

## 🔀 Entering Switching

After the fee is escrowed, Moonz records:

```text
pending_quote_asset
=
target_quote_asset

switch_started_at
=
current time

switch_swap_executed
=
false

state
=
Switching
```

It then emits:

```text
PoolSwitchStartedEvent
```

## 📡 PoolSwitchStartedEvent

The start event contains:

```text
mint

creator

from_asset

to_asset

switch_fee_lamports
```

This lets indexers observe the intended transition before reserve conversion has taken place.

## ⏸️ Trading During Switching

`Switching` is not a normal trade route.

Normal AMM trading requires:

```text
AmmLive
```

So once PCLS begins:

```text
AmmLive
   ↓
Switching
   ↓
Normal AMM trading unavailable
```

Trading resumes after the switch completes or the pending switch is cancelled or aborted.

## 🏦 What PCLS Actually Changes

The Moonz token side remains:

```text
lp_vault
```

PCLS changes the quote reserve.

For example:

```text
BEFORE

WSOL treasury
      ×
Moonz LP vault


PCLS


AFTER

USDC treasury
      ×
Moonz LP vault
```

The Moonz token reserve stays inside the same protocol controlled LP vault.

## 🔒 Sale and LP Inventory Are Protected

The PCLS swap path explicitly prevents the external swap instruction from touching:

```text
sale_vault

lp_vault
```

The quote conversion is restricted to the selected source and destination quote vaults.

Conceptually:

```text
Allowed

WSOL treasury
      ↓
USDC treasury


Not allowed

Sale vault
LP vault
```

## 🌊 Source and Destination Vaults

For:

```text
WSOL
↓
USDC
```

the source must be the canonical:

```text
treasury_wsol_vault
```

and the destination must be:

```text
treasury_usdc_vault
```

For the reverse direction:

```text
USDC
↓
WSOL
```

the source and destination are reversed.

## 🛡️ Both Vaults Stay Under Launch State Authority

Moonz verifies both quote vaults are owned by:

```text
launch_state
```

The swap therefore does not hand custody of the protocol quote reserve to the creator.

## 🪐 Jupiter v6

The current approved swap program is:

```text
JUPITER_V6_PROGRAM_ID

JUP6LkbZbjS1jKKwapdHNy74zcZ3tLUZoi5QNyVTaV4
```

The program does not accept an arbitrary swap program for PCLS.

The supplied swap program must be executable and must match the approved Jupiter v6 program ID.

## 🧭 Platform Executes the Reserve Conversion

The reserve conversion instruction is:

```text
execute_pool_switch_swap
```

This stage requires:

```text
platform_signer
=
PLATFORM_WALLET
```

The creator initiates the switch.

The Moonz platform execution path performs the constrained quote reserve conversion.

## 🧱 CPI Limits

Moonz places explicit limits on the Jupiter CPI.

The maximum remaining account count is:

```text
MAX_SWITCH_REMAINING_ACCOUNTS
=
64
```

The maximum supplied swap data length is:

```text
MAX_SWITCH_SWAP_DATA_LEN
=
8192 bytes
```

A route that cannot fit the protocol boundary should not be executed through PCLS.

## 🛡️ No Extra Signers

The PCLS account validation prevents arbitrary additional signers from being introduced through the Jupiter account list.

The only signer Jupiter should see from the protocol controlled route is the Launch State PDA.

The Moonz program signs for that PDA using program signer seeds.

## 🔐 Launch State Signs the CPI

The signer seeds are based on:

```text
"launch_state"

mint

Launch State bump
```

Moonz then invokes the approved swap instruction using:

```text
invoke_signed
```

This allows the program controlled Launch State authority to authorize movement from the quote treasury.

## 🚫 Jupiter Cannot Touch Arbitrary Moonz Vaults

Moonz checks token accounts passed through the swap account list.

If a token account is owned by the Launch State PDA, it must be either:

```text
source_quote_vault

or

destination_quote_vault
```

This prevents PCLS from using the Jupiter CPI to reach unrelated program controlled token accounts.

## 💱 amount_in

The execution instruction receives:

```text
amount_in
```

Moonz first verifies the source treasury has at least that amount available.

After execution it checks that the source balance decreased by exactly:

```text
amount_in
```

So the actual source movement must match the declared swap amount.

## 🛡️ min_amount_out

The switch also receives:

```text
min_amount_out
```

After the Jupiter CPI completes, Moonz reloads both quote vaults and measures the destination increase.

It requires:

```text
destination increase
>=
min_amount_out
```

If that protection is not satisfied, the instruction fails.

## 🧹 Source Dust Limit

The program defines:

```text
SWITCH_DUST_LIMIT
=
10000
```

After a successful reserve conversion, the old quote treasury must contain no more than that many raw units.

## ☀️ WSOL Dust

WSOL uses:

```text
9 decimals
```

So:

```text
10000 lamports
=
0.00001 SOL
```

is the maximum source dust allowed by the protocol.

## 💵 USDC Dust

USDC uses:

```text
6 decimals
```

So:

```text
10000 raw USDC units
=
0.01 USDC
```

is the maximum source dust allowed when USDC is the old quote asset.

## ✅ Successful Swap Validation

After Jupiter returns, Moonz verifies:

```text
Source balance did not increase

Destination balance did not decrease

Source remaining
<=
SWITCH_DUST_LIMIT

Source decrease
=
amount_in

Destination increase
>=
min_amount_out
```

Only then is the reserve conversion recorded as executed.

## 🧬 switch_swap_executed

After all conversion checks pass:

```text
switch_swap_executed
=
true
```

The program also updates:

```text
last_trade_ts
```

to the current timestamp.

## 📡 Swap Event

A successful reserve conversion emits:

```text
PoolSwitchSwapExecutedEvent
```

with:

```text
mint

executor

from_asset

to_asset

amount_in

amount_out

source_remaining

destination_balance
```

This gives an indexer a direct record of the quote reserve conversion.

## 🌕 Completion Is a Separate Stage

Executing the Jupiter conversion does not itself change:

```text
quote_asset
```

Moonz next calls:

```text
complete_pool_switch
```

The completion stage requires:

```text
state = Switching

switch_swap_executed = true
```

## 🔍 Completion Rechecks the Treasuries

Before activating the new market, Moonz checks the canonical WSOL and USDC treasuries again.

If the pending asset is USDC:

```text
USDC treasury
> 0

WSOL treasury
<= SWITCH_DUST_LIMIT
```

If the pending asset is WSOL:

```text
WSOL treasury
> 0

USDC treasury
<= SWITCH_DUST_LIMIT
```

So a successful CPI alone is not enough.

The final reserve state must also satisfy the Moonz completion contract.

## 💸 Successful Fee Settlement

On successful completion, Moonz verifies:

```text
switch_fee_escrowed_lamports
=
POOL_SWITCH_FEE_LAMPORTS
```

The escrowed:

```text
0.5 SOL
```

is then transferred from the Launch State account to the canonical platform fee receiver.

## 🔄 Activate the New Quote Asset

After successful completion:

```text
quote_asset
=
pending_quote_asset
```

Moonz then records:

```text
last_pool_switch_ts
=
current time
```

and clears:

```text
switch_started_at

switch_fee_escrowed_lamports

switch_swap_executed
```

Finally:

```text
state
=
AmmLive
```

## 📡 Completion Event

Moonz emits:

```text
PoolSwitchCompletedEvent
```

with:

```text
mint

creator

new_asset
```

At that point the new quote asset is the active Moonz AMM market.

## 🌌 Successful PCLS Journey

```text
AmmLive
SOL
   ↓
Creator starts PCLS
   ↓
0.5 SOL fee escrowed
   ↓
pending = USDC
   ↓
Switching
   ↓
Trading unavailable
   ↓
Platform executes
constrained Jupiter v6 swap
   ↓
WSOL treasury
converted to USDC
   ↓
Source dust checked
   ↓
Minimum output checked
   ↓
Complete switch
   ↓
0.5 SOL fee
to platform
   ↓
quote_asset = USDC
   ↓
AmmLive
USDC
```

## ↩️ Creator Cancellation

PCLS also has a creator recovery path.

The instruction is:

```text
cancel_pool_switch
```

The creator must sign.

Cancellation is available only while:

```text
state = Switching

switch_swap_executed = false
```

So the creator cannot use this path to cancel after the reserve conversion has successfully executed.

## ⏳ Cancellation Timeout

The program defines:

```text
POOL_SWITCH_CANCEL_TIMEOUT_SECONDS
=
1800
```

which is:

```text
30 minutes
```

The creator cancellation path becomes available after that timeout if the swap has not executed.

## 🧭 Original Reserve Must Still Be Present

Cancellation checks that the original quote treasury still contains more than the dust threshold.

For a pending:

```text
WSOL
↓
USDC
```

switch, the WSOL treasury must still contain more than:

```text
SWITCH_DUST_LIMIT
```

For the reverse direction the same rule applies to the USDC treasury.

## 💰 Cancellation Refund

On creator cancellation, the escrowed:

```text
0.5 SOL
```

is returned to the creator.

Moonz then restores:

```text
pending_quote_asset
=
quote_asset
```

and clears the pending switch state.

Finally:

```text
state
=
AmmLive
```

Normal trading can resume on the original quote asset.

## 📡 Cancellation Event

Cancellation emits:

```text
PoolSwitchCancelledEvent
```

with:

```text
mint

creator

active_asset

cancelled_at
```

## 🚨 Immediate Invalid Route Abort

There is also a separate safety path:

```text
abort_pool_switch_route_invalid
```

This exists for a route that cannot be safely executed within the Moonz Jupiter CPI boundary.

Examples include a route that cannot fit the protocol account constraints.

## ⚡ No 30 Minute Wait for an Invalid Route

The invalid route abort is platform authorized.

It can restore the original market before the swap executes rather than leaving the token in `Switching` for the normal creator cancellation timeout.

The instruction requires:

```text
state = Switching

switch_swap_executed = false

platform signer authorized

creator matches Launch State
```

## 💰 Invalid Route Refund

The invalid route abort also returns the full escrowed:

```text
0.5 SOL
```

to the creator.

It resets:

```text
pending_quote_asset
=
quote_asset

switch_started_at
=
0

switch_fee_escrowed_lamports
=
0

switch_swap_executed
=
false

state
=
AmmLive
```

Trading therefore returns immediately to the original quote asset.

## 🧠 Successful Fee Versus Refunded Fee

The switch fee has two possible destinations.

```text
SUCCESSFUL PCLS

0.5 SOL
      ↓
Platform fee receiver


CANCELLED OR
INVALID ROUTE

0.5 SOL
      ↓
Creator refunded
```

This is why the fee is escrowed when the switch begins rather than immediately settled to the platform.

## 🔐 PCLS Does Not Give Jupiter Protocol Control

Jupiter provides the constrained quote asset conversion.

Moonz still controls:

```text
Who can start the switch

Who can execute it

Approved swap program

Source treasury

Destination treasury

Allowed protocol accounts

Signer boundary

Maximum account count

Swap data size

Input amount

Minimum output

Source dust

Completion rules

Fee settlement

Cancellation rules
```

Jupiter is the execution venue for the quote conversion, not the authority that determines Moonz protocol state.

## 🛡️ PCLS Does Not Move the LP Tokens

The Moonz LP token reserve remains in:

```text
lp_vault
```

through the quote asset switch.

The conversion is:

```text
Old quote reserve
      ↓
New quote reserve
```

not:

```text
Moonz LP tokens
      ↓
External pool
```

## 📊 What Developers Should Watch

For applications and indexers, the important state fields are:

```text
state

quote_asset

pending_quote_asset

last_pool_switch_ts

switch_started_at

switch_fee_escrowed_lamports

switch_swap_executed
```

Together they describe the current PCLS state.

## 📡 What Indexers Should Watch

The canonical PCLS events are:

```text
PoolSwitchStartedEvent

PoolSwitchSwapExecutedEvent

PoolSwitchCompletedEvent

PoolSwitchCancelledEvent
```

The public Moonz SDK exposes corresponding normalized pool switch events.

## 🛣️ Trading Applications Should Follow Current State

Do not permanently assume a Moonz AMM uses SOL.

After PCLS:

```text
Yesterday
SOL

Today
USDC
```

The Trading SDK handles that automatically by reading current Launch State before quoting or building a trade.

## 🌕 After a Successful Switch

Suppose a Moonz token changes from SOL to USDC.

The lifecycle becomes:

```text
AmmLive
+
quote_asset = USDC
```

Then the Trading SDK routes buys through:

```text
amm_buy_usdc
```

and sells through:

```text
amm_sell_usdc
```

If the creator later switches back to SOL, normal WSOL AMM trading becomes active again.

## 🧭 PCLS in One Picture

```text
             CREATOR

                ↓

       begin_pool_switch

                ↓

          0.5 SOL escrow

                ↓

            SWITCHING

                ↓

      Normal trading paused

                ↓

        Moonz controlled
        Jupiter v6 CPI

                ↓

      Old quote treasury
               to
      New quote treasury

                ↓

         Safety checks

       amount_in exact

       min_amount_out

       dust threshold

       vault authority

                ↓

         COMPLETE?
         /       \
       YES        NO
        ↓          ↓
New quote active   No swap executed
        ↓          ↓
0.5 SOL platform  Creator cancel
        ↓          or invalid route abort
     AmmLive        ↓
                  0.5 SOL refund
                     ↓
                 Original quote
                     ↓
                   AmmLive
```

{% hint style="success" %}
PCLS lets liquidity evolve without surrendering protocol control.

The quote reserve can change between SOL and USDC while Moonz keeps the LP inventory, vault authority, lifecycle and safety rules inside the program.
{% endhint %}

## 🛡️ Next Stop

We now understand the Moonz lifecycle from launch through bonding, AMM and PCLS.

The final step is to collect the guarantees the protocol enforces across all of those stages.

Next stop:

**Protocol Guarantees**
