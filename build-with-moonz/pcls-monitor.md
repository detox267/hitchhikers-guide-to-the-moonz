# 🔀 PCLS Monitor

PCLS changes the quote asset of a live Moonz AMM.

A monitoring application can follow that transition from the moment the creator starts it until the token returns to a live market.

You can display:

```text
Current quote asset

Pending quote asset

Switch start time

Switch age

Escrowed switch fee

Swap execution state

Current source reserve

Current destination reserve

Creator cancellation time

Next successful switch time

PCLS events

Final active market
```

{% hint style="success" %}
Use Launch State for the current PCLS state.

Use Moonz events for the sequence of actions that led there.
{% endhint %}

## 📦 Create the SDK

```ts
import {
  MoonzSDK
} from "@moonz-fun/sdk";

const moonz =
  new MoonzSDK({
    rpcUrl:
      process.env.SOLANA_RPC_URL,

    wsEndpoint:
      process.env.SOLANA_WSS_URL
  });
```

For this monitor we will use:

```text
getToken()

getMarketData()

watchToken()
```

## 🌙 Load the Token

```ts
const mint =
  "TOKEN_MINT";

const token =
  await moonz.getToken(
    mint
  );

if (!token) {
  throw new Error(
    "Moonz token not found"
  );
}
```

The normalized PCLS state is:

```ts
token.switching
```

## 🧬 Switch State

The public SDK exposes:

```text
active

currentQuoteAsset

pendingQuoteAsset

startedAt

lastCompletedAt

feeEscrowedLamports

amountInRaw

minAmountOutRaw

swapExecuted
```

The most important current state values are:

```ts
const switching =
  token.switching;

console.log({
  active:
    switching.active,

  from:
    switching
      .currentQuoteAsset,

  to:
    switching
      .pendingQuoteAsset,

  startedAt:
    switching.startedAt,

  lastCompletedAt:
    switching.lastCompletedAt,

  feeEscrowedLamports:
    switching
      .feeEscrowedLamports,

  swapExecuted:
    switching.swapExecuted
});
```

## 🚦 Is a Switch Active?

The SDK sets:

```ts
token.switching.active
```

from the current lifecycle phase.

It is true when:

```text
phase
=
SWITCHING
```

For example:

```ts
if (
  token.switching.active
) {
  console.log(
    "PCLS in progress"
  );
}
```

## ☀️ Current Quote Asset

```ts
token.switching
  .currentQuoteAsset
```

is the active quote asset recorded in Launch State.

It can be:

```text
SOL

USDC

UNKNOWN
```

During a pending switch, this remains the original asset until completion.

## 💵 Pending Quote Asset

```ts
token.switching
  .pendingQuoteAsset
```

is the requested destination.

So an active switch may look like:

```text
Current
SOL

Pending
USDC
```

or:

```text
Current
USDC

Pending
SOL
```

## 🧭 Build the Direction Label

```ts
function switchDirection(
  token
) {
  return [
    token.switching
      .currentQuoteAsset,

    token.switching
      .pendingQuoteAsset
  ].join(" → ");
}
```

For example:

```text
SOL → USDC
```

## ⏱️ Switch Start Time

The SDK exposes:

```ts
token.switching
  .startedAt
```

from the on chain:

```text
switch_started_at
```

This is a Unix timestamp in seconds.

## 🕒 Convert a Protocol Timestamp

```ts
function dateFromUnix(
  seconds: number
) {
  if (
    seconds <= 0
  ) {
    return null;
  }

  return new Date(
    seconds * 1000
  );
}
```

Then:

```ts
const started =
  dateFromUnix(
    token.switching
      .startedAt
  );
```

## ⌛ Switch Age

```ts
function nowSeconds() {
  return Math.floor(
    Date.now() / 1000
  );
}

function switchAgeSeconds(
  startedAt: number
) {
  if (
    startedAt <= 0
  ) {
    return 0;
  }

  return Math.max(
    0,
    nowSeconds() -
      startedAt
  );
}
```

This is application display logic.

It does not replace the program's own timeout checks.

## 💰 Escrowed Switch Fee

The current program switch fee is:

```text
500,000,000 lamports

=

0.5 SOL
```

The SDK exposes the amount currently recorded in Launch State as:

```ts
token.switching
  .feeEscrowedLamports
```

Use the state value when displaying a pending switch.

## 🧮 Format the Escrowed SOL

```ts
function formatLamports(
  raw: string
) {
  const value =
    BigInt(raw);

  const base =
    1_000_000_000n;

  const whole =
    value / base;

  const fraction =
    (value % base)
      .toString()
      .padStart(
        9,
        "0"
      )
      .replace(
        /0+$/,
        ""
      );

  return fraction
    ? `${whole}.${fraction}`
    : whole.toString();
}
```

Then:

```ts
const escrowedSol =
  formatLamports(
    token.switching
      .feeEscrowedLamports
  );
```

## 🛰️ Swap Execution State

The SDK exposes:

```ts
token.switching
  .swapExecuted
```

An active switch can therefore be in two useful monitoring stages.

```text
SWITCHING
+
swapExecuted = false

Waiting for reserve conversion


SWITCHING
+
swapExecuted = true

Reserve conversion executed
Waiting for completion
```

## 🧩 Derive a Display Status

```ts
function pclsStatus(
  token
) {
  if (
    !token.switching.active
  ) {
    return "IDLE";
  }

  if (
    token.switching
      .swapExecuted
  ) {
    return "SWAP_EXECUTED";
  }

  return "WAITING_FOR_SWAP";
}
```

These labels belong to your application.

They are not additional Moonz lifecycle phases.

## ⏸️ Trading While Switching

While the token lifecycle is:

```text
SWITCHING
```

normal trading is unavailable.

You can confirm this with:

```ts
const market =
  await moonz.getMarketData(
    mint
  );
```

During the switch the market view is:

```text
market
UNAVAILABLE

tradable
false
```

## 🏦 Monitor Both Quote Treasuries

The token object exposes:

```ts
token.reserves.wsol

token.reserves.usdc
```

So your monitor can display both reserve balances while the switch progresses.

```ts
const reserves = {
  sol:
    token.reserves.wsol
      ?.amount.ui ??
    null,

  usdc:
    token.reserves.usdc
      ?.amount.ui ??
    null
};
```

## 🔀 Identify Source and Destination

```ts
function reserveForAsset(
  token,
  asset:
    "SOL" |
    "USDC" |
    "UNKNOWN"
) {
  if (
    asset === "SOL"
  ) {
    return token.reserves
      .wsol;
  }

  if (
    asset === "USDC"
  ) {
    return token.reserves
      .usdc;
  }

  return null;
}
```

Then:

```ts
const sourceReserve =
  reserveForAsset(
    token,
    token.switching
      .currentQuoteAsset
  );

const destinationReserve =
  reserveForAsset(
    token,
    token.switching
      .pendingQuoteAsset
  );
```

## 🌊 What You Should See

During:

```text
SOL → USDC
```

the old WSOL reserve is the source and the USDC treasury is the destination.

During:

```text
USDC → SOL
```

the roles reverse.

After successful conversion, the old source should be reduced to the protocol permitted dust level before completion.

## 🧹 Dust Threshold

The current program defines:

```text
SWITCH_DUST_LIMIT
=
10000 raw units
```

For SOL that represents:

```text
0.00001 SOL
```

For USDC:

```text
0.01 USDC
```

Your monitor may display the source reserve, but the program itself decides whether the completion condition is satisfied.

## ⚠️ A Current SDK Compatibility Detail

The `MoonzSwitchState` interface currently contains:

```ts
amountInRaw

minAmountOutRaw
```

However, the current public Moonz `LaunchState` layout does not persist those two switch amount fields.

The current SDK therefore falls back to:

```text
"0"
```

when those account fields are absent.

{% hint style="warning" %}
Do not use `amountInRaw` or `minAmountOutRaw` as the authoritative record of the executed PCLS conversion on the current program layout.
{% endhint %}

## 📡 Use the Swap Event for Executed Amounts

The canonical executed conversion event is:

```text
POOL_SWITCH_SWAP_EXECUTED
```

Its underlying Moonz event carries:

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

The SDK keeps these values inside:

```ts
event.data
```

## 🧬 Read Event Fields Safely

Depending on the decoded IDL field representation, application code can tolerate either naming style.

```ts
function eventField(
  data,
  camel,
  snake
) {
  return (
    data[camel] ??
    data[snake] ??
    null
  );
}
```

For the executed swap:

```ts
const amountIn =
  eventField(
    event.data,
    "amountIn",
    "amount_in"
  );

const amountOut =
  eventField(
    event.data,
    "amountOut",
    "amount_out"
  );

const sourceRemaining =
  eventField(
    event.data,
    "sourceRemaining",
    "source_remaining"
  );

const destinationBalance =
  eventField(
    event.data,
    "destinationBalance",
    "destination_balance"
  );
```

These are raw protocol units.

## 💱 Format Executed Amounts by Asset

The source amount uses the source quote asset decimals.

The destination amount uses the destination quote asset decimals.

```ts
function decimalsForQuote(
  asset:
    "SOL" |
    "USDC" |
    "UNKNOWN"
) {
  if (
    asset === "SOL"
  ) {
    return 9;
  }

  if (
    asset === "USDC"
  ) {
    return 6;
  }

  return null;
}
```

This matters because:

```text
SOL
9 decimals

USDC
6 decimals
```

## 📡 Watch PCLS Events

Subscribe with:

```ts
const stop =
  await moonz.watchToken(
    mint,
    {
      events: [
        "POOL_SWITCH"
      ],

      onEvent(event) {
        console.log(
          event.type,
          event.data
        );
      }
    }
  );
```

The normalized event types are:

```text
POOL_SWITCH_STARTED

POOL_SWITCH_SWAP_EXECUTED

POOL_SWITCH_COMPLETED

POOL_SWITCH_CANCELLED
```

## 🚀 Switch Started

When:

```ts
event.type ===
  "POOL_SWITCH_STARTED"
```

refresh the token.

The underlying event carries:

```text
mint

creator

from_asset

to_asset

switch_fee_lamports
```

The refreshed Launch State gives you the canonical:

```text
Current quote asset

Pending quote asset

Start timestamp

Escrowed fee

Switching phase
```

## 🪐 Swap Executed

When:

```ts
event.type ===
  "POOL_SWITCH_SWAP_EXECUTED"
```

store the event if you need the actual swap measurements.

Then refresh the token reserves.

The switch is not necessarily complete yet.

```text
Swap executed
≠
New market activated
```

Completion is a separate Moonz state transition.

## 🌕 Switch Completed

When:

```ts
event.type ===
  "POOL_SWITCH_COMPLETED"
```

refresh both:

```ts
moonz.getToken(
  mint
);

moonz.getMarketData(
  mint
);
```

The expected lifecycle is now:

```text
phase
AMM_LIVE

switching.active
false

market
AMM

tradable
true
```

and:

```text
quote asset
=
new asset
```

## ↩️ Switch Cancelled

When:

```ts
event.type ===
  "POOL_SWITCH_CANCELLED"
```

refresh current state.

The token should have returned to:

```text
AMM_LIVE
```

on its original quote asset.

## ⚠️ Cancellation Reason Is Not Encoded Separately

The current program emits:

```text
PoolSwitchCancelledEvent
```

for both:

```text
Creator timeout cancellation

Platform invalid route abort
```

The public SDK normalizes both as:

```text
POOL_SWITCH_CANCELLED
```

So the event alone does not tell your monitor which recovery path caused the cancellation.

{% hint style="info" %}
Treat `POOL_SWITCH_CANCELLED` as proof that the pending switch was cancelled and the original market was restored.

Do not invent a cancellation reason that the event does not contain.
{% endhint %}

## 📦 Cancellation Event Data

The cancellation event carries:

```text
mint

creator

active_asset

cancelled_at
```

The current Launch State should be reread after receiving it.

## ⏳ Creator Cancellation Time

The current creator cancellation timeout is:

```text
1800 seconds

=

30 minutes
```

For an active switch where the swap has not executed:

```ts
const creatorCancelAt =
  token.switching.startedAt > 0 &&
  !token.switching.swapExecuted
    ? token.switching
        .startedAt +
      1800
    : null;
```

This gives your interface a countdown target.

## 🛡️ Countdown Is Informational

The program still enforces the cancellation rule.

A frontend timer reaching zero does not itself cancel anything.

Your monitor should label the timer as:

```text
Creator cancellation
available after
```

rather than claiming a cancellation will automatically occur.

## 🕛 Successful Switch Cooldown

After a successful switch, the current program requires:

```text
86400 seconds

=

24 hours
```

before another successful switch cycle can begin.

The SDK exposes the last successful completion timestamp as:

```ts
token.switching
  .lastCompletedAt
```

## 🧮 Calculate the Next Cooldown Boundary

```ts
const nextSwitchAt =
  token.switching
    .lastCompletedAt > 0
    ? token.switching
        .lastCompletedAt +
      86400
    : null;
```

If:

```text
lastCompletedAt = 0
```

there is no previous successful switch timestamp to count from.

## 🧠 Cancellation Does Not Become a Successful Completion

Use:

```ts
lastCompletedAt
```

as the successful completion timestamp.

Do not replace it with the cancellation event time.

The two represent different protocol outcomes.

## 🧩 Build a PCLS Snapshot

```ts
async function loadPclsSnapshot(
  mint: string
) {
  const [
    token,
    market
  ] =
    await Promise.all([
      moonz.getToken(
        mint
      ),

      moonz.getMarketData(
        mint
      )
    ]);

  if (
    !token ||
    !market
  ) {
    return null;
  }

  const sourceReserve =
    reserveForAsset(
      token,
      token.switching
        .currentQuoteAsset
    );

  const destinationReserve =
    reserveForAsset(
      token,
      token.switching
        .pendingQuoteAsset
    );

  const cancelAvailableAt =
    token.switching.active &&
    !token.switching
      .swapExecuted &&
    token.switching
      .startedAt > 0
      ? token.switching
          .startedAt +
        1800
      : null;

  const nextSwitchAt =
    token.switching
      .lastCompletedAt > 0
      ? token.switching
          .lastCompletedAt +
        86400
      : null;

  return {
    mint:
      token.mint,

    phase:
      token.phase,

    active:
      token.switching.active,

    status:
      pclsStatus(
        token
      ),

    currentQuoteAsset:
      token.switching
        .currentQuoteAsset,

    pendingQuoteAsset:
      token.switching
        .pendingQuoteAsset,

    startedAt:
      token.switching
        .startedAt,

    lastCompletedAt:
      token.switching
        .lastCompletedAt,

    feeEscrowedLamports:
      token.switching
        .feeEscrowedLamports,

    swapExecuted:
      token.switching
        .swapExecuted,

    sourceReserve:
      sourceReserve
        ?.amount ??
      null,

    destinationReserve:
      destinationReserve
        ?.amount ??
      null,

    cancelAvailableAt,

    nextSwitchAt,

    market:
      market.market,

    tradable:
      market.tradable,

    integrity:
      token.integrity.all &&
      market.integrityAll
  };
}
```

## 🖥️ Waiting for Swap

A monitor could display:

```text
PCLS

SOL → USDC

Status
Waiting for reserve conversion

Trading
Unavailable

Switch fee escrowed
0.5 SOL

Swap executed
No

Creator cancellation
Available in 18m 42s
```

The countdown should be calculated from live state.

## 🪐 Swap Executed View

After the reserve conversion event:

```text
PCLS

SOL → USDC

Status
Reserve conversion executed

Trading
Unavailable

Swap executed
Yes

Waiting for
Moonz completion
```

If you stored the executed event, you can additionally show:

```text
Amount in

Amount out

Source remaining

Destination balance
```

from the event data.

## 🌕 Successful Completion View

After completion:

```text
PCLS COMPLETE

Active quote
USDC

Market
AMM

Trading
Live

Next switch
Available after cooldown
```

The actual quote asset and timestamp must come from the refreshed state.

## ↩️ Recovery View

After:

```text
POOL_SWITCH_CANCELLED
```

you can display:

```text
PCLS CANCELLED

Original market restored

Active quote
SOL

Trading
Live
```

Do not label the reason as creator cancellation or invalid route unless you have another authoritative source for that distinction.

## 📡 Keep the Last Executed Event

A useful monitor can keep the latest:

```text
POOL_SWITCH_SWAP_EXECUTED
```

event separately from the current Launch State.

For example:

```ts
let lastSwapEvent =
  null;

const stop =
  await moonz.watchToken(
    mint,
    {
      events: [
        "POOL_SWITCH"
      ],

      onEvent(event) {
        if (
          event.type ===
          "POOL_SWITCH_SWAP_EXECUTED"
        ) {
          lastSwapEvent =
            event;
        }
      }
    }
  );
```

This gives you:

```text
Current state
from Launch State

+

Executed swap measurements
from event
```

without pretending the account stores data that it currently does not.

## 🔄 Refresh on Every PCLS Event

A strong pattern is:

```text
PCLS event
    ↓
Store event if useful
    ↓
Read current token
    ↓
Read current market
    ↓
Replace monitor snapshot
```

This works for:

```text
Started

Swap executed

Completed

Cancelled
```

## 🔐 Deduplicate Stored Events

If you persist event history, use:

```text
signature
+
eventIndex
```

as the event identity.

The monitor snapshot itself should still come from current state.

## 🛰️ Reconnect Recovery

If your WebSocket disconnects:

```text
Reconnect
    ↓
getToken()
    ↓
getMarketData()
    ↓
Rebuild current PCLS snapshot
    ↓
Resume watchToken()
```

If you need historical executed swap measurements that occurred while disconnected, recover the relevant transaction events through your historical ingestion strategy.

## 🚀 Complete Monitor Example

```ts
import {
  MoonzSDK
} from "@moonz-fun/sdk";

const moonz =
  new MoonzSDK({
    rpcUrl:
      process.env.SOLANA_RPC_URL,

    wsEndpoint:
      process.env.SOLANA_WSS_URL
  });

const mint =
  "TOKEN_MINT";

let snapshot =
  await loadPclsSnapshot(
    mint
  );

let lastSwapEvent =
  null;

console.log(
  snapshot
);

const stop =
  await moonz.watchToken(
    mint,
    {
      events: [
        "POOL_SWITCH"
      ],

      async onEvent(event) {
        if (
          event.type ===
          "POOL_SWITCH_SWAP_EXECUTED"
        ) {
          lastSwapEvent =
            event;
        }

        snapshot =
          await loadPclsSnapshot(
            mint
          );

        console.log({
          event:
            event.type,

          snapshot,

          lastSwapEvent
        });
      },

      onError(error) {
        console.error(
          "PCLS monitor error",
          error
        );
      }
    }
  );

// Later:
// await stop();
```

## 🌌 What You Have Built

Your PCLS monitor now understands:

```text
Whether a switch is active

Which asset is being replaced

Which asset is pending

When the switch began

How much switch fee is escrowed

Whether the reserve swap executed

What both quote treasuries currently hold

When creator cancellation becomes available

When the successful switch cooldown ends

What the executed swap event measured

When the new market becomes active

When the original market is restored
```

{% hint style="success" %}
The monitor separates current state from event history.

That distinction matters during PCLS because the account tells you where the token is now while the events tell you how the switch progressed.
{% endhint %}

## 🚀 Next Build

We can now read Moonz, watch it, follow bonding, inspect the AMM and monitor PCLS.

The final practical build adds execution.

Next build:

**Trading Interface**
