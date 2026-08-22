# 🛰️ Protocol Addresses

Every Moonz launch leaves coordinates behind.

Not random coordinates.

Deterministic ones.

Give the Moonz SDK a mint and it can calculate where the important protocol accounts for that launch are supposed to live.

That is the job of the PDA helpers.

{% hint style="info" %}
**You do not need Moonz to hand you a list of account addresses.**

The important Moonz protocol addresses can be derived independently from public inputs and the Moonz Program ID.
{% endhint %}

## 🌙 The Moonz Program

The Moonz mainnet Program ID is:

```text
DBc9SEQghiJUj52YPqTKk8R4CMRgagBxi2LU1yBbeMpk
```

The SDK exports it as:

```ts
import {
  MOONZ_PROGRAM_ID
} from "@moonz-fun/sdk";

console.log(
  MOONZ_PROGRAM_ID.toBase58()
);
```

Most Moonz PDAs are derived under this program.

## 🧭 What Is a PDA?

A Program Derived Address is an address calculated from known seeds and a Solana program.

There is no normal private key behind it.

For Moonz, this means developers can independently calculate where a protocol account should be.

For example:

```text
Moonz Program
+
"launch_state"
+
token mint
=
Launch State PDA
```

The same mint and the same program always lead to the same deterministic address.

That is extremely useful for verification.

## 🗺️ Derive the Main Token Addresses

The easiest helper is:

```ts
deriveMoonzAddresses()
```

Example:

```ts
import {
  deriveMoonzAddresses
} from "@moonz-fun/sdk";

const addresses =
  deriveMoonzAddresses(
    "TOKEN_MINT"
  );

console.log(addresses);
```

It returns:

```ts
{
  launchState,
  saleVault,
  lpVault,
  treasuryWsolVault,
  treasuryUsdcVault,
  escrowSolVault
}
```

These are the main deterministic Moonz addresses associated with a launch.

## 🌙 Launch State

The Launch State PDA is derived from:

```text
"launch_state"
+
mint
```

Use:

```ts
import {
  deriveLaunchStatePda
} from "@moonz-fun/sdk";

const [
  launchState,
  bump
] = deriveLaunchStatePda(
  "TOKEN_MINT"
);
```

The Launch State is one of the most important Moonz accounts.

It contains the protocol state used to understand the launch.

That includes information such as:

```text
Creator

Lifecycle phase

Quote asset

Sale supply

Tokens sold

SOL collected

Vault addresses

Switch state

Trade timestamps
```

When the SDK checks whether a mint belongs to Moonz, this deterministic relationship is fundamental.

## 🛒 Sale Vault

The Sale Vault PDA uses:

```text
"sale_vault"
+
mint
```

Derive it with:

```ts
import {
  deriveSaleVaultPda
} from "@moonz-fun/sdk";

const [
  saleVault,
  bump
] = deriveSaleVaultPda(
  "TOKEN_MINT"
);
```

This is the expected protocol address for the token sale vault associated with that launch.

## 🌊 LP Vault

The LP Vault uses:

```text
"lp_vault"
+
mint
```

Derive it with:

```ts
import {
  deriveLpVaultPda
} from "@moonz-fun/sdk";

const [
  lpVault,
  bump
] = deriveLpVaultPda(
  "TOKEN_MINT"
);
```

Once a token is trading through the Moonz AMM, this vault becomes particularly important when reading market reserves.

## ☀️ Treasury WSOL Vault

The WSOL treasury address uses:

```text
"treasury_wsol"
+
mint
```

Use:

```ts
import {
  deriveTreasuryWsolPda
} from "@moonz-fun/sdk";

const [
  treasuryWsol,
  bump
] = deriveTreasuryWsolPda(
  "TOKEN_MINT"
);
```

This is the deterministic Moonz WSOL treasury address for the launch.

## 💵 Treasury USDC Vault

The USDC treasury address uses:

```text
"treasury_usdc"
+
mint
```

Use:

```ts
import {
  deriveTreasuryUsdcPda
} from "@moonz-fun/sdk";

const [
  treasuryUsdc,
  bump
] = deriveTreasuryUsdcPda(
  "TOKEN_MINT"
);
```

A Moonz AMM can use SOL or USDC as its active quote asset.

The SDK can derive both treasury addresses regardless of which one is currently active.

## 🔐 Escrow SOL

The SOL escrow address uses:

```text
"escrow_sol"
+
mint
```

Derive it with:

```ts
import {
  deriveEscrowSolPda
} from "@moonz-fun/sdk";

const [
  escrowSol,
  bump
] = deriveEscrowSolPda(
  "TOKEN_MINT"
);
```

This gives developers the deterministic SOL escrow address associated with the launch.

## 🚀 Derive Them Together

Usually you do not need to derive those six addresses one at a time.

Use:

```ts
const addresses =
  deriveMoonzAddresses(
    mint
  );
```

Then:

```ts
console.log({
  launchState:
    addresses.launchState
      .toBase58(),

  saleVault:
    addresses.saleVault
      .toBase58(),

  lpVault:
    addresses.lpVault
      .toBase58(),

  treasuryWsol:
    addresses.treasuryWsolVault
      .toBase58(),

  treasuryUsdc:
    addresses.treasuryUsdcVault
      .toBase58(),

  escrowSol:
    addresses.escrowSolVault
      .toBase58()
});
```

This is the fastest way to create an address map for one Moonz token.

## 🧠 Individual Helpers and Bumps

The individual PDA helpers return the normal Solana PDA result:

```ts
[
  PublicKey,
  bump
]
```

For example:

```ts
const [
  address,
  bump
] = deriveLaunchStatePda(
  mint
);
```

`deriveMoonzAddresses()` gives you the addresses themselves and leaves the individual bump values out of the combined object.

Use whichever form your integration needs.

## 🪐 Mint Authority

The Mint Authority PDA is different.

It is not derived from a token mint.

Its seed is simply:

```text
"mint_authority"
```

under the Moonz Program ID.

Use:

```ts
import {
  deriveMintAuthorityPda
} from "@moonz-fun/sdk";

const [
  mintAuthority,
  bump
] = deriveMintAuthorityPda();
```

Because no mint is supplied, this is a global Moonz program address rather than a separate PDA for every launch.

## 👤 Creator Fees

Creator Fees are scoped to a creator wallet rather than a token mint.

The seeds are:

```text
"creator_fees"
+
creator wallet
```

Use:

```ts
import {
  deriveCreatorFeesPda
} from "@moonz-fun/sdk";

const [
  creatorFees,
  bump
] = deriveCreatorFeesPda(
  "CREATOR_WALLET"
);
```

That means one creator address deterministically maps to its corresponding Creator Fees PDA.

## 🖼️ Metadata Is Different

Moonz metadata uses the Token Metadata Program.

The Token Metadata Program ID exported by the SDK is:

```text
metaqbxxUerdq28cj1RbAWkYQm3ybzjb6a8bt518x1s
```

Metadata is derived using:

```text
"metadata"
+
Token Metadata Program ID
+
mint
```

under the Token Metadata Program itself.

Use:

```ts
import {
  deriveMetadataPda
} from "@moonz-fun/sdk";

const [
  metadata,
  bump
] = deriveMetadataPda(
  "TOKEN_MINT"
);
```

This follows the standard Token Metadata PDA derivation rather than deriving the metadata account under the Moonz program.

## 🪙 Important Mint Constants

The SDK also exports the supported quote asset mint addresses.

### WSOL

```text
So11111111111111111111111111111111111111112
```

Use:

```ts
import {
  WSOL_MINT
} from "@moonz-fun/sdk";
```

### USDC

```text
EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v
```

Use:

```ts
import {
  USDC_MINT
} from "@moonz-fun/sdk";
```

These constants are useful when an integration needs to identify the actual Solana mint behind a Moonz quote asset.

## 🧱 Token Program

The SDK exports the Solana Token Program ID:

```text
TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA
```

Available as:

```ts
import {
  TOKEN_PROGRAM_ID
} from "@moonz-fun/sdk";
```

This is useful when validating the runtime ownership of token accounts.

## 🔎 Build an Address Inspector

We can now build something practical.

```ts
import {
  deriveMoonzAddresses,
  deriveMetadataPda
} from "@moonz-fun/sdk";

const mint =
  "TOKEN_MINT";

const addresses =
  deriveMoonzAddresses(
    mint
  );

const [
  metadata
] = deriveMetadataPda(
  mint
);

console.log({
  mint,

  launchState:
    addresses.launchState
      .toBase58(),

  saleVault:
    addresses.saleVault
      .toBase58(),

  lpVault:
    addresses.lpVault
      .toBase58(),

  treasuryWsolVault:
    addresses.treasuryWsolVault
      .toBase58(),

  treasuryUsdcVault:
    addresses.treasuryUsdcVault
      .toBase58(),

  escrowSolVault:
    addresses.escrowSolVault
      .toBase58(),

  metadata:
    metadata.toBase58()
});
```

Give this tool a mint and it produces the expected Moonz account map.

That could become:

* An explorer
* An integration debugger
* An audit utility
* An account inspector
* A protocol verification tool

## 🛡️ Why Derivation Matters

Imagine a Launch State account contains an address claiming to be the Sale Vault.

Your application does not have to blindly trust that stored value.

It can independently derive:

```ts
deriveSaleVaultPda(
  mint
)
```

and compare the expected address with the stored address.

The same principle applies to the other deterministic Moonz accounts.

That is exactly why deterministic protocol addresses are powerful.

The expected answer can be calculated independently.

## 🧭 The Token Address Map

For one mint, the main Moonz map looks like this:

```text
Token Mint
    |
    |__ Launch State
    |
    |__ Sale Vault
    |
    |__ LP Vault
    |
    |__ Treasury WSOL Vault
    |
    |__ Treasury USDC Vault
    |
    |__ Escrow SOL Vault
    |
    |__ Token Metadata
```

Then outside that token scoped group:

```text
Moonz Program
    |
    |__ Global Mint Authority

Creator Wallet
    |
    |__ Creator Fees
```

Now the architecture starts becoming visible.

## 🌌 You Know the Coordinates

At this point your application can do more than read values.

It can independently calculate where Moonz protocol accounts should exist.

That is useful for anyone building:

```text
Explorers

Bots

Auditors

Analytics

Indexers

Dashboards

Developer tools
```

You are no longer navigating by names alone.

You have the coordinates.

{% hint style="success" %}
**Mission complete.**

Give the SDK a mint and it can calculate the expected Moonz protocol map.

The next question is whether the accounts you find actually match that map.
{% endhint %}

## 🛡️ Next Stop

We know where the important accounts should be.

Now let us verify what is actually there.

Next stop:

**Integrity Checks**
