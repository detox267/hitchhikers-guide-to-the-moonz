# 🛡️ Integrity Checks

You know where the Moonz accounts should be.

Now comes the more important question.

**Are the accounts you found actually the accounts Moonz expects?**

The SDK does not simply read addresses and hope for the best.

When you load a complete Moonz token with `getToken()`, it performs a collection of deterministic account checks and gives you the results through:

```ts
token.integrity
```

This is the Moonz integrity model.

{% hint style="info" %}
**Integrity is about relationships.**

The SDK checks whether the important accounts around a Moonz launch line up with the deterministic protocol structure that should exist for that mint.
{% endhint %}

## 🔎 Read the Integrity Result

Start with a normal token read.

```ts
const token =
  await moonz.getToken(
    mint
  );

if (!token) {
  console.log(
    "Not a Moonz token"
  );

  process.exit(0);
}

console.log(
  token.integrity
);
```

The result contains:

```text
programOwner

launchStatePda

saleVaultPda
lpVaultPda

treasuryWsolPda
treasuryUsdcPda

escrowSolPda

tokenProgramOwners
vaultAuthorities
vaultMints

all
```

Each field answers a different question about the protocol structure.

## 🌙 Program Ownership

The first field is:

```ts
token.integrity.programOwner
```

Before the full integrity calculation is reached, the SDK has already loaded the Launch State through `getLaunchState()`.

That read requires the account to be owned by the Moonz Program.

So by the time the full integrity object is created, the Launch State program ownership check has already passed.

The integrity object records that result as:

```text
programOwner = true
```

This is an important distinction.

The SDK is not blindly setting trust to true.

The Launch State read has already passed the ownership gate before `getToken()` continues.

## 🛰️ Launch State PDA

The SDK independently derives the expected Launch State address for the mint.

Then it compares that with the Launch State being used.

Conceptually:

```text
Expected Launch State
derived from mint

        =?

Actual Launch State
```

The result is:

```ts
token.integrity
  .launchStatePda
```

If this is true, the Launch State address matches the deterministic Moonz PDA expected for that mint.

## 🛒 Sale Vault PDA

The Launch State stores a Sale Vault address.

The SDK does not simply accept it.

It independently derives:

```text
"sale_vault"
+
mint
```

and compares the expected address against the stored address.

Read:

```ts
token.integrity
  .saleVaultPda
```

## 🌊 LP Vault PDA

The same process applies to the LP Vault.

The SDK derives the expected address from:

```text
"lp_vault"
+
mint
```

and compares it with the LP Vault stored in Launch State.

Read:

```ts
token.integrity
  .lpVaultPda
```

## ☀️ Treasury WSOL PDA

The expected WSOL treasury address is derived from:

```text
"treasury_wsol"
+
mint
```

The stored and derived addresses are compared.

Read:

```ts
token.integrity
  .treasuryWsolPda
```

## 💵 Treasury USDC PDA

The expected USDC treasury address is derived from:

```text
"treasury_usdc"
+
mint
```

Read the result with:

```ts
token.integrity
  .treasuryUsdcPda
```

## 🔐 Escrow SOL PDA

The expected SOL escrow address is derived from:

```text
"escrow_sol"
+
mint
```

Then compared with the address stored in Launch State.

Read:

```ts
token.integrity
  .escrowSolPda
```

## 🧭 Stored Versus Derived

These PDA checks are based on a useful principle.

The Launch State tells you:

```text
Here are my vault addresses
```

The SDK independently calculates:

```text
Here are the vault addresses
that should exist
```

Then it compares them.

For example:

```ts
token.vaults.stored.saleVault
```

is checked against:

```ts
token.vaults.derived.saleVault
```

That is much stronger than simply exposing whatever address happened to be stored on chain.

## 🪙 Token Program Ownership

Correct addresses are only part of the story.

The SDK also inspects these token accounts:

```text
Sale token vault

LP token vault

WSOL treasury

USDC treasury
```

For:

```ts
token.integrity
  .tokenProgramOwners
```

to be true, every one of those accounts must exist and its runtime owner must be the Solana Token Program.

The expected Token Program is:

```text
TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA
```

Conceptually:

```text
Sale Vault
        ↓
Token Program

LP Vault
        ↓
Token Program

WSOL Treasury
        ↓
Token Program

USDC Treasury
        ↓
Token Program
```

All four must pass.

## 🏛️ Vault Authorities

Next the SDK asks:

**Who actually controls these token accounts?**

For:

```ts
token.integrity
  .vaultAuthorities
```

to be true, every inspected token account must have the Moonz Launch State as its token authority.

That includes:

```text
Sale token vault

LP token vault

WSOL treasury

USDC treasury
```

The expected relationship is:

```text
Token Vault
    |
    | authority
    ↓
Launch State
```

for each of those token accounts.

{% hint style="info" %}
A correct PDA address is not enough by itself.

The SDK also checks that the token accounts are actually controlled by the expected Launch State authority.
{% endhint %}

## 🧬 Vault Mints

There is another question.

**Does each vault contain the kind of token it is supposed to contain?**

The SDK checks exactly that.

For:

```ts
token.integrity
  .vaultMints
```

to be true:

```text
Sale token vault
must use the launch mint

LP token vault
must use the launch mint

WSOL treasury
must use the WSOL mint

USDC treasury
must use the USDC mint
```

The expected WSOL mint is:

```text
So11111111111111111111111111111111111111112
```

The expected USDC mint is:

```text
EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v
```

This prevents a vault relationship from looking correct while actually holding the wrong mint.

## 🧩 All Four Token Accounts Matter

There is an important implementation detail.

The full integrity calculation expects all four inspected token accounts to exist:

```text
Sale token vault
LP token vault
WSOL treasury
USDC treasury
```

The SDK uses all four when checking:

```text
tokenProgramOwners
vaultAuthorities
vaultMints
```

If one of those accounts is missing, these combined checks cannot all pass.

That matters even when only one quote asset is currently active.

The integrity model is checking the complete Moonz account structure, not just the reserve currently being used for pricing.

## ✅ The Combined Result

The final field is:

```ts
token.integrity.all
```

The SDK calculates this by requiring every individual integrity field to be true.

Conceptually:

```text
programOwner
        AND

launchStatePda
        AND

saleVaultPda
        AND

lpVaultPda
        AND

treasuryWsolPda
        AND

treasuryUsdcPda
        AND

escrowSolPda
        AND

tokenProgramOwners
        AND

vaultAuthorities
        AND

vaultMints
        ↓
       all
```

If every check passes:

```text
all = true
```

If even one fails:

```text
all = false
```

## 🔍 Find the Failed Check

Do not stop at `all`.

If it is false, inspect the individual fields.

```ts
const integrity =
  token.integrity;

console.log({
  programOwner:
    integrity.programOwner,

  launchStatePda:
    integrity.launchStatePda,

  saleVaultPda:
    integrity.saleVaultPda,

  lpVaultPda:
    integrity.lpVaultPda,

  treasuryWsolPda:
    integrity.treasuryWsolPda,

  treasuryUsdcPda:
    integrity.treasuryUsdcPda,

  escrowSolPda:
    integrity.escrowSolPda,

  tokenProgramOwners:
    integrity.tokenProgramOwners,

  vaultAuthorities:
    integrity.vaultAuthorities,

  vaultMints:
    integrity.vaultMints,

  all:
    integrity.all
});
```

Now you can see exactly which relationship failed.

That is much more useful than receiving a single generic validation error.

## 🚨 Build an Integrity Alert

You can turn the result into a simple safety check.

```ts
const token =
  await moonz.getToken(
    mint
  );

if (!token) {
  throw new Error(
    "Not a Moonz token"
  );
}

if (!token.integrity.all) {
  console.error(
    "Moonz integrity check failed",
    token.integrity
  );
}
```

This could become part of:

* An explorer
* A trading interface
* An analytics pipeline
* An auditor
* An integration test
* A monitoring service

## 📊 Integrity and Market Data

Remember the Market Data section?

The integrity result matters there too.

Before the SDK exposes canonical Moonz market data, it checks:

```ts
token.integrity.all
```

If that is false, the market helper returns:

```text
market      UNAVAILABLE
priceSource UNAVAILABLE
tradable    false
```

That means integrity is not merely decorative information.

It directly protects the canonical market representation exposed by the SDK.

## 🔭 Discovery Integrity Is Smaller

There is another distinction worth understanding.

The discovery method:

```ts
moonz.getTokens()
```

returns a lighter `MoonzTokenSummary`.

Its discovery integrity result contains PDA checks such as:

```text
launchStatePda
saleVaultPda
lpVaultPda
treasuryWsolPda
treasuryUsdcPda
escrowSolPda

allPdas
```

That is useful while scanning many tokens.

But it is not the same as the complete integrity result returned by:

```ts
moonz.getToken()
```

The full token read additionally checks:

```text
Token Program ownership

Vault authorities

Vault mints
```

Think of it like this:

```text
getTokens()

Fast discovery integrity
Focused on deterministic PDAs

        ↓

getToken()

Complete token integrity
PDAs
+
runtime owners
+
authorities
+
mints
```

Use the level that matches what your application is doing.

## 🖼️ What About Metadata?

Metadata validation is separate.

`MoonzIntegrity` does not include the metadata checks.

When metadata exists, the SDK exposes its own validation fields through:

```ts
token.metadata
```

These include:

```text
validOwner
validMint
matchesLaunchState
matchesDerivedPda
```

For example:

```ts
if (token.metadata) {
  console.log({
    validOwner:
      token.metadata.validOwner,

    validMint:
      token.metadata.validMint,

    matchesLaunchState:
      token.metadata
        .matchesLaunchState,

    matchesDerivedPda:
      token.metadata
        .matchesDerivedPda
  });
}
```

Do not assume:

```ts
token.integrity.all
```

means:

```text
every possible metadata property
has also been validated
```

They are intentionally separate surfaces.

## 🧠 Build a Moonz Inspector

Now combine the pieces.

```ts
import {
  MoonzSDK
} from "@moonz-fun/sdk";

const moonz =
  new MoonzSDK({
    rpcUrl:
      "YOUR_SOLANA_RPC"
  });

const token =
  await moonz.getToken(
    "TOKEN_MINT"
  );

if (!token) {
  console.log(
    "Not a Moonz token"
  );

  process.exit(0);
}

console.log({
  mint:
    token.mint,

  phase:
    token.phase,

  quoteAsset:
    token.quoteAsset,

  integrity:
    token.integrity,

  metadataIntegrity:
    token.metadata
      ? {
          validOwner:
            token.metadata
              .validOwner,

          validMint:
            token.metadata
              .validMint,

          matchesLaunchState:
            token.metadata
              .matchesLaunchState,

          matchesDerivedPda:
            token.metadata
              .matchesDerivedPda
        }
      : null
});
```

Give it a mint.

Get back the Moonz state.

Then inspect whether the important relationships line up.

## 🛡️ Trust, But Calculate

That is the real idea behind this section.

Do not trust an address simply because it appeared inside an account.

Derive what should exist.

Check who owns it.

Check who controls it.

Check what mint belongs there.

Then compare.

```text
Stored state
    +
Derived addresses
    +
Runtime accounts
    +
Expected authorities
    +
Expected mints
    ↓
Moonz integrity
```

{% hint style="success" %}
**Mission complete.**

You can now find Moonz.

Read Moonz.

Price Moonz.

Listen to Moonz.

Decode Moonz history.

Derive Moonz addresses.

And independently check whether the important protocol relationships line up.
{% endhint %}

## 🌙 You Have the SDK

That completes our tour of the core public Moonz SDK.

You now know how to use it as more than a collection of functions.

You know what the functions are actually telling you.

From here we can move deeper into the Moonz protocol itself.

The next part of the guide will explain the journey behind the state you have been reading.

**The Token Journey**
