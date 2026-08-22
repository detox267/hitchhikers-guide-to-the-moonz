# 🛡️ Protocol Guarantees

Moonz has several moving parts.

Bonding.

AMM trading.

Creator fees.

Launch escrow.

PCLS.

The important question for an integrator is not only what these systems do.

It is what the program actually enforces.

{% hint style="info" %}
This page describes guarantees enforced by the current Moonz program logic.

It does not make promises about market price, RPC availability, external services or future program code.
{% endhint %}

## 🌙 Canonical Program

The Moonz mainnet program ID is:

```text
DBc9SEQghiJUj52YPqTKk8R4CMRgagBxi2LU1yBbeMpk
```

Moonz protocol accounts are derived and validated relative to this program.

## 🧬 Canonical Launch State

Each Moonz token has one canonical Launch State PDA derived from:

```text
"launch_state"

+

token mint
```

The program does not simply trust an arbitrary account that contains Launch State shaped data.

It also performs a runtime PDA check through:

```text
require_launch_state_pda
```

Conceptually:

```text
Recorded mint
      ↓
Derive expected Launch State PDA
      ↓
Compare with supplied Launch State
      ↓
Must match
```

## 🔒 Why Canonical State Matters

Without canonical state validation, an attacker could attempt to present unrelated accounts as protocol state.

Moonz instead ties the state identity to:

```text
Program ID

Launch State seed

Token mint
```

This is one of the core integrity boundaries used throughout trading and protocol control.

## 🪙 Fixed Launch Supply

A Moonz launch uses:

```text
1,000,000,000 tokens
```

with:

```text
6 decimals
```

The initial allocation is:

```text
650,000,000
Sale vault

350,000,000
LP vault
```

The program verifies:

```text
sale allocation
+
LP allocation
=
total supply
```

before minting those allocations.

## 🏦 Supply Is Minted Into Protocol Vaults

During launch initialization, the program mints:

```text
SALE_TOKENS
      ↓
sale_vault

LP_TOKENS
      ↓
lp_vault
```

The mint authority at this stage must be the canonical Moonz mint authority PDA.

## 🔐 Mint Authority Is Removed

After metadata has been created, Moonz finalizes the mint.

The program calls the SPL Token authority instruction with:

```text
AuthorityType::MintTokens

new authority = None
```

After that operation, the token no longer has an active mint authority.

That prevents additional tokens from being minted through the normal SPL Token authority model.

## ❄️ Freeze Authority Is Removed

The same finalization stage also removes:

```text
AuthorityType::FreezeAccount
```

by setting:

```text
new authority = None
```

This removes the token mint's ability to freeze holder token accounts through the SPL Token freeze authority.

## 🧱 Finalization Happens Once

Before finalization, Moonz requires:

```text
metadata_initialized = true

mint_finalized = false
```

After both token authorities have been removed:

```text
mint_finalized = true
```

The finalization path cannot simply be repeated as though the mint were still in its creation state.

## 🖼️ Metadata Is Created Immutable

Moonz creates Metaplex metadata using:

```text
CreateMetadataAccountV3
```

with:

```text
is_mutable = false
```

That means the metadata created through this launch path is marked immutable at creation.

## 🚦 Trading Requires Finalization

Moonz centralizes the launch readiness check through:

```text
require_launch_immutable_and_finalized
```

That check requires:

```text
metadata_initialized = true

mint_finalized = true
```

Normal trading and AMM control are not reachable until those launch conditions have been satisfied.

{% hint style="success" %}
A Launch State existing is not enough to make a Moonz token tradeable.

Metadata and mint finalization are part of the protocol gate.
{% endhint %}

## 🌗 Lifecycle Is Enforced

Moonz instructions verify the current lifecycle state.

The on chain phases are:

```text
PendingDevBuy

Curve

AmmLive

Switching
```

An instruction intended for one lifecycle stage cannot simply treat another stage as equivalent.

For example:

```text
Bonding
requires Curve

Normal AMM
requires AmmLive

PCLS execution
requires Switching
```

## ⏸️ Switching Blocks Normal Trading

PCLS temporarily changes the lifecycle to:

```text
Switching
```

Normal trade routes require:

```text
Curve

or

AmmLive
```

So a pool being switched is not treated as an ordinary live AMM until the switch completes or returns to the original market.

## 🏦 Sale Vault Is Canonical

Trading account validation ties:

```text
sale_vault
```

to the address recorded in Launch State.

It also requires:

```text
sale_vault.mint
=
launch_state.mint
```

and:

```text
sale_vault.owner
=
launch_state.key()
```

The creator wallet is not the SPL Token owner of the sale inventory.

## 🌊 LP Vault Is Canonical

The LP vault receives the same style of protection.

Moonz requires:

```text
lp_vault address
=
launch_state.lp_vault
```

with:

```text
lp_vault.mint
=
launch_state.mint
```

and:

```text
lp_vault.owner
=
launch_state.key()
```

The Moonz token side of AMM liquidity therefore remains under Launch State PDA authority.

## ☀️ WSOL Treasury Is Canonical

For WSOL trading, Moonz verifies the supplied treasury is the one recorded by Launch State.

The vault must contain:

```text
WSOL_MINT
```

and its token authority must be:

```text
launch_state.key()
```

An arbitrary WSOL token account cannot be substituted for the protocol treasury.

## 💵 USDC Treasury Is Canonical

The USDC market follows the same principle.

Moonz uses the canonical Solana USDC mint:

```text
EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v
```

and the USDC treasury remains part of the Launch State controlled vault structure.

## 🔑 PDA Authority Instead of Creator Custody

For protocol vault movement, Moonz uses the Launch State PDA as token authority.

Program signer seeds are based on:

```text
"launch_state"

mint

bump
```

This means:

```text
Creator wallet
≠
LP vault authority

Creator wallet
≠
Sale vault authority

Creator wallet
≠
Quote treasury authority
```

The creator participates only where the program specifically requires creator authorization.

## 👤 User Accounts Are Checked Too

Moonz verifies that user token accounts correspond to the expected user and mint.

For example, a buy requires the buyer token account owner to be:

```text
buyer
```

and its mint to be:

```text
launch_state.mint
```

Sell paths apply equivalent ownership checks for the seller.

## 💸 Canonical Fee Destinations

Fee transfers are not directed to arbitrary caller supplied wallets.

The program has a fixed:

```text
PLATFORM_FEE_WALLET
```

and checks the platform fee token account owner against that address.

## 🧬 Canonical Creator Fee Authority

Creator fee authority is derived from:

```text
CREATOR_FEES_SEED

+

creator public key
```

The seed is:

```text
"creator_fees"
```

This gives each creator a deterministic creator fee authority PDA.

## 🏦 Canonical Fee ATAs

Moonz derives the expected associated token account using:

```text
owner

Token Program

mint
```

and verifies fee accounts through:

```text
require_canonical_ata
```

This prevents an integration from substituting another token account merely because it holds the same mint.

## ✍️ Creator Fee Claims Require Creator Authorization

The creator fee claim path requires the creator as a signer.

The creator fee vault must also belong to the canonical creator fee authority.

So knowing the creator fee vault address is not sufficient to withdraw its balance.

## 🧮 Checked Arithmetic

Protocol accounting uses checked integer operations throughout critical calculations.

Examples include:

```text
checked_add

checked_sub

checked_mul

checked_div
```

Arithmetic failure maps to:

```text
MathOverflow
```

rather than intentionally relying on silent integer wraparound.

## 🛡️ Minimum Output Protection

Trade instructions carry protected minimum outputs.

Examples include:

```text
min_tokens_out

min_wsol_out

min_usdc_out
```

If execution cannot satisfy the user's minimum, Moonz rejects the trade rather than deliberately accepting a worse output.

The program error surface includes:

```text
SlippageExceeded
```

## 🧹 Minimum Trade Sizes

Moonz also rejects tiny trades below its protocol minimums.

Current values include:

```text
MIN_WSOL_TRADE_LAMPORTS
=
10000

MIN_USDC_TRADE_UNITS
=
10000

MIN_TOKEN_TRADE_UNITS
=
1000
```

These boundaries reduce dust trade and rounding abuse.

## 🌊 AMM Uses Actual Reserves

Once the token reaches:

```text
AmmLive
```

Moonz prices against real protocol vault balances.

For WSOL:

```text
treasury_wsol_vault.amount

lp_vault.amount
```

For USDC:

```text
treasury_usdc_vault.amount

lp_vault.amount
```

Bonding counters are not substituted for the live AMM reserve accounts.

## 📈 Bonding Cannot Sell More Inventory Than Exists

During bonding, the program tracks:

```text
sale_supply

tokens_sold
```

and the sale vault is the actual token inventory.

A buy reaching the end of the curve is capped against remaining sale inventory before migration into the AMM path.

## 🌉 Migration Requires Real Liquidity

Moonz does not enter the AMM merely because a frontend says bonding is complete.

The program verifies the actual migration conditions, including:

```text
Sale vault empty

tokens_sold
=
sale_supply

WSOL treasury liquidity

LP vault liquidity
```

Only then does the lifecycle move into:

```text
AmmLive
```

## 🔀 PCLS Preserves Vault Boundaries

During PCLS, Jupiter is not given unrestricted access to Moonz controlled token accounts.

The swap path restricts program controlled token accounts to the selected:

```text
source_quote_vault

destination_quote_vault
```

The Moonz:

```text
sale_vault

lp_vault
```

cannot be used as PCLS swap source or destination accounts.

## 🪐 Approved Jupiter Program Only

Moonz accepts the configured Jupiter v6 program:

```text
JUP6LkbZbjS1jKKwapdHNy74zcZ3tLUZoi5QNyVTaV4
```

for PCLS execution.

An arbitrary executable program cannot be substituted as the switch swap program.

## 🛡️ PCLS Checks Input and Output

After reserve conversion, Moonz verifies:

```text
Source decrease
=
amount_in

Destination increase
>=
min_amount_out

Old quote reserve
<=
SWITCH_DUST_LIMIT
```

The reserve switch is therefore checked by Moonz after the external CPI returns.

## 🔐 PCLS Fee Is Escrowed

The creator's:

```text
0.5 SOL
```

switch fee is stored in Launch State while the switch is pending.

Successful completion sends it to the platform fee receiver.

Cancellation or invalid route recovery returns it to the creator according to the PCLS rules.

## ↩️ Launch Escrow Has a Recovery Path

Creation funding is also escrowed before launch execution.

The program records:

```text
creator

mint

create fee

Dev Buy amount

deposit amount

creation time

execution status

refund status

initialization status
```

## ⏱️ Failed Launch Refund Timeout

The program defines:

```text
LAUNCH_REFUND_TIMEOUT_SECONDS
=
900
```

which is:

```text
15 minutes
```

If the launch has not been initialized or executed, and has not already been refunded, the creator can use the launch escrow refund path once the timeout has elapsed.

## 👤 Only the Recorded Creator Can Refund

The refund instruction verifies:

```text
launch_escrow.creator
=
creator signer
```

and:

```text
launch_escrow.mint
=
supplied mint
```

It also requires:

```text
executed = false

refunded = false

initialized = false
```

before the timeout based refund is available.

## 💰 Refund Returns the Escrow Balance

When the launch refund conditions are satisfied, Moonz transfers the refundable SOL from the canonical escrow SOL vault back to the creator.

It then records:

```text
refunded = true
```

and emits:

```text
LaunchEscrowRefundedEvent
```

## 📡 Events Make State Changes Observable

Moonz emits protocol events for important operations including:

```text
Launch escrow funding

Launch escrow refund

Bonding buys

Bonding sells

AMM buys

AMM sells

Migration

Creator fee claims

PCLS start

PCLS swap execution

PCLS completion

PCLS cancellation
```

These events allow external systems to observe the same state transitions enforced by the program.

## 🔎 Integrity Can Be Independently Checked

The public Moonz SDK exposes integrity checks around:

```text
Program owner

Launch State PDA

Sale vault

LP vault

Quote treasuries

Vault mint

Vault authority
```

That gives integrators a public way to verify that an account set corresponds to the expected Moonz protocol structure.

## ⚠️ What These Guarantees Do Not Mean

Protocol checks should not be confused with economic or infrastructure guarantees.

Moonz cannot guarantee:

```text
Token price will rise

A trader will make a profit

Liquidity will have a particular value

An RPC provider will always be online

A wallet will sign a transaction

A transaction will always land

Jupiter will always have a valid PCLS route

Network conditions will remain unchanged
```

Those are outside the scope of program invariant enforcement.

## 🧠 Source Logic Versus Deployment Authority

There is another important distinction.

This page describes what the current published program source enforces.

Whether the deployed Solana program itself can still be upgraded is a separate deployment property.

Do not infer:

```text
Token mint is immutable
```

to mean:

```text
Program deployment is immutable
```

Those are different authority systems.

## 🔐 Token Authority Versus Program Authority

For each launched token, Moonz removes:

```text
MintTokens authority

FreezeAccount authority
```

and creates immutable metadata.

Those are properties of the launched token.

The Solana program upgrade authority, if any, belongs to the deployed program itself and must be verified separately from token mint state.

## 🛰️ Developers Should Verify What Matters

For an integration, useful checks include:

```text
Correct Moonz Program ID

Canonical Launch State PDA

Correct token mint

Correct sale vault

Correct LP vault

Correct quote treasuries

Correct vault authorities

Current lifecycle phase

Current quote asset
```

The public Moonz SDK already provides these primitives.

## 🌌 Moonz Trust Boundary

The overall model can be summarized as:

```text
TOKEN CREATION

Fixed allocation
      ↓
Metadata immutable
      ↓
Mint authority removed
      ↓
Freeze authority removed

         ↓

TRADING

Canonical Launch State
      ↓
Canonical vaults
      ↓
Phase checks
      ↓
Minimum output
      ↓
Checked arithmetic

         ↓

LIQUIDITY

Sale vault
      ↓
Bonding
      ↓
Migration checks
      ↓
LP vault
+
Quote treasury
      ↓
AMM

         ↓

PCLS

Creator requests
      ↓
Fee escrow
      ↓
Switching
      ↓
Constrained Jupiter CPI
      ↓
Reserve checks
      ↓
Complete or recover

         ↓

OBSERVABILITY

Launch State
+
Vault balances
+
Protocol events
```

{% hint style="success" %}
Moonz integrations do not have to trust a frontend description of protocol state.

The important identities, authorities, lifecycle gates, vault relationships and execution boundaries are enforced by the program and can be independently inspected.
{% endhint %}

## 🌕 Understanding Moonz Complete

We have now followed a Moonz token through:

```text
Launch

Dev Buy

Bonding

Migration

AMM

Fees

PCLS

Protocol guarantees
```

The protocol layer is now mapped from creation through its live market lifecycle.
