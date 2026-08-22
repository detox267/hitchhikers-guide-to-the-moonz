# 🔁 Production Reliability

A Geyser connection is a live pipe.

Live pipes can close.

Networks fail.

Processes restart.

Machines reboot.

Deployments happen.

A production Moonz consumer must assume that its stream will eventually disconnect.

The goal is not to prevent every interruption.

The goal is to make interruption boring.

{% hint style="info" %}
**Reliable ingestion means your system knows what it has processed, can reconnect safely and does not treat every repeated message as new information.**
{% endhint %}

## 🌊 The Public Stream Is Live

The current Moonz public Geyser is intended as a live program stream.

Historical replay is not currently part of the public integration interface.

That means you should not build an architecture that assumes you can reconnect and ask Moonz Geyser:

```text
Give me everything I missed
since slot X
```

That replay interface does not currently exist.

## 🧭 Keep Your Own Checkpoint

Your consumer should record enough information to know where it was when processing succeeded.

For example:

```text
Last processed transaction slot

Last processed transaction signature

Last processed account slot

Last successful stream activity time
```

A checkpoint is not a guarantee that every possible event can be replayed from Moonz Geyser.

It is operational evidence.

It tells your system and your operators where ingestion was when the connection disappeared.

## 🧾 Acknowledge After Storage

Do not advance your checkpoint merely because a message arrived.

Advance it after the message has been successfully processed and stored.

Conceptually:

```text
Receive message
      ↓
Validate
      ↓
Decode
      ↓
Write database
      ↓
Commit transaction
      ↓
Advance checkpoint
```

If your process crashes before storage completes, the checkpoint should not claim that the message was safely processed.

## 🔁 Duplicate Safe Processing

Reconnects and distributed systems can produce repeated observations.

Your storage layer should therefore be idempotent wherever practical.

For transaction records, useful identity fields include:

```text
signature

slot

transaction index
```

For decoded Moonz events, also preserve the event position within the transaction.

A practical event identity can be based on:

```text
transaction signature
+
event position
```

The Moonz SDK also exposes event indexing that follows this same principle.

## 🧬 Do Not Deduplicate by Event Contents

Two legitimate events can contain identical values.

For example, two trades could theoretically involve the same:

```text
mint

amount

user

timestamp value
```

That does not make them the same chain event.

Deduplicate using chain identity.

Not business field similarity.

## 🌙 Account Updates Need Different Logic

Account updates represent state rather than unique protocol actions.

For state tables, the pattern is often:

```text
Account address
      ↓
Decode new state
      ↓
Update current state record
```

You may also keep an account history table if your application needs historical state snapshots.

The current state table should generally be replaceable by newer confirmed state for the same account.

## ⚠️ Do Not Use write_version for Ordering

The Moonz bridge currently reports:

```text
write_version = 0
```

The current upstream account source does not expose validator native write version semantics.

Therefore:

```text
write_version
must not determine
Moonz account ordering
```

Use the confirmed slot information and your own ingestion rules instead.

## 📍 Preserve Slot Information

Both transaction and account updates include a slot.

Store it.

The slot gives your records useful chain context and helps with:

```text
Diagnostics

Ordering

Gap detection

State reconciliation

Operational monitoring
```

Do not throw away the slot after decoding.

## 🔗 Preserve Transaction Signatures

Transaction signatures are one of the strongest identities available to your transaction pipeline.

Store them in their canonical representation.

The example client prints signature bytes as base64 only as a connectivity demonstration.

A production indexer should convert and store them in the representation expected by its Solana tooling and database schema.

## 💓 Watch the Heartbeat

Accepted Moonz streams receive a server heartbeat approximately every fifteen seconds.

The client may also send a Subscribe ping and receive a matching pong.

A healthy consumer should track stream activity.

For example:

```text
Last data message

Last heartbeat

Last pong

Connection opened at

Connection closed at
```

This gives you an operational view of the stream rather than merely knowing that the process is running.

## ⏱️ Silence Is Information

A running process does not necessarily mean a healthy stream.

If your consumer has not observed data or heartbeat activity for longer than your chosen health window, treat the connection as unhealthy.

The exact timeout policy belongs to your infrastructure.

But the principle should be:

```text
No activity
for unexpectedly long period
      ↓
Connection considered unhealthy
      ↓
Close
      ↓
Reconnect
```

Do not allow a dead connection to remain forever simply because no explicit exception was thrown.

## 🔄 Reconnect With Backoff

When a recoverable network error occurs, reconnect gradually.

Conceptually:

```text
Disconnect
    ↓
Wait
    ↓
Reconnect
    ↓
Failure
    ↓
Wait longer
    ↓
Reconnect
```

A typical implementation uses increasing retry delays with a maximum ceiling.

Add a small random delay if many of your own workers could reconnect simultaneously.

That avoids creating a retry storm.

## 🚫 Not Every Error Should Retry Forever

Different gRPC statuses mean different things.

### PERMISSION_DENIED

```text
PERMISSION_DENIED
```

means the requested subscription exceeds the permitted Moonz scope.

Retrying the same invalid request repeatedly will not fix it.

Correct the subscription.

### RESOURCE_EXHAUSTED

```text
RESOURCE_EXHAUSTED
```

means a service capacity boundary has been reached.

The current public service limits include:

```text
5 simultaneous streams
per real client IP

100 total service clients

10 second subscription
handshake window

4 MiB maximum
gRPC message size
```

For a capacity error, reduce unnecessary connections and retry with delay.

### UNIMPLEMENTED

```text
UNIMPLEMENTED
```

means the requested RPC is not implemented by the Moonz bridge.

Retrying it will not make the RPC appear.

Use the supported Moonz surface.

## 📡 Keep Stream Count Under Control

The current limit is:

```text
5 simultaneous streams
per real client IP
```

Do not create one Geyser connection per token.

That is not the intended architecture.

A much stronger design is:

```text
One Moonz stream
      ↓
Central consumer
      ↓
Internal routing
      ↓
Token A processor

Token B processor

Analytics processor

Notification processor
```

Consume the Moonz stream once.

Distribute it inside your own system.

## 🎯 Filter Inside Your Application

The public endpoint deliberately exposes Moonz scoped data rather than arbitrary per token server filters.

So instead of:

```text
Open 200 Geyser streams
for 200 tokens
```

prefer:

```text
Open one Moonz stream
      ↓
Decode messages
      ↓
Route by mint
inside your application
```

This scales much better and respects the public service design.

## 🧱 Keep the Listener Lightweight

Your gRPC listener should not become the entire application.

Prefer:

```text
Geyser listener
      ↓
Fast validation
      ↓
Queue
      ↓
Workers
      ↓
Decode
      ↓
Database
```

This separates connection health from expensive processing.

If database writes become slow, you do not want the gRPC listener itself becoming unresponsive.

## 📦 Apply Backpressure

Every production stream eventually faces the same question:

**What happens if messages arrive faster than you can process them?**

Your application should have an explicit answer.

Possible controls include:

```text
Bounded queues

Worker limits

Database batching

Lag monitoring

Graceful overload handling
```

An unlimited in memory queue is not a reliability strategy.

It is delayed failure.

## 🧪 Validate Before Storage

Treat incoming data as external input.

Even though the server is Moonz scoped, your decoder should still validate expected structure.

For example:

```text
Expected update type?

Required fields present?

Program owner correct?

Known account discriminator?

Known event discriminator?

Integer values preserved?
```

If decoding fails, preserve enough raw context to investigate it.

Do not silently discard malformed or unknown data.

## 🛰️ Unknown Does Not Mean Invalid

Programs evolve.

Tooling evolves.

Your indexer may eventually encounter a valid structure your current decoder does not yet understand.

A useful production strategy is:

```text
Known message
      ↓
Decode normally

Unknown message
      ↓
Record safely
      ↓
Alert
      ↓
Do not invent meaning
```

Failing closed is better than producing incorrect protocol data.

## 🌙 Reconcile State After Reconnect

Because Moonz Geyser does not currently provide public historical replay, reconnecting alone cannot prove that your application observed every state transition during the outage.

For current Moonz state, you can reconcile using canonical chain reads.

Conceptually:

```text
Geyser disconnect
      ↓
Reconnect
      ↓
Stream resumes
      ↓
Read canonical Moonz state
      ↓
Compare with database
      ↓
Repair current state if needed
```

The Moonz SDK is useful here.

The Geyser gives you continuous ingestion.

The SDK gives you canonical read access for reconciliation.

## 🔭 Geyser and SDK Together

This is one of the strongest integration patterns.

```text
GEYSER

Fast live ingestion
        +
SDK

Canonical state reads
        =
Reliable Moonz application
```

Do not think of them as competing integration methods.

They complement each other.

## 🕰️ What About Missed Historical Events?

This requires care.

The current public Moonz Geyser does not provide historical replay.

So if your application requires guaranteed reconstruction of every transaction that occurred during a long outage, you need a separate historical Solana data source or your own retained history.

Do not pretend the Moonz live stream provides that functionality when it does not.

{% hint style="warning" %}
**Live state recovery and historical event recovery are different problems.**

The Moonz SDK can help reconcile current state.

Recovering every missed historical transaction requires a historical source outside the current public Moonz Geyser replay interface.
{% endhint %}

## 🗄️ Keep Raw Provenance

For important records, preserve source information such as:

```text
slot

signature

transaction index

event position

account address

subscription filter

raw canonical name
```

This lets you answer an important question later:

**Where did this database record come from?**

If your system cannot answer that, debugging becomes much harder.

## 📈 Monitor Lag

A useful production indexer should expose operational metrics.

Examples include:

```text
Connected

Last heartbeat time

Last message time

Last transaction slot

Last account slot

Queue depth

Decode failures

Database failures

Reconnect count

Processing latency
```

You do not need a giant observability platform on day one.

But you should know whether your pipeline is alive and keeping up.

## 🛑 Graceful Shutdown

When the process is intentionally stopping:

```text
Stop accepting work

Finish safe queued writes

Persist checkpoints

Cancel stream

Close gRPC client

Exit
```

The public example already demonstrates the final two operations:

```js
stream.cancel();

client.close();
```

A production worker should also make sure its own storage operations are in a safe state before exiting.

## 🧠 A Reliable Consumer Loop

Conceptually, the complete lifecycle is:

```text
START
  ↓
GetVersion
  ↓
Connect
  ↓
Subscribe
  ↓
Receive
  ↓
Validate
  ↓
Queue
  ↓
Decode
  ↓
Store
  ↓
Checkpoint
  ↓
Monitor heartbeat
  ↓
Connection fails?
  ├── NO → continue
  |
  └── YES
        ↓
     Close
        ↓
     Backoff
        ↓
     Reconnect
        ↓
     Reconcile state
        ↓
     Continue
```

That is the difference between a demo client and infrastructure.

## 🌌 The Reliability Rules

If you remember only a few things, remember these:

```text
Store chain identity

Make writes duplicate safe

Preserve slots

Do not trust write_version ordering

Monitor heartbeat health

Reconnect with backoff

Do not retry permanent errors forever

Use one stream efficiently

Reconcile current state after gaps

Do not assume historical replay exists
```

{% hint style="success" %}
**The stream is allowed to fail.**

Your architecture is not.
{% endhint %}

## 🏗️ Next Stop

We now know how to connect, subscribe, decode and survive interruptions.

There is one final step.

Put everything together.

Next stop:

**Build a Moonz Indexer**
