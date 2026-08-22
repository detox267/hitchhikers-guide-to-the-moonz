# 🧭 Don’t Panic

You have made it this far.

Time to establish communications with the Moonz.

This is your first hands on trip into the protocol.

We are going to install the Moonz SDK, connect it to Solana and teach your application how to recognise and read a Moonz token.

No trading yet.

No giant architecture diagrams.

No mysterious account bytes.

Just your application asking Solana a question and the Moonz SDK helping it understand the answer.

{% hint style="info" %}
**Your RPC is the doorway. The Moonz SDK is the translator.**

The public Moonz SDK reads protocol state from Solana.

You do not need access to a private Moonz database to begin.
{% endhint %}

## 🌐 What Is an RPC?

Before your application can read Moonz, it needs a way to communicate with Solana.

That is what an RPC gives you.

Think of it as the connection between your application and the Solana network.

Your application can use an RPC to ask questions such as:

* What data is stored in this account?
* Who owns this account?
* What accounts belong to this program?
* What happened in this transaction?
* Has something changed?

The Moonz SDK sits above that connection and understands what Moonz data means.

Instead of deriving addresses, fetching raw account bytes and decoding everything yourself, you can ask:

```ts
const token = await moonz.getToken(mint);
```

Much nicer.

## 📦 Install the Moonz SDK

The public Moonz SDK is:

```text
@moonz-fun/sdk
```

Install it with npm:

```bash
npm install @moonz-fun/sdk
```

The current SDK requires Node.js 18 or newer.

{% hint style="info" %}
**Reading and trading are separate.**

`@moonz-fun/sdk` is the read only SDK.

Later we will meet `@moonz-fun/trading-sdk` for quotes, transaction construction, buying, selling and token creation.
{% endhint %}

## 📡 Talk to Solana

Import `MoonzSDK` and give it your Solana RPC endpoint.

```ts
import { MoonzSDK } from "@moonz-fun/sdk";

const moonz = new MoonzSDK({
  rpcUrl: "YOUR_SOLANA_RPC"
});
```

That is enough to begin reading Moonz.

If you already have a Solana `Connection`, the SDK can use that instead.

```ts
const moonz = new MoonzSDK({
  connection
});
```

If your RPC provider gives you a separate WebSocket endpoint, you can provide that too.

```ts
const moonz = new MoonzSDK({
  rpcUrl: "YOUR_SOLANA_RPC",
  wsEndpoint: "YOUR_SOLANA_WSS"
});
```

The SDK uses the `confirmed` Solana commitment level by default.

For your first read, an RPC endpoint is all you need.

## 🌙 Meet the Moonz Program

The Moonz program lives on Solana mainnet at:

```text
DBc9SEQghiJUj52YPqTKk8R4CMRgagBxi2LU1yBbeMpk
```

The SDK already knows this address.

You do not need to provide it when creating the client.

## 🔎 Your First Question

Let us start with one mint and one question.

**Is this a Moonz token?**

```ts
const mint = "TOKEN_MINT";

const isMoonz = await moonz.isMoonzToken(
  mint
);

console.log(isMoonz);
```

The answer is simply:

```text
true
```

or:

```text
false
```

Simple on the surface.

There is quite a bit happening underneath.

## 🧠 What Did Moonz Actually Check?

`isMoonzToken()` does not check a private Moonz token list.

It asks the chain.

The SDK first derives the Launch State address that should belong to the supplied mint.

Then it checks that account on Solana.

For the token to be accepted as Moonz, the SDK verifies that:

* The Launch State account exists
* The Moonz program owns the account
* The account contains the correct Launch State discriminator
* The account can be decoded using the Moonz IDL
* The mint inside the decoded Launch State matches the mint you supplied

If those checks fail, the SDK does not accept the mint as a Moonz token.

{% hint style="success" %}
**Congratulations. You now understand the foundation of a Moonz token detector.**

A mint goes in.

Verified Moonz program state decides the answer.
{% endhint %}

## 🔭 Read the Token

Knowing that a token belongs to Moonz is useful.

Knowing what it is doing is much more interesting.

```ts
const token = await moonz.getToken(
  mint
);

console.log(token);
```

If the mint is not recognised as a valid Moonz token, `getToken()` returns:

```text
null
```

If it is recognised, the SDK returns a `MoonzTokenInfo` object.

That object gives your application a Moonz aware view of the token.

## 🗺️ Ask Better Questions

Printing the entire token object is useful while experimenting.

A real application will normally care about particular pieces of information.

```ts
if (token) {
  console.log(
    "Creator:",
    token.creator
  );

  console.log(
    "Phase:",
    token.phase
  );

  console.log(
    "Quote Asset:",
    token.quoteAsset
  );

  console.log(
    "Bonding Progress:",
    token.supply.bondingProgress
  );

  console.log(
    "Integrity:",
    token.integrity.all
  );
}
```

Now your application understands considerably more than a mint address.

It can begin to understand where that token sits inside Moonz.

## 📦 What Does getToken() Give You?

The public SDK brings together the important information needed to understand a Moonz token.

### Identity

* Mint
* Creator
* Moonz program ID

### Lifecycle

* Current phase
* Phase code
* Current quote asset
* Quote asset code

### Protocol State

* Launch State
* Protocol vaults
* Protocol reserves
* Token supply
* Bonding progress
* Switch state
* Last trade timestamp

### Metadata

* Name
* Symbol
* URI
* Metadata account information

### Integrity

The SDK also checks important relationships between the token and the Moonz protocol.

These include:

* Program ownership
* Launch State PDA
* Sale Vault PDA
* LP Vault PDA
* Treasury PDAs
* Escrow PDA
* Token program ownership
* Vault authorities
* Vault mints

The combined result is available at:

```ts
token.integrity.all
```

When every integrity check represented by the SDK passes, this value is `true`.

## 🧬 The Moonz Lifecycle

The current public SDK understands these Moonz phases:

```text
0  PENDING_DEV_BUY
1  BONDING
2  AMM_LIVE
3  SWITCHING
4  CANCELLED
```

The SDK can also return:

```text
UNKNOWN
```

when a value cannot be mapped to a recognised phase.

Do not memorise all of this yet.

We are going to explore the lifecycle properly later.

For now, remember one thing:

**A Moonz token has state, and your application can read it.**

## 💱 Quote Assets

The SDK currently recognises two Moonz quote assets:

```text
0  SOL
1  USDC
```

Your application normally receives the developer friendly value:

```text
SOL
```

or:

```text
USDC
```

It may also receive `UNKNOWN` if a value cannot be mapped.

This becomes much more interesting when we reach Moonz markets and PCLS.

## 🧪 Your First Tiny Build

You now know enough to create a genuinely useful little Moonz integration.

```ts
import { MoonzSDK } from "@moonz-fun/sdk";

const moonz = new MoonzSDK({
  rpcUrl: "YOUR_SOLANA_RPC"
});

const mint = "TOKEN_MINT";

const token = await moonz.getToken(mint);

if (!token) {
  console.log("Not a Moonz token");
} else {
  console.log("Moonz detected");
  console.log("Creator:", token.creator);
  console.log("Phase:", token.phase);
  console.log(
    "Quote Asset:",
    token.quoteAsset
  );
  console.log(
    "Bonding Progress:",
    token.supply.bondingProgress
  );
}
```

That is already the beginning of a Moonz scanner.

Give it a mint.

Recognise Moonz.

Read its lifecycle.

Read its market context.

Then decide where you want to take it.

## 🚀 Mission Complete

You have now:

* Connected the Moonz SDK to Solana
* Asked whether a mint belongs to Moonz
* Read a Moonz token
* Read its lifecycle phase
* Read its quote asset
* Read its bonding progress
* Seen how Moonz token detection is verified from program state

That is enough for your first trip.

{% hint style="success" %}
**You are no longer looking at Moonz from the outside.**

Your application can now read it.
{% endhint %}

## 🌌 Next Stop

Now that we can communicate with Moonz, it is time to understand the toolkit properly.

Next we will meet the Moonz SDK and explore what developers can read, discover and listen to.
